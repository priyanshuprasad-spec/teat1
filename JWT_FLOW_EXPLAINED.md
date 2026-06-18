# JWT Auth Flow — Complete Explainer (From Scratch)

> Assume you know AWS. Assume you know nothing about Kubernetes or Helm. This doc explains everything.

---

## Part 1 — The Big Picture (AWS Analogy First)

You already know this AWS pattern:

```
User → Route53 → ALB → Target Group → EC2 instance
```

In Kubernetes (K8s), it works the same way conceptually:

```
User → DNS → AWS NLB → kgateway → Kubernetes Service → Pod (your app)
```

The difference is that instead of EC2 instances, you have **Pods** (containers). Instead of an ALB with listener rules, you have **kgateway** with routing rules called **HTTPRoutes**. And instead of AWS Cognito JWT validation, you have a **GatewayExtension** that does JWT validation directly inside the gateway.

---

## Part 2 — What is Kubernetes? (One Paragraph)

Kubernetes is just a system that runs containers (Docker images) at scale. Think of it like ECS but with a lot more control. Instead of task definitions you have **Pods**. Instead of ECS services you have **Deployments**. Instead of ALB target groups you have **Services** (ClusterIP). The config is all written in YAML files called **manifests**.

---

## Part 3 — What is Helm?

Helm is like AWS CloudFormation but for Kubernetes. You write template YAML files with variables (like `{{ .Values.host }}`), and then a `values.yaml` file fills in those variables. This repo (`ure-shunya-helm-management`) is a collection of Helm **charts** — one chart per microservice.

When you deploy, Helm renders the templates with the values and applies them to the cluster.

```
templates/jwt-gateway-extension.yaml  ← template with {{ .Values.jwt.issuer }}
            +
dev-values.yaml                        ← fills in: issuer: "https://auth.dev.shunya.app"
            =
Final YAML applied to K8s cluster
```

---

## Part 4 — What is kgateway?

kgateway is the **API Gateway** for this cluster. Think of it as the front door — every request from the internet hits kgateway first before it reaches any microservice.

In AWS terms: **kgateway = ALB + WAF rules + Cognito JWT validation**, all in one.

kgateway runs as a **Pod** inside the cluster in a namespace called `kgateway-system`. AWS provisions an **NLB** (Network Load Balancer) in front of it automatically — so traffic path is:

```
Internet → AWS NLB → kgateway Pod → your microservice Pod
```

---

## Part 5 — What is Zitadel?

Zitadel is the **Identity Provider (IdP)** for this system. Think of it like AWS Cognito but self-hosted.

- It lives at `https://auth.dev.shunya.app`
- It issues **JWT tokens** when a user logs in
- It exposes a public key endpoint (JWKS) at `https://auth.dev.shunya.app/oauth/v2/keys`

That JWKS endpoint is basically: "Here are my public keys. Use them to verify that any JWT you receive was actually signed by me."

---

## Part 6 — What is a JWT Token?

A JWT (JSON Web Token) is a string that looks like this:

```
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIiwiaXNzIjoiaHR0cHM6Ly9hdXRoLmRldi5zaHVueWEuYXBwIiwiYXVkIjoiMzczNTExOTcyMjExMTk3NDUwIiwiZXhwIjoxNzUwMDAwMDAwfQ.SIGNATURE
```

It has three parts separated by dots:

| Part | What it is |
|------|-----------|
| `eyJhbGci...` | Header — says which algorithm was used to sign |
| `eyJzdWIi...` | Payload (claims) — the actual user data (base64 encoded JSON) |
| `SIGNATURE` | Cryptographic proof the token wasn't tampered with |

If you base64-decode the payload, you get something like:

```json
{
  "sub": "user-id-123",
  "iss": "https://auth.dev.shunya.app",
  "aud": "373511972211197450",
  "exp": 1750000000,
  "urn:zitadel:iam:org:project:roles": { "admin": {} }
}
```

- `sub` = subject = the user's unique ID
- `iss` = issuer = who issued this token (must be Zitadel)
- `aud` = audience = which app/project this token is for
- `exp` = expiry timestamp
- `urn:zitadel:iam:org:project:roles` = the user's roles

---

## Part 7 — The Full Request Flow (Step by Step)

Let's trace a request from a browser hitting `shunya-dot-tap-backend.dev.shunya.app`:

### Step 1 — User logs in

The frontend app redirects the user to Zitadel (`https://auth.dev.shunya.app`). User enters credentials. Zitadel returns a **JWT token** (a Bearer token) to the frontend.

