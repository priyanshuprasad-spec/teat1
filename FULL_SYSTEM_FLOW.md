# Shunya Platform — Complete System Flow (Dev Environment)

> Written for someone who knows AWS but has never touched Kubernetes or Helm.  
> Dev environment only. Use this to answer questions in your meeting.

---

## Part 1 — What Even Is This System?

Shunya is a **gaming platform**. Think of it like a collection of mini-games (Sudoku, Maze, Dot Tap, etc.), each running as its own microservice. There is also an **Admin Panel** where admins configure those games.

The whole thing runs on **AWS EKS** (Elastic Kubernetes Service) — which is basically AWS's managed version of Kubernetes. Instead of EC2 instances you manage yourself, EKS runs containers (Pods) for you.

```
Users play games   →   Game backends (one per game)
Admins manage      →   Admin Backend + Admin Frontend
Both log in via    →   Zitadel (the auth server)
Everything enters  →   kgateway (the API Gateway inside EKS)
```

---

## Part 2 — The Whole Architecture at a Glance

```
INTERNET
   │
   ├── shunya-admin.dev.shunya.app          → Admin Frontend (Next.js UI)
   ├── shunya-admin-backend.dev.shunya.app  → Admin Backend API
   ├── api.dev.shunya.app                   → Auth Service (login, users)
   ├── auth.dev.shunya.app                  → Zitadel (identity provider)
   │
   ├── shunya-dot-tap-backend.dev.shunya.app      → Dot Tap game
   ├── shunya-sudoku-backend.dev.shunya.app        → Sudoku game
   ├── shunya-maze-backend... (not confirmed)      → Maze game
   ├── infinity-loop-backend.dev.shunya.app        → Infinity Loop
   ├── sequence-recall.dev.shunya.app              → Sequence Recall
   ├── arrows.dev.shunya.app                       → Arrow game
   ├── block-fill-backend.dev.shunya.app           → Block Fill
   └── ... (and more games)
   
   ALL of the above go through:
   AWS NLB → kgateway (inside EKS) → backend Pod
```

---

## Part 3 — Quick Kubernetes/Helm Glossary (Mapped to AWS)

| Kubernetes term | AWS equivalent | What it does |
|----------------|---------------|-------------|
| **Pod** | EC2 / ECS Task | One running container (your app) |
| **Deployment** | ECS Service | Keeps N copies of your Pod running |
| **Service (ClusterIP)** | Internal ALB Target Group | Load balances to Pods inside the cluster, not reachable from internet |
| **Namespace** | VPC Subnet (roughly) | Logical isolation inside the cluster |
| **kgateway** | ALB + WAF + Cognito JWT | The internet-facing API gateway |
| **NLB** | Network Load Balancer | Same as AWS NLB — kgateway gets one per service |
| **HTTPRoute** | ALB Listener Rule | "If hostname is X and path is Y, send to service Z" |
| **TrafficPolicy** | WAF Rule Group | "Apply JWT check / CORS / rate limit to this route" |
| **GatewayExtension** | Cognito JWT Authorizer | Validates JWT tokens against Zitadel |
| **Helm chart** | CloudFormation stack | Template + values → deployed YAML |
| **dev-values.yaml** | CloudFormation parameters | The variables that fill in the template |
| **Restate** | AWS Step Functions | Distributed workflow engine for game state |
| **Zitadel** | AWS Cognito | Self-hosted identity provider (issues JWTs) |
| **cert-manager** | ACM | Auto-provisions Let's Encrypt TLS certificates |
| **ArgoCD** | CodePipeline/CodeDeploy | Watches this Git repo and deploys changes automatically |

---

## Part 4 — How a Request Gets Into the Cluster

Every service gets its own **AWS NLB** (provisioned automatically by kgateway). Traffic path:

```
Browser/App
    │
    ▼
AWS NLB  (created automatically per service, points to kgateway)
    │
    ▼
kgateway Pod  (runs in namespace: kgateway-system)
    │   ← does TLS termination, JWT validation, CORS, routing
    ▼
Kubernetes Service (ClusterIP)  ← internal only, not internet-facing
    │
    ▼
Your App Pod  ← receives plain HTTP with injected headers
```

