# Signera

Signera något med användarens HSM nyckel på serverns från wallet app.

```plantuml
@startuml

participant "Wallet App" as walletApp
participant "PID Issuer" as pidIssuer
participant "App BFF" as walletBff
participant "Session store" as sessionStore
participant "Wallet Access\nLoA high " as walletAccess

group Device authentication and session establishment
note right of walletApp: Förslag: Mellan Wallet App ochApp BFF skall det finnas integritet och konfidentialitet. \n Föreslagen lösning är HPKE med ephemeral key. App teamet ansvarar för detta.  
walletApp -> walletApp: generate ephemeral EC key
walletApp -> walletBff: HPKE (ephemeral, bff.pub)
walletBff -> walletBff: compute appSessionKey,
walletBff -> sessionStore: store appSessionKey,
walletBff -> walletApp: sessionId, 
walletApp -> walletApp: compute appSessionKey,
end
group Sign payload with private key in HSM
walletApp -> walletBff: authenticate hsm context Opaque KE1 Evaluate\nJWE(appSessionKey, JWS(deviceKey, KE1))
walletBff -> walletAccess: authenticate hsm context Opaque KE1 Evaluate\nJWS(deviceKey, KE1)
note right of walletBff: Förslag: Wallet Access LoA high behöver autenticitet av deviceKey. Förslag för detta är JWS. \n Om konfidentialitet dessutom behövs föreslår vi JWE+JWS 
walletAccess -> walletAccess: compute pakeSessionKey
walletAccess --> walletBff: JWS(hsmServer, KE2)
walletBff --> walletApp: JWE(appSessionKey, JWS(hsmServer, KE2))

walletApp -> walletBff: authenticate hsm context Opaque KE3 Finalize\nJWE(appSessionKey, JWS(deviceKey, KE3))
walletBff -> walletAccess: authenticate hsm context Opaque KE3 Finalize\nJWS(deviceKey, KE3)
walletAccess --> walletBff: JWS(hsmServer, pakeSessionId)
walletBff --> walletApp: JWE(appSessionKey, JWS(hsmServer, pakeSessionId))
walletApp -> walletApp: compute pakeSessionKey,

walletApp -> walletBff: hsm context hsm_ecdsa sign payload\nJWE(appSessionKey, JWE(pakeSessionKey, payload))
walletBff -> walletAccess: hsm context hsm_ecdsa sign jwt payload\nJWE(pakeSessionKey, payload)
walletAccess --> walletBff: JWE(pakeSessionKey, signature)
walletBff->walletApp: JWE(appSessionKey, JWE(pakeSessionKey, signature))

walletAccess --> walletAccess: remove sign session when ttl expires
note left: Det som händer först av ttl expiry eller \n explicit stängning av session
walletApp -> walletBff: close sign session\nJWE(appSessionKey, JWE(pakeSessionKey, "close"))
walletBff -> walletAccess: JWE(pakeSessionKey, "close")
walletAccess --> walletBff: JWE(pakeSessionKey, "OK")
walletBff --> walletApp: JWE(appSessionKey, JWE(pakeSessionKey, "OK"))
end
@enduml
```

