

# Wallet Registrering

```plantuml
@startuml

participant "Wallet App" as walletApp
participant "App BFF" as walletBff
participant "App BFF\nSession store" as sessionStore
participant "Wallet Access" as walletAccess
participant "Wallet Access\nPersistent store" as walletAccessPersistentStore
participant "HSM" as hsm
participant "Wallet Provider" as walletProvider


walletApp -> walletBff: Authenticate with eId  (???)
note right: eId Auth Request with OIDC or something \n Keep authorization header/cookie during the registration process
walletBff --> walletApp: Bearer token, session cookie or similar


walletApp -> walletApp: generate device key pairs\n - device key \n - PIN hardening key

group User registration 
walletApp -> walletBff: Bearer token 
note right: Någon sorts registrering och "mina sidor \n Inga detaljer här - separat flöde mellan walletApp och utfärdarteamet?" 
walletBff -> walletApp: 
end

group Session establishment
note right of walletApp: Förslag: Mellan Wallet App ochApp BFF skall det finnas integritet och konfidentialitet. \n Föreslagen lösning är HPKE med ephemeral key. App teamet ansvarar för detta.  
walletApp -> walletApp: get device key handle
walletApp -> walletBff: HPKE (device.pub, bff.pub)
walletBff -> walletBff: compute sessionKey
walletBff -> sessionStore: store sessionKey
walletBff -> walletApp: sessionId 
walletApp -> walletApp: compute sessionKey
end
group Register wallet device
walletApp -> walletBff: Register wallet instance, contents: \n - Bearer token\n - device public key \n - opaque KE1 with user selected PIN\n \n JWE(sessionKey, JWS(device, contents))

walletBff -> walletAccess: Initialize wallet with device public key \n JWS(device, contents)
walletAccess -> hsm: Create wallet key pair in HSM
hsm --> walletAccess: wallet public key
walletAccess -> walletAccess: compute \n - deviceId = sha256(device public key) \n - wallet key identifier = sha256(wallet public key)
walletAccess -> walletAccessPersistentStore: Store wallet: \n - deviceId \n - device public key \n - wallet key identifier \n - wallet public key \n - opaque verifier  
walletAccess -> walletAccess: compute pakeSessionKey (optional)

walletAccess --> walletProvider: Register wallet instance with wallet public key and deviceId 
note left: På något sätt behöver deviceId, device public key, wallet public key och wallet key identifier \n göras tillgängligt för wallet provider och kanske BFF \n mikrotjänst eller via kafka topic?

walletAccess -> walletBff: The User's wallet \n - device public key \n - wallet public key? \n - wallet key identifier? \n - JWS(hsmServer, KE2)

walletBff --> walletApp: JWE(sessionKey, JWS(hsmServer, KE2))

walletApp -> walletBff: authenticate hsm context Opaque KE3 Finalize\nJWE(sessionKey, JWS(device, KE3))
walletBff -> walletAccess: authenticate hsm context Opaque KE3 Finalize\nJWS(device, KE3)
walletAccess --> walletBff: JWS(hsmServer, pakeSessionId)
walletBff --> walletApp: JWE(sessionKey, JWS(hsmServer, pakeSessionId))
walletApp -> walletApp: compute pakeSessionKey (optional)
note right: Om session key behövs för efterföljande flöde så kan man tänka sig att den räknas ut här. \n T.ex. för att hämta WUA eller göra proof of possesion i OpenID4VCI PID flödet

end

group Get Wallet User Attestation (WUA)
walletApp -> walletBff: JWE(sessionKey, deviceId)
walletBff -> walletProvider: get WUA deviceId
walletProvider --> walletBff: WUA
walletProvider -> walletAccess: get public key for deviceId
note right: Reda ut om vi har i en eller flera databaser \n ev ha deviceId och publika nyklar i en kafka topic \n som alla intresserade konsumerar
walletAccess --> walletProvider: wallet public key
walletProvider --> walletBff: WUA 
walletBff --> walletApp: JWE(sessionKey, WUA)
end

group Continue with PID credential issuance
walletApp --> walletBff: Continue
note right: OpenID4VCI flow something. 
end

```

# Frågor att diskutera med andra team

## Metadata
Vi har metadata som deviceId, wallet public key och wallet key identifier som vi behöver lagra och göra tillgänglig för t.ex. wallet provider.
Vi har även metadata som opaque verifier som är hemligare och som bara hsm server i wallet access behöver.

Förslagsvis: Accessmekanismen äger det metadatat och har det i sin databas. Vidare publiceras allt som ska delas med andra team/tjänster på en kafka topic som alla intresserade konsumerar from beginning. Utöver det finns det en mikrotjänst som servar datat via REST GET.
