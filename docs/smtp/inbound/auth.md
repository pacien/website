---
sidebar_position: 7
---

# AUTH stage

The `AUTH` command is used to authenticate a user who wants to send an email message through an SMTP server. The `AUTH` command initiates the authentication process, after which the client must provide their credentials for authentication using SASL.
SASL (Simple Authentication and Security Layer) is a framework for authentication and data security in network protocols. It provides a mechanism for clients to authenticate themselves to a server using a variety of authentication methods, such as username and password, public key cryptography, or other mechanisms. SASL can be used with various network protocols, including SMTP, to provide secure authentication and data exchange between client and server.

## Settings

SASL authentication is configured using the following attributes which are available under the `session.auth` key:

- `mechanisms`: A list of [SASL authentication mechanisms](https://www.iana.org/assignments/sasl-mechanisms/sasl-mechanisms.xhtml) offered to clients, or an empty list to disable authentication. Stalwart SMTP currently supports `PLAIN`, `LOGIN`, and `OAUTHBEARER` mechanisms.
- `directory`: Specifies the [directory](/docs/directory/overview) to use for authentication.
- `require`: A boolean attribute that specifies whether authentication is necessary to send email messages.
- `errors.total`: The maximum number of authentication errors allowed before the session is disconnected.
- `errors.wait`: The time interval to wait after an authentication failure.

Example using an LDAP directory for authentication:

```txt
[session.auth]
mechanisms = [ { if = "listener", ne = "smtp", then = ["plain", "login"]},
               { else = [] } ]
directory = [ { if = "listener", ne = "smtp", then = "ldap" }, 
           { else = false } ]
require = [ { if = "listener", ne = "smtp", then = true},
            { else = false } ]

[session.auth.errors]
total = 3
wait = "5s"
```