```
Frontend → POST /login → Zitadel
Zitadel → 200 OK + JWT Token
```

### Step 2 — Frontend makes an API call

The frontend sends an API request and puts the JWT in the `Authorization` header:

```http
GET /api/game/start HTTP/1.1
Host: shunya-dot-tap-backend.dev.shunya.app
Authorization: Bearer eyJhbGci....<the JWT token>
```

### Step 3 — Request hits AWS NLB

DNS for `shunya-dot-tap-backend.dev.shunya.app` points to an AWS **NLB** (Network Load Balancer). The NLB forwards the TCP traffic to the **kgateway Pod** inside the cluster on port 443.

### Step 4 — kgateway receives the request

kgateway is the first thing in the cluster that sees this request. It looks at the hostname (`shunya-dot-tap-backend.dev.shunya.app`) and finds the matching **HTTPRoute** (think of this like ALB listener rules).

**Configured in:** `shunya-dot-tap/templates/kgateway.yaml`  
**Gateway definition:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shunya-dot-tap-helm-auth-gateway
  namespace: kgateway-system
spec:
  gatewayClassName: kgateway
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    hostname: shunya-dot-tap-backend.dev.shunya.app
    tls:
      mode: Terminate        # kgateway handles TLS, backend sees plain HTTP
      certificateRefs:
      - name: dot-tap-helm-auth-tls
```

TLS is **terminated here** — kgateway decrypts the HTTPS connection and talks to your backend service over plain HTTP internally.

### Step 5 — TrafficPolicy intercepts the request

Before routing to the backend, the **TrafficPolicy** kicks in. This is where JWT validation and CORS happen.

**Configured in:** `shunya-dot-tap/templates/jwt-authpolicy.yaml`

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: dot-tap-policy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: dot-tap-auth-route        # applies to this specific route
  jwtAuth:
    extensionRef:
      name: zitadel-jwt             # points to the GatewayExtension below
  cors:
    allowOrigins: ["*"]
    allowMethods: [GET, POST, PUT, DELETE, PATCH, OPTIONS]
    allowCredentials: true
```

**Order of processing:**
1. CORS check first (OPTIONS preflight passes through without JWT)
2. JWT validation second

### Step 6 — JWT Validation (The Core Part)

The **GatewayExtension** named `zitadel-jwt` does the actual JWT work.

**Configured in:** `shunya-dot-tap/templates/jwt-gateway-extension.yaml`

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: GatewayExtension
metadata:
  name: zitadel-jwt
spec:
  jwt:
    validationMode: Strict
    providers:
      - name: zitadel
        issuer: "https://auth.dev.shunya.app"
        audiences:
          - "373511972211197450"
          - "373510453084554762"
          - "373510383006123530"
          - "377054813776512561"
        tokenSource:
          header:
            header: Authorization
            prefix: "Bearer "
        jwks:
          remote:
            url: "https://auth.dev.shunya.app/oauth/v2/keys"
        claimsToHeaders:
          - name: "sub"
            header: x-user-id
          - name: "urn:zitadel:iam:org:project:roles"
            header: x-user-roles
        forwardToken: true
```

Here is exactly what kgateway does at this step, line by line:

#### 6a — Extract the token

```
tokenSource.header.header: Authorization
tokenSource.header.prefix: "Bearer "
```

kgateway strips `Bearer ` from the Authorization header and extracts the raw JWT string.

#### 6b — Fetch public keys from Zitadel (JWKS)

```
jwks.remote.url: "https://auth.dev.shunya.app/oauth/v2/keys"
```

kgateway calls this URL to get Zitadel's **public keys** (JWKS = JSON Web Key Set). This is the set of public keys Zitadel uses to sign tokens. kgateway caches these keys for **1 hour** (configured as `zitadel_jwks_cache_ttl_ms: "3600000"` in dev-values.yaml) so it doesn't call Zitadel on every single request.

The JWKS response looks like:
```json
{
  "keys": [
    {
      "kty": "RSA",
      "kid": "key-id-abc",
      "n": "...big public key modulus...",
      "e": "AQAB"
    }
  ]
}
```

#### 6c — Verify the JWT signature

Using the public key from JWKS, kgateway verifies the JWT's cryptographic signature. If the token was tampered with (payload changed after signing), this verification **fails** and kgateway returns `401 Unauthorized`.

#### 6d — Validate claims

```
issuer: "https://auth.dev.shunya.app"       → must match token's "iss" field
audiences: ["373511972211197450", ...]       → token's "aud" must be one of these
validationMode: Strict                       → exp, iat, nbf must all be valid
```

- If `iss` doesn't match → 401
- If `aud` not in the list → 401
- If token is expired (`exp` in the past) → 401

#### 6e — Extract claims and inject headers

```
claimsToHeaders:
  - name: "sub"                              → header: x-user-id
  - name: "urn:zitadel:iam:org:project:roles" → header: x-user-roles
