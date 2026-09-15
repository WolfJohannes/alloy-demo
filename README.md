# Alloy → Envoy → Keycloak Demo

A local Podman Compose proof-of-concept demonstrating how **Grafana Alloy** can send OpenTelemetry telemetry through **Envoy** using OAuth2 Client Credentials authentication with **Keycloak**.

The demo runs entirely locally.

## Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                    Podman Compose Network                   │
│                                                             │
│  ┌─────────────────┐                                        │
│  │  alloy-onprem   │                                        │
│  │                 │                                        │
│  │  OAuth2 Client  │                                        │
│  │  OTLP/gRPC      │                                        │
│  └────────┬────────┘                                        │
│           │                                                  │
│           │ HTTPS :443                                      │
│           │                                                  │
│           ▼                                                  │
│  ┌─────────────────┐                                        │
│  │     Envoy       │                                        │
│  │                 │                                        │
│  │  TLS            │                                        │
│  │  /token         │                                        │
│  │  JWT validation │                                        │
│  │  OTLP routing   │                                        │
│  └───────┬─────┬───┘                                        │
│          │     │                                             │
│          │     │ OTLP/gRPC                                  │
│          │     ▼                                             │
│          │  ┌──────────────────┐                            │
│          │  │ alloy-receiver   │                            │
│          │  │                  │                            │
│          │  │ OTLP :4317       │                            │
│          │  │ debug exporter   │                            │
│          │  └──────────────────┘                            │
│          │                                                   │
│          │ HTTP :8080                                       │
│          ▼                                                   │
│  ┌─────────────────┐                                        │
│  │    Keycloak     │                                        │
│  │                 │                                        │
│  │  realm: demo    │                                        │
│  │  demo-client    │                                        │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Components

| Component        | Purpose                                                   | Internal port |
| ---------------- | --------------------------------------------------------- | ------------: |
| `alloy-onprem`   | Generates/scrapes and exports telemetry                   |             — |
| `envoy`          | TLS gateway, token proxy, JWT validation and OTLP routing |         `443` |
| `keycloak`       | OAuth2/OIDC identity provider                             |        `8080` |
| `alloy-receiver` | Receives OTLP/gRPC telemetry                              |        `4317` |

### Host ports

```text
localhost:4443 → Envoy :443
localhost:9901 → Envoy admin interface
localhost:8080 → Keycloak :8080
```

The `4443` port is only the host-side mapping to Envoy's internal port `443`.

Containers communicate using their Compose service names:

```text
alloy-onprem → envoy:443
envoy        → keycloak:8080
envoy        → alloy-receiver:4317
```

---

# Prerequisites

The demo requires:

* Podman
* Podman Compose
* `curl`
* `jq`

Check the installed versions:

```bash
podman --version
podman compose version
curl --version
jq --version
```

---

# Repository structure

```text
.
├── docker-compose.yml
├── alloy-onprem/
│   └── config.alloy
├── alloy-receiver/
│   └── config.alloy
├── envoy/
│   ├── envoy.yaml
│   └── certs/
│       ├── server.crt
│       └── server.key
└── keycloak/
    └── realm.json
```

---

# Keycloak

The Keycloak configuration creates a realm:

```text
demo
```

and a confidential client:

```text
Client ID:     demo-client
Client secret: demo-secret
```

The client uses the OAuth2 Client Credentials flow.

Keycloak listens internally on:

```text
keycloak:8080
```

The token endpoint is:

```text
http://keycloak:8080/realms/demo/protocol/openid-connect/token
```

The JWKS endpoint is:

```text
http://keycloak:8080/realms/demo/protocol/openid-connect/certs
```

---

# Envoy

Envoy listens internally on:

```text
0.0.0.0:443
```

The host maps port `4443` to Envoy's port `443`:

```yaml
ports:
  - "4443:443"
```

Therefore:

```text
Host:
https://localhost:4443

Container:
https://envoy:443
```

## Token endpoint

Alloy does **not** connect directly to Keycloak.

Instead, Alloy sends its OAuth2 token request to:

```text
https://envoy:443/token
```