kgateway handles:
- HTTPS termination (using Let's Encrypt cert from cert-manager)
- JWT validation via Zitadel
- CORS headers
- Rate limiting (configured per service, currently disabled on dev)
- HTTP → HTTPS redirect (301)
- Blocking `/webhook` paths (returns 403)

---

## Part 5 — The Login Flow (How a User Gets a JWT)

```
1. User opens the app (web or mobile)

2. App redirects to Zitadel login page:
   https://auth.dev.shunya.app

3. User enters credentials (email/password, Google SSO, etc.)

4. Zitadel verifies credentials and issues a JWT token
   The JWT contains:
   {
     "sub":  "user-unique-id-123",       ← user's ID
     "iss":  "https://auth.dev.shunya.app",  ← who issued it
     "aud":  "373511972211197450",        ← which app
     "exp":  1750000000,                  ← expiry timestamp
     "urn:zitadel:iam:org:project:roles": { "admin": {} }  ← roles
   }

5. App stores the JWT (in memory or localStorage)

6. App includes it in every API call:
   Authorization: Bearer eyJhbGci....
```

The JWT is signed with Zitadel's **private key**. Anyone can verify it using Zitadel's **public key** (available at `https://auth.dev.shunya.app/oauth/v2/keys`). This is the JWKS (JSON Web Key Set) endpoint.

---

## Part 6 — JWT Validation at kgateway (The Core Part)

Every backend service (games, admin) has this setup in its Helm chart:

**GatewayExtension** (`templates/jwt-gateway-extension.yaml`):
```yaml
kind: GatewayExtension
spec:
  jwt:
    validationMode: Strict
    providers:
      - name: zitadel
        issuer: "https://auth.dev.shunya.app"       # must match token's iss
        audiences:
          - "373511972211197450"                     # token's aud must be one of these
          - "373510453084554762"
          - "373510383006123530"
          - "377054813776512561"
        tokenSource:
          header:
            header: Authorization
            prefix: "Bearer "                        # strips "Bearer " and reads the token
        jwks:
          remote:
            url: "https://auth.dev.shunya.app/oauth/v2/keys"   # fetch public keys here
        claimsToHeaders:
          - name: "sub"
            header: x-user-id                        # extracts user ID → injects as header
          - name: "urn:zitadel:iam:org:project:roles"
            header: x-user-roles                     # extracts roles → injects as header
        forwardToken: true/false                     # see Part 9 for the difference
```

**TrafficPolicy** (`templates/jwt-authpolicy.yaml`):
```yaml
kind: TrafficPolicy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: xxx-auth-route        # applies JWT to this specific route
  jwtAuth:
    extensionRef:
      name: zitadel-jwt         # uses the GatewayExtension above
  cors:
    allowOrigins: ["*"]
    allowMethods: [GET, POST, PUT, DELETE, PATCH, OPTIONS]
```

**What happens step by step:**

```
Request arrives with: Authorization: Bearer eyJhbGci...
                                              │
                              ┌───────────────▼────────────────┐
                              │  Step 1: Extract token          │
                              │  Strip "Bearer ", get raw JWT   │
                              └───────────────┬────────────────┘
                                              │
                              ┌───────────────▼────────────────┐
                              │  Step 2: Get Zitadel public key │
                              │  Fetch from JWKS URL            │
                              │  (cached 1 hour = 3600000ms)    │
                              └───────────────┬────────────────┘
                                              │
                              ┌───────────────▼────────────────┐
                              │  Step 3: Verify signature       │
                              │  Token tampered? → 401          │
                              └───────────────┬────────────────┘
                                              │
                              ┌───────────────▼────────────────┐
                              │  Step 4: Validate claims        │
                              │  iss ≠ auth.dev.shunya.app? 401 │
                              │  aud not in list? → 401         │
                              │  Token expired? → 401           │
                              └───────────────┬────────────────┘
                                              │
                              ┌───────────────▼────────────────┐
                              │  Step 5: Extract claims         │
                              │  sub → x-user-id header         │
                              │  roles → x-user-roles header    │
                              └───────────────┬────────────────┘
                                              │
                              ┌───────────────▼────────────────┐
                              │  Step 6: Forward to backend     │
                              │  Backend gets:                  │
                              │    x-user-id: user-123          │
                              │    x-user-roles: {"admin":{}}   │
                              └────────────────────────────────┘
```

**The backend never parses a JWT itself** — it just reads headers. All JWT work is done at the gateway.

---

## Part 7 — Route Types: What's Protected vs Public

Each service has multiple HTTPRoutes. The TrafficPolicy attaches JWT validation to specific routes only. Other routes are public.

### Example: Dot Tap (`shunya-dot-tap-backend.dev.shunya.app`)

| Route name | Paths | JWT required? | Reason |
|-----------|-------|--------------|--------|
| `http-redirect` | all HTTP traffic | No | Just redirects to HTTPS |
| `webhook-block-route` | `/webhook` | No (blocked) | Returns 403 immediately |
| `api-route` | `/api` | **No** | Public API endpoints |
| `auth-route` | `/` (everything else) | **Yes** | JWT enforced via TrafficPolicy |

```
Request to /api/...  →  skips JWT  →  backend
Request to /game/... →  JWT check  →  valid? → backend  /  invalid? → 401
Request to /webhook  →  403 Forbidden (blocked)
```

### Example: Admin (`shunya-admin-backend.dev.shunya.app`)

| Route | Paths | JWT required? |
|-------|-------|--------------|
| `auth-public-route` | `/admin/auth/token`, `/admin/auth`, `/admin/games/stage-content` | **No** |
| `admin-route` | `/api` | **No** |
| `me-route` | `/admin/auth/me`, `/admin/auth/logout` | **Yes** |
| `auth-route` | `/` (everything else) | **Yes** |

The admin has more nuanced public routes because the login endpoint itself (`/admin/auth/token`) must be reachable without a JWT (you can't log in if you need to already be logged in!).

---

## Part 8 — All Services and Their Roles

### Zitadel (`auth.dev.shunya.app`)
- The **identity provider** — issues JWTs
- Users log in here
- Exposes JWKS endpoint for JWT verification
- Runs as its own Pod in the `zitadel` namespace

### shunya-application (`api.dev.shunya.app`)
The **central platform service**. Actually 3–4 microservices deployed together:

| Sub-service | Port | What it does |
|------------|------|-------------|
| **authentication-service** | 3000 | Login/user management, talks to Zitadel |
| **google-pubsub-service** | 3003 | Handles Google Play purchase events |
| **ios-notification-service** | 3004 | Handles Apple App Store purchase events |
| **mcq-admin-service** | 3005 | Admin for MCQ (multiple choice) questions |

All share the same gateway (`api.dev.shunya.app`) and a shared wildcard TLS cert. Subscriptions routes are JWT protected. Auth routes are mostly public (login doesn't require being logged in).

### shunya-admin-frontend (`shunya-admin.dev.shunya.app`)
- A **Next.js web app** — serves HTML/CSS/JS to the browser
- **NO JWT validation at kgateway** — it's a frontend, not an API
- Authentication happens inside the Next.js app (it calls the admin backend)
- Runs in its own Pod, gets its own NLB + kgateway gateway
- Has no `jwt-gateway-extension.yaml` at all

### shunya-admin (`shunya-admin-backend.dev.shunya.app`)
- The **admin backend API** — used by the admin frontend
- JWT validation: **Yes**, `forwardToken: false` (explained in Part 9)
- Knows about all game services — has SQS queue URLs for every game:
  - `aws_sqs_cheat_flag_queue_url` 
  - `aws_sqs_arrows_queue_url`
  - `aws_sqs_sudoku_queue_url`
  - `aws_sqs_sequence_recall_queue_url` … etc
- Games call the admin at `admin_microservice_base_url: "https://shunya-admin-backend.dev.shunya.app"` to fetch game configs (level data, leaderboard rules, etc.)

### Game Services (all backends)

All on the pattern `shunya-<game>-backend.dev.shunya.app` (some vary):

| Service folder | Hostname | Restate instance | Team |
|---------------|----------|-----------------|------|
| `shunya-dot-tap` | `shunya-dot-tap-backend.dev.shunya.app` | `restate-chicmic` | **Chicmic** |
| `shunya-sudoku` | `shunya-sudoku-backend.dev.shunya.app` | `restate-dev` | Shunya |
| `shunya-maze` | `shunya-maze-backend...` | `restate-dev` | Shunya |
| `shunya-infinity-loop` | `infinity-loop-backend.dev.shunya.app` | `restate-dev` | Shunya |
| `shunya-arrow` | `arrows.dev.shunya.app` | `restate-dev` | Shunya |
| `shunya-block-fill` | `block-fill-backend.dev.shunya.app` | `restate-dev` | Shunya |
| `shunya-logic-reflector` | *(check values)* | `restate-dev` | Shunya |
| `shunya-memory-card-matching` | *(check values)* | `restate-dev` | Shunya |
| `shunya-sequence-recall` | `sequence-recall.dev.shunya.app` | `restate-dev` | Shunya |
| `shunya-sliding-puzzle` | *(check values)* | `restate-dev` | Shunya |
| `shunya-spot-the-difference` | *(check values)* | `restate-dev` | Shunya |
| `shunya-water-color-sorting` | *(check values)* | `restate-dev` | Shunya |

**ALL game services use Zitadel JWT at the gateway level.** All have `kgateway.jwt.enabled: true` in their dev-values.yaml.

---

## Part 9 — `forwardToken: true` vs `forwardToken: false`

This is the biggest difference between game services and admin/auth services.

### Game services — `forwardToken: true`

```
kgateway validates JWT
        │
        ▼
Injects x-user-id and x-user-roles headers
        │
        ▼
ALSO forwards the original Authorization: Bearer <token> header
        │
        ▼
Backend receives both the headers AND the original JWT
```

Why? The game backend also has its own `zitadel_jwks_cache_ttl_ms` config — it may do its own secondary JWT validation for certain sensitive operations. The backend can verify the JWT itself if needed.

### Admin service — `forwardToken: false`

```
kgateway validates JWT
        │
        ▼
Injects x-user-id and x-user-roles headers
        │
        ▼
DROPS the Authorization header (token NOT forwarded)
        │
        ▼
Backend only receives x-user-id and x-user-roles
```

Why? The admin backend uses its own `JWT_SECRET` (from AWS Secrets Manager) for its own internal session tokens. It doesn't need the original Zitadel token forwarded — the gateway already proved the user is authenticated. Dropping the token is also slightly more secure (less data in transit to the backend).

---

## Part 10 — The `JWT_SECRET` (Separate from Zitadel)

Every service has a `JWT_SECRET` in `secretsProviderClass`:
```yaml
secretsProviderClass:
  JWT_SECRET: /shunya-dev/dot-tap-jwt-secret
```

This is a **different** JWT secret — stored in AWS Secrets Manager, not related to Zitadel. It's used by the game/admin backend to:
- Sign its **own internal JWT tokens** (e.g., game session tokens)
- Verify requests between internal services (service-to-service auth)
- The Zitadel JWT is for proving who the user is; this JWT_SECRET is for the game's own session management

Think of it like: Zitadel JWT = your passport (who you are), JWT_SECRET = the game's own ticket (you're allowed to play this session).

---

## Part 11 — What is Restate?

Restate is a **distributed workflow engine** — think AWS Step Functions but open source and running inside the cluster.

Games use Restate to manage game state reliably. For example: when a user submits a Sudoku solution, rather than directly hitting the database (which can fail), the game service calls Restate, which durably processes the event (save score, update leaderboard, notify admin, etc.) even if something crashes mid-way.

There are **two Restate instances** in dev:

| Instance | Namespace | Used by |
|---------|-----------|--------|
| `restate-dev` | `restate-dev` | All main Shunya games (sudoku, maze, infinity-loop, arrow, etc.) |
| `restate-chicmic` | `restate-chicmic` | Chicmic team's dot-tap service |

Each game registers its service endpoint with Restate during startup via a `restate-registration-job.yaml` (a Kubernetes Job — runs once on deploy).

Internal Restate URL format:
```
http://restate.restate-dev.svc.cluster.local:8080
```
`restate` = service name, `restate-dev` = namespace, `svc.cluster.local` = K8s internal DNS suffix.

---

## Part 12 — How Admin Panel Connects to Games

### Admin Frontend → Admin Backend

```
Browser (shunya-admin.dev.shunya.app)
    │
    ├── Logs in via admin backend (/admin/auth/token)  ← public route, no JWT needed
    │
    ├── Stores session/token
    │
    └── Calls admin API with token in header
            │
            ▼
     shunya-admin-backend.dev.shunya.app
     (JWT validated at kgateway, forwardToken: false)
```

### Admin Backend → Games (via AWS SQS)

The admin service doesn't call game backends directly for most operations. Instead it uses **AWS SQS queues** — one queue per game:

```
Admin backend
    │
    ├── aws_sqs_arrows_queue_url         → shunya-dev-arrows SQS
    ├── aws_sqs_sudoku_queue_url         → shunya-dev-sliding-puzzle SQS  (note: possible config bug)
    ├── aws_sqs_infinity_loop_queue_url  → shunya-dev-infinity-loop SQS
    ├── aws_sqs_block_fill_queue_url     → shunya-dev-block-fill SQS
    ├── aws_sqs_sequence_recall_queue_url → shunya-dev-sequence-recall SQS
    ├── aws_sqs_memory_card_matching_queue_url → ...
    ├── aws_sqs_sliding_puzzle_queue_url → ...
    └── aws_sqs_spot_the_difference_queue_url → ...
```

Each game backend also polls its own SQS queue (`aws_sqs_job_queue_url`) — when admin sends a message (e.g. "update game config", "flag cheater"), the game picks it up asynchronously.

### Games → Admin Backend

Every game service has this config:
```yaml
admin_microservice_base_url: "https://shunya-admin-backend.dev.shunya.app"
```

Games call the admin backend directly (HTTPS) to fetch game configs, level data, etc. This call goes through kgateway (JWT required — the game sends its own service JWT or the forwarded user JWT).

### Admin Frontend → Some Game Backends (Direct)

The admin frontend also calls a few game backends **directly from the browser** (client-side Next.js environment variables):
```yaml
next_public_sequence_api_base_url: "https://sequence-recall.dev.shunya.app/"
next_public_arrows_api_base_url: "https://arrows.dev.shunya.app/"
next_public_infinity_api_base_url: "https://infinity-loop-backend.dev.shunya.app"
next_public_block_api_base_url: "https://block-fill-backend.dev.shunya.app/"
```

These game backends have public `/api` routes (no JWT at gateway) specifically for admin management operations that the admin frontend calls directly.

---

## Part 13 — How ArgoCD Deploys Everything

ArgoCD is the **CD (Continuous Deployment)** tool. It watches **this Git repo** and automatically deploys changes to the cluster.

Each service folder has an `argocd-application.yaml` (or similar) that tells ArgoCD: "Watch the `shunya-dot-tap/` folder, when it changes, deploy to the cluster using these dev-values.yaml values."

```
Developer pushes to Git
        │
        ▼
ArgoCD detects change (polls every few minutes)
        │
        ▼
ArgoCD runs Helm (renders templates + dev-values.yaml)
        │
        ▼
ArgoCD applies the rendered YAML to EKS cluster
        │
        ▼
Kubernetes rolls out new Pods (RollingUpdate strategy)
```

The CI/CD pipeline (GitHub Actions) builds Docker images, pushes to **AWS ECR**, and updates the `tag:` field in the service's `dev-values.yaml`. That tag change triggers the ArgoCD deployment.

---

## Part 14 — How TLS/HTTPS Works

Each service has a **certificate.yaml** in its templates:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  secretName: dot-tap-helm-auth-tls
  dnsNames:
  - shunya-dot-tap-backend.dev.shunya.app
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

**cert-manager** automatically:
1. Requests a certificate from Let's Encrypt
2. Proves domain ownership via HTTP challenge
3. Stores the certificate in a Kubernetes Secret
4. Renews it before expiry

kgateway reads this secret and uses it for HTTPS termination. Your backends never deal with TLS — they receive plain HTTP from kgateway.

There's also a **wildcard certificate** `*.dev.shunya.app` stored as `shunya-dev-wildcard-tls` — used by the shared `shunya-dev-gateway` that the auth service uses.

---

## Part 15 — Redis (Shared Cache)

All game services share a single Redis instance:
```yaml
redis_host: "redis.redis.svc.cluster.local"
redis_port: "6379"
redis_db: "0"
```

Each service uses its own Redis key prefix to avoid collisions:
- Admin data: `redis_admin_project_key: "Shunya-Admin"`
- Sudoku data: `redis_SUDOKU_project_key: "Shunya-Sudoku"`
- Dot Tap data: `redis_DOT_TAP_project_key: "Shunya-Dot-Tap"`
- User data: `redis_users_project_key: "Shunya-Users"`

---

## Part 16 — Observability (Monitoring)

**SignOz** is the observability platform (like AWS CloudWatch + X-Ray combined, but open source). All services send traces to it:
```yaml
otel:
  endpoint: "http://signoz-otel-collector.signoz.svc.cluster.local:4318/v1/traces"
  environment: "dev"
```

Prometheus metrics are also scraped from each Pod:
```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "8000"
```

**Grafana** is used for dashboards on top of these metrics.

---

## Part 17 — Complete End-to-End Flow for a Game Request

Let's trace a user playing Dot Tap (chicmic team's game):

```
1. User opens app, gets redirected to:
   https://auth.dev.shunya.app  (Zitadel)
   
2. User logs in → Zitadel issues JWT
   JWT contains: sub=user123, roles={player:{}}, aud=373511972211197450

3. App starts a Dot Tap game session:
   POST https://shunya-dot-tap-backend.dev.shunya.app/game/start
   Authorization: Bearer <JWT>
   
4. DNS resolves to AWS NLB (auto-created by kgateway)

5. NLB forwards TCP to kgateway Pod (namespace: kgateway-system)

6. kgateway:
   a. Terminates TLS (HTTPS → HTTP internally)
   b. Matches HTTPRoute: host=shunya-dot-tap-backend.dev.shunya.app, path=/game → auth-route
   c. TrafficPolicy fires:
      - CORS headers added
      - JWT validation:
        * Strips "Bearer " from header
        * Uses cached Zitadel public key (JWKS, cached 1hr)
        * Verifies signature ✓
        * iss = auth.dev.shunya.app ✓
        * aud = 373511972211197450 (in allowed list) ✓
        * not expired ✓
        * Extracts sub → x-user-id: user123
        * Extracts roles → x-user-roles: {player:{}}
        * forwardToken: true → keeps Authorization header
   d. Forwards to dot-tap-service (ClusterIP) port 10000

7. dot-tap Pod receives:
   POST /game/start
   x-user-id: user123
   x-user-roles: {player:{}}
   Authorization: Bearer <JWT>   ← still here because forwardToken: true

8. Dot Tap backend:
   - Reads x-user-id from header (no JWT parsing needed)
   - Creates game session
   - Sends game state event to Restate (restate-chicmic namespace)
   - Restate durably processes: save to DB, notify admin via SQS

9. During the game, backend can call:
   - Admin backend: https://shunya-admin-backend.dev.shunya.app (get level config)
   - Redis: redis.redis.svc.cluster.local:6379 (cache game state)
   - AWS S3: for CDN assets (images, audio)
   - AWS SQS: for async job processing

10. Response flows back:
    Backend Pod → ClusterIP Service → kgateway → NLB → Browser
```

---

## Part 18 — Complete End-to-End Flow for Admin Panel

```
1. Admin opens https://shunya-admin.dev.shunya.app
   (Next.js frontend — NO JWT check at kgateway, just serves HTML)

2. Next.js page loads in browser, makes login request:
   POST https://shunya-admin-backend.dev.shunya.app/admin/auth/token
   (public route — no JWT required)
   
3. Admin backend validates admin credentials, returns session token

4. Admin views game configs:
   GET https://shunya-admin-backend.dev.shunya.app/admin/games
   Authorization: Bearer <admin-session-token>
   
   kgateway validates JWT:
   - forwardToken: false → JWT validated, DROPPED, only x-user-id/x-user-roles forwarded
   - Admin backend trusts x-user-roles header to authorize admin actions

5. Admin updates Sudoku level config:
   Admin backend → sends message to AWS SQS (shunya-dev-sudoku queue)
   Sudoku backend (polling SQS) → picks up message → applies config update

6. Admin views Sequence Recall stats (direct):
   Browser → https://sequence-recall.dev.shunya.app/api/stats
   (Public /api route — no JWT at gateway)
   Sequence Recall backend responds with stats directly to admin frontend
```

---

## Part 19 — Meeting Q&A Cheat Sheet

**Q: Where is JWT validated?**  
A: At kgateway, before the request reaches any backend. The `GatewayExtension` named `zitadel-jwt` does it. No backend service parses JWTs itself — they just read `x-user-id` and `x-user-roles` headers.

**Q: Which services use Zitadel?**  
A: All backend services use Zitadel JWT at the kgateway level. The admin frontend (Next.js UI) does not — it just serves HTML and has no JWT gateway extension. Authentication for the admin frontend happens through the admin backend API.

**Q: What is the difference between dot-tap and other games?**  
A: At the gateway/auth level, they are identical — all use Zitadel JWT with `forwardToken: true`. The one infrastructure difference is Restate: dot-tap uses `restate-chicmic` (chicmic team's own Restate instance), while all other games use `restate-dev` (the main shunya Restate instance).

**Q: What is the `JWT_SECRET` in each service?**  
A: Completely separate from Zitadel. It's the service's own secret key (stored in AWS Secrets Manager) for signing internal session tokens or service-to-service authentication. Zitadel JWT = proves user identity; JWT_SECRET = the game's own session/service tokens.

**Q: How does kgateway know the JWT is real?**  
A: It fetches Zitadel's public keys from `https://auth.dev.shunya.app/oauth/v2/keys` (the JWKS endpoint), caches them for 1 hour, and uses them to verify the JWT's cryptographic signature. Only Zitadel has the private key to sign tokens.

**Q: What happens if the JWT is expired or fake?**  
A: kgateway returns `401 Unauthorized` and the request never reaches the backend.

**Q: How does admin control games?**  
A: Two ways — (1) admin backend sends messages to each game's AWS SQS queue for async config updates, and (2) games call the admin backend directly at startup/runtime to fetch game configs.

**Q: What is Restate?**  
A: A distributed workflow engine (like AWS Step Functions) that runs inside the cluster. Games use it to reliably process game state changes. Two instances: `restate-dev` for most games, `restate-chicmic` for dot-tap.

**Q: How do new game versions get deployed?**  
A: CI builds a Docker image, pushes to AWS ECR, updates the `tag:` in dev-values.yaml in this Git repo. ArgoCD (CD tool) detects the change and deploys to the cluster automatically.

**Q: What is CORS and why does it come before JWT?**  
A: CORS is a browser security rule. When a browser makes a cross-origin request, it first sends an OPTIONS "preflight" request. That preflight must be answered without a JWT check (the browser doesn't send the JWT on preflights). So CORS is handled first, then JWT.

**Q: Where are secrets stored?**  
A: AWS Secrets Manager. Each service has a `secret-provider-class-and-sa.yaml` that mounts secrets from Secrets Manager into the Pod as environment variables using IRSA (IAM Role for Service Account — like ECS task role but for K8s Pods).

**Q: What does `forwardToken: false` on admin mean?**  
A: kgateway validates the JWT but doesn't forward the `Authorization` header to the admin backend. Only the extracted claims (`x-user-id`, `x-user-roles`) are sent as headers. The admin doesn't need the raw JWT — it has its own internal session management.