```

kgateway reads the claims from the JWT payload and injects them as **HTTP headers** into the forwarded request. Your backend service never needs to parse the JWT itself — it just reads headers:

```
x-user-id: user-id-123
x-user-roles: {"admin":{}}
Authorization: Bearer eyJhbGci...  ← original token also forwarded (forwardToken: true)
```

### Step 7 — HTTPRoute routes to backend

If JWT is valid, the HTTPRoute sends the request to the **Kubernetes Service** (ClusterIP), which load-balances to the correct Pod.

**Configured in:** `shunya-dot-tap/templates/httproute.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: dot-tap-auth-route
spec:
  hostnames:
  - shunya-dot-tap-backend.dev.shunya.app
  parentRefs:
  - name: shunya-dot-tap-helm-auth-gateway
    namespace: kgateway-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: dot-tap-service    # Kubernetes Service name
      port: 10000
```

The Kubernetes Service (`dot-tap-service`) is a ClusterIP — it's internal only and forwards to the actual Pods running your game backend.

### Step 8 — Backend responds

Your backend Pod processes the request. It can trust the `x-user-id` and `x-user-roles` headers completely because the gateway already validated the JWT — it would have rejected the request before it reached here if anything was wrong.

Response flows back:

```
Backend Pod → dot-tap-service → kgateway → NLB → User
```

---

## Part 8 — The Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                  INTERNET                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    1. GET /api/game  │  Authorization: Bearer <JWT>
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          AWS NLB (auto-provisioned)                         │
│                    shunya-dot-tap-backend.dev.shunya.app                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                         2. forwards TCP on port 443
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    kgateway Pod (namespace: kgateway-system)                │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  3. TLS Termination — decrypts HTTPS using Let's Encrypt cert        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  4. HTTPRoute match — hostname matches, selects dot-tap-auth-route   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                      │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  5. TrafficPolicy (dot-tap-policy)                                   │  │
│  │     a. CORS check                                                    │  │
│  │     b. JWT validation via GatewayExtension (zitadel-jwt)            │  │
│  │        ├── Extract Bearer token from Authorization header            │  │
│  │        ├── Fetch/use cached JWKS from auth.dev.shunya.app            │  │
│  │        ├── Verify signature with public key                          │  │
│  │        ├── Validate: issuer, audience, expiry                        │  │
│  │        └── Extract claims → inject x-user-id, x-user-roles headers  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                    │ VALID              │ INVALID                           │
│                    ▼                   ▼                                    │
│             route to backend     return 401 Unauthorized                   │
└─────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Kubernetes Service: dot-tap-service (ClusterIP, port 10000)                │
└─────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Backend Pod                                                                │
│  Receives headers:                                                          │
│    x-user-id: user-id-123                                                   │
│    x-user-roles: {"admin":{}}                                               │
│    Authorization: Bearer <original JWT>                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 9 — Public vs Protected Routes

Not every route requires JWT. In the HTTPRoutes file there are two types:

| Route name | Path | JWT required? | Why |
|------------|------|--------------|-----|
| `dot-tap-auth-route` | `/` (everything) | Yes | TrafficPolicy targets this route |
| `dot-tap-api-route` | `/api` | No | Only CORS policy, no jwtAuth |

The `/api` route has a separate TrafficPolicy that only applies CORS — no JWT check. So public endpoints live under `/api`, and protected endpoints are on the auth route.

---

## Part 10 — TLS Certificates (How HTTPS Works Here)

TLS certificates are managed by **cert-manager**, which automatically gets free certificates from **Let's Encrypt**.

Each service has its own certificate for its hostname (e.g., `shunya-dot-tap-backend.dev.shunya.app`). The certificate secret is stored in the `kgateway-system` namespace so kgateway can read it.

**Configured in:** `shunya-dot-tap/templates/kgateway.yaml` (the TLS section) and cert-manager CRDs.

There's also a **wildcard certificate** for `*.dev.shunya.app` for the shared gateway.

---

## Part 11 — Zitadel: The Auth Server

Zitadel runs as a Pod in the cluster in its own namespace (`zitadel`). It's a full OAuth2/OIDC server.

Key config values from `dev-values.yaml`:

| Key | Value | What it means |
|-----|-------|--------------|
| `ZITADEL_HOST` | `https://auth.dev.shunya.app` | Where Zitadel lives |
| `ZITADEL_PROJECT_ID` | `361865814808200593` | The main project in Zitadel |
| `zitadel_web_client_id` | `377055111773423153` | The client ID for the web frontend |
| `zitadel_jwks_cache_ttl_ms` | `3600000` | Cache public keys for 1 hour |
| `audiences` | 4 IDs | Any token for these projects is accepted |