Envoy forwards `/token` to Keycloak:

```text
/token
   ↓
/realms/demo/protocol/openid-connect/token
```

The actual Keycloak connection is:

```text
http://keycloak:8080
```

## JWT validation

Envoy uses the JWT authentication filter to validate the access token.

The configuration contains a Keycloak provider:

```yaml
providers:
  keycloak:
    issuer: "..."
    audiences:
      - "account"

    remote_jwks:
      http_uri:
        uri: "http://keycloak:8080/realms/demo/protocol/openid-connect/certs"
        cluster: keycloak
        timeout: 5s
```

The `issuer` must exactly match the `iss` claim contained in the JWT.

For example, if the token contains:

```json
{
  "iss": "http://keycloak:443/realms/demo"
}
```

then Envoy must use:

```yaml
issuer: "http://keycloak:443/realms/demo"
```

The issuer value is the JWT identity. It is separate from the actual network endpoint used to contact Keycloak.

## OTLP/gRPC

Envoy forwards OTLP/gRPC requests to:

```text
alloy-receiver:4317
```

The upstream Envoy cluster uses HTTP/2:

```yaml
http2_protocol_options: {}
```

---

# Alloy on-prem

The on-prem Alloy instance obtains its OAuth2 token through Envoy:

```alloy
otelcol.auth.oauth2 "keycloak" {
  client_id     = "demo-client"
  client_secret = sys.env("ALLOY_CLIENT_SECRET")

  token_url = "https://envoy:443/token"

  tls {
    insecure_skip_verify = true
  }
}
```

The OTLP exporter sends telemetry to Envoy:

```alloy
otelcol.exporter.otlp "aws" {
  client {
    endpoint = "envoy:443"

    auth = otelcol.auth.oauth2.keycloak.handler

    tls {
      insecure_skip_verify = true
    }
  }
}
```

The important point is that `envoy:443` is used because Alloy and Envoy are on the same Compose network.

Do **not** use:

```text
localhost:4443
```

inside the Alloy container.

Inside the Alloy container, `localhost` refers to the Alloy container itself.

---

# Alloy receiver

The internal Alloy receiver listens on:

```text
0.0.0.0:4317
```

It accepts OTLP/gRPC:

```alloy
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }

  output {
    metrics = [otelcol.exporter.debug.default.input]
    logs    = [otelcol.exporter.debug.default.input]
    traces  = [otelcol.exporter.debug.default.input]
  }
}

otelcol.exporter.debug "default" {}
```

The debug exporter allows us to see the telemetry that successfully passed through Envoy.

---

# TLS

The local Envoy endpoint uses a TLS certificate:

```text
envoy/certs/server.crt
envoy/certs/server.key
```

Because this is a local POC, the Alloy configuration currently uses:

```alloy
tls {
  insecure_skip_verify = true
}
```

This avoids certificate trust and hostname issues while testing the architecture.

---

# Start the demo

Start all services:

```bash
podman compose up -d
```

Check the running containers:

```bash
podman compose ps
```

Or:

```bash
podman ps
```

---

# Test Keycloak

Check the Keycloak OpenID configuration:

```bash
curl http://localhost:8080/realms/demo/.well-known/openid-configuration
```

The response should contain the OIDC endpoints for the `demo` realm.

---

# Get an access token

The token request can be sent through Envoy:

```bash
curl -k \
  -X POST \
  https://localhost:4443/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'grant_type=client_credentials' \
  --data 'client_id=demo-client' \
  --data 'client_secret=demo-secret'
```

The request path is:

```text
Host
  ↓
localhost:4443
  ↓
Envoy:443
  ↓
Keycloak:8080
```

A successful response contains an access token:

```json
{
  "access_token": "...",
  "token_type": "Bearer",
  "expires_in": 1200
}
```

---

# Inspect the JWT

Save the token:

```bash
TOKEN=$(curl -sk \
  -X POST \
  https://localhost:4443/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'grant_type=client_credentials' \
  --data 'client_id=demo-client' \
  --data 'client_secret=demo-secret' \
  | jq -r '.access_token')
```