The **audiences** list in the GatewayExtension allows tokens from 4 different Zitadel projects/clients — this handles the case where the frontend uses a different client ID than the mobile app, but both should be allowed through.

---

## Part 12 — What Happens When JWT Validation Fails?

| Scenario | What kgateway returns |
|----------|----------------------|
| No Authorization header | `401 Unauthorized` |
| Token is expired | `401 Unauthorized` |
| Token signature is invalid (tampered) | `401 Unauthorized` |
| Wrong issuer (not from Zitadel) | `401 Unauthorized` |
| Audience doesn't match any in the list | `401 Unauthorized` |
| Valid token | Request forwarded to backend with injected headers |

---

## Part 13 — How Claims Get to the Backend

The backend service doesn't do any JWT parsing. It simply reads HTTP headers:

```python
# Example in backend code
user_id = request.headers.get("x-user-id")   # comes from JWT "sub" claim
user_roles = request.headers.get("x-user-roles")  # comes from JWT roles claim
```

kgateway does all the heavy lifting — signature verification, claim extraction, header injection — before the request ever reaches your code.

---

## Part 14 — Key Files Reference

| What | File |
|------|------|
| Gateway definition (NLB + listeners) | `shunya-dot-tap/templates/kgateway.yaml` |
| JWT validation config (the actual logic) | `shunya-dot-tap/templates/jwt-gateway-extension.yaml` |
| JWT enforcement (which routes are protected) | `shunya-dot-tap/templates/jwt-authpolicy.yaml` |
| Route rules (which path goes where) | `shunya-dot-tap/templates/httproute.yaml` |
| All configurable values (dev) | `shunya-dot-tap/dev-values.yaml` |
| Shared gateway for the whole cluster | `gateway-resources/gateway.yaml` |
| Wildcard TLS cert for `*.dev.shunya.app` | `gateway-resources/dev-wildcard-certificate.yaml` |

---

## Part 15 — Quick Q&A for the Meeting

**Q: Where does JWT validation happen?**  
A: Inside kgateway, before the request reaches any backend service. The `GatewayExtension` named `zitadel-jwt` does it.

**Q: How does kgateway know the JWT is legitimate?**  
A: It fetches public keys from Zitadel's JWKS endpoint (`/oauth/v2/keys`) and uses them to verify the cryptographic signature on the JWT. Only Zitadel has the private key to sign tokens, so only valid Zitadel tokens pass.

**Q: Does the backend service need to validate the JWT?**  
A: No. The gateway validates it. The backend just reads the `x-user-id` and `x-user-roles` headers that kgateway injects.

**Q: What is JWKS caching?**  
A: kgateway doesn't call Zitadel on every request. It caches the public keys for 1 hour. On a cache miss (or first request), it calls `https://auth.dev.shunya.app/oauth/v2/keys`.

**Q: What is an audience?**  
A: It's a claim in the JWT that says "this token was issued for application X." We configure 4 audience IDs so tokens from 4 different Zitadel clients (web app, mobile, etc.) are all accepted.

**Q: How does HTTPS work?**  
A: kgateway terminates TLS (decrypts HTTPS) using a Let's Encrypt certificate managed by cert-manager. Communication between kgateway and the backend pod is plain HTTP inside the cluster.

**Q: What is Helm doing here?**  
A: Helm is a templating tool. The YAML configs in `templates/` have variables like `{{ .Values.jwt.issuer }}`. When deployed, Helm replaces these with values from `dev-values.yaml` and applies the final YAML to the cluster.

**Q: Why is there an NLB and not an ALB?**  
A: kgateway uses NLB (Layer 4, TCP-level) instead of ALB (Layer 7, HTTP-level). kgateway itself handles all the Layer 7 routing and JWT logic. The NLB is just the entry point that gets traffic to kgateway's pod. There is also an older ALB-based ingress pattern in this repo (`shunya-chicmic-gateway`) but the main services use NLB + kgateway.