Decode the payload:

```bash
echo "$TOKEN" \
  | cut -d. -f2 \
  | base64 -d 2>/dev/null \
  | jq
```

Important claims include:

```text
iss
aud
azp
exp
iat
```

For example:

```json
{
  "iss": "http://keycloak:443/realms/demo",
  "aud": "account",
  "azp": "demo-client"
}
```

The `iss` value must match Envoy's configured `issuer`.

---

# Envoy admin interface

Envoy exposes its admin interface on:

```text
http://localhost:9901
```

Check the active configuration:

```bash
curl -s http://localhost:9901/config_dump | jq
```

Inspect the JWT configuration:

```bash
curl -s http://localhost:9901/config_dump \
  | jq '.. | objects | select(has("providers")) | .providers'
```

This is useful when checking whether Envoy has actually loaded the configuration currently present in `envoy.yaml`.

---

# Logs

## Envoy

```bash
podman logs envoy
```

Follow the logs:

```bash
podman logs -f envoy
```

JWT-related messages:

```bash
podman logs envoy 2>&1 | grep -Ei 'jwt|auth'
```

## On-prem Alloy

```bash
podman logs alloy-onprem
```

Follow the logs:

```bash
podman logs -f alloy-onprem
```

## Alloy receiver

```bash
podman logs alloy-receiver
```

The receiver's debug exporter should show telemetry once the complete flow is working.

---

# Troubleshooting

## `Connection refused`

Example:

```text
upstream connect error
connection refused
```

Check that all containers are running:

```bash
podman compose ps
```

Check Keycloak:

```bash
podman logs keycloak
```

Envoy should connect to:

```text
keycloak:8080
```

not the host port.

---

## `Jwt issuer is not configured`

This means Envoy received the JWT but could not match the token's issuer to a configured JWT provider.

First decode the token:

```bash
echo "$TOKEN" \
  | cut -d. -f2 \
  | base64 -d 2>/dev/null \
  | jq '.iss'
```

Then inspect Envoy's active configuration:

```bash
curl -s http://localhost:9901/config_dump \
  | jq '.. | objects | select(has("providers")) | .providers'
```

Compare:

```text
JWT:
iss = ...

Envoy:
issuer = ...
```

They must be identical.

---

## `missing selected ALPN property`

If Alloy reports:

```text
missing selected ALPN property
```

make sure Envoy's downstream TLS configuration contains:

```yaml
alpn_protocols:
  - h2
  - http/1.1
```

The `h2` protocol is required for OTLP/gRPC.

---

## TLS certificate errors

For this local POC, Alloy uses:

```alloy
tls {
  insecure_skip_verify = true
}
```

This is intentional because the demo uses a local/self-signed certificate.

---

# Useful commands

Start:

```bash
podman compose up -d
```

Stop:

```bash
podman compose down
```

Recreate Envoy:

```bash
podman compose up -d --force-recreate envoy
```

Restart on-prem Alloy:

```bash
podman compose restart alloy-onprem
```

Show services:

```bash
podman compose ps
```

Follow Envoy logs:

```bash
podman logs -f envoy
```

Follow on-prem Alloy logs:

```bash
podman logs -f alloy-onprem
```

Follow receiver logs:

```bash
podman logs -f alloy-receiver
```

Inspect Envoy's active configuration:

```bash
curl -s http://localhost:9901/config_dump | jq
```

---

# Demo flow

The complete authentication and telemetry flow is:

```text
1. Alloy requests an OAuth2 token
             │
             ▼
       Envoy :443
             │
             ▼
       Keycloak :8080
             │
             ▼
        JWT access token
             │
             ▼
2. Alloy sends OTLP/gRPC + JWT
             │
             ▼
       Envoy :443
             │
             ├── Validate JWT
             │
             └── Forward OTLP/gRPC
                       │
                       ▼
                Alloy receiver :4317
                       │
                       ▼
                  Debug exporter
```

This repository is intended as a small, reproducible local environment for testing the interaction between **Grafana Alloy, Envoy, Keycloak, OAuth2, JWT validation and OTLP/gRPC**.
