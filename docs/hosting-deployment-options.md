# Hosting & Deployment Options for the AI Cyber Value Creator Stack

> **Status:** Planning / research. Not yet implemented.
> **Date:** 2026-06-28
> **Scope:** How to let our users (operators) run the Paperclip + `ai-cyber-value-creator`
> plugin + `harper-agentic-company` template with the least friction — ideally a
> one-click / template-driven deploy we control.
> **Private:** this file is gitignored; it is internal planning, not for upstream.

---

## 1. The goal

Give our users a **template we control** that they can **deploy with minimal steps**,
ideally a "Deploy to X" button or link. They bring their own account + LLM keys; we
own and version the template content.

Two distinct things this could mean — the build and the costs differ a lot:

- **Scenario A — Host the whole always-on stack** (Paperclip server + UI + Postgres +
  persistent workspace + heartbeat agents) so a user gets a working instance in a browser.
  This is the "deploy the product" reading.
- **Scenario B — Agent execution sandbox only.** A controlled image/snapshot where the
  *agents'* execution runs, while Paperclip itself is hosted elsewhere. This is what the
  existing Paperclip **sandbox-provider plugins** already do.

Paperclip is an **always-on, stateful app** (Node server + Postgres + persistent volume +
internal heartbeat timers). That fact drives every recommendation below: a **PaaS app host**
(Railway/Render/Fly) fits Scenario A; **sandbox infra** (Daytona/Cloudflare) fits Scenario B.

---

## 2. What Paperclip already ships

`packages/plugins/sandbox-providers/` contains provider plugins: **daytona, cloudflare, e2b,
modal, novita, kubernetes, exe-dev**. These make Paperclip *drive* a sandbox to run agent
execution (Scenario B). They are **not** app hosts.

- **Daytona plugin** (`sandbox-providers/daytona`): uses `@daytonaio/sdk`, supports
  `snapshot`- and `image`-based creation, API key set in *Instance Settings → Environments*.
- **Cloudflare plugin** (`sandbox-providers/cloudflare`): uses `@cloudflare/sandbox` (a
  `Sandbox` Durable Object on Cloudflare Containers). Operator deploys a **bridge Worker**
  (template included in the package) exposing a JSON HTTP surface; Paperclip talks to it.

### Key integration hook (verified)
`packages/db/src/runtime-config.ts` resolves the DB connection in priority order:
1. `DATABASE_URL` env var (`source: "DATABASE_URL"`)
2. paperclip-env file `DATABASE_URL`
3. `config.database.connectionString` (mode `postgres`)
4. else **embedded Postgres** (dev default `postgres://paperclip:paperclip@127.0.0.1:<port>/paperclip`)

➡️ **Setting `DATABASE_URL` as a secret is all it takes to point a hosted Paperclip at an
external Postgres.** Migrations run on boot. State (companies, workspaces, instance data)
lives under `PAPERCLIP_HOME` — that path needs a persistent volume. The e2e tests also use
`PORT`, `PAPERCLIP_HOME`, `PAPERCLIP_INSTANCE_ID`.

---

## 3. Platform landscape & rate comparison

| Platform | One-click button | Managed Postgres + volume | Pricing model | Always-on full-stack est. | Creator kickback |
|---|---|---|---|---|---|
| **Railway** | ✅ button + marketplace | ✅ | usage-metered (**actual** use) | **~$25–60/mo** | ✅ 15–25% |
| **Render** | ✅ Blueprint + "Deploy to Render" | ✅ (disks $0.25/GB-mo) | fixed instance tiers | ~$25–50/mo | ❌ |
| **Fly.io** | ⚠️ CLI `fly launch` (no button) | ✅ | per-second, scale-to-zero capable | ~$15–45/mo | ❌ |
| **DO App Platform / Heroku** | ✅ button (app spec / app.json) | ✅ | fixed tiers | ~$20–50/mo | ❌ |
| **Daytona** | ❌ (we'd build it) | ❌ (it's a sandbox) | **provisioned** while running | ~$169–243/mo (off-label as host) | ❌ |
| **Cloudflare Containers** | ❌ | ❌ (sandbox) | active-CPU + awake-RAM, scale-to-zero | ~$83–120/mo (off-label) | ❌ |

### Normalized rates (per-hour where useful)

| Resource | Daytona | Cloudflare Containers | Railway | Fly.io |
|---|---|---|---|---|
| vCPU-hour | $0.0504 (provisioned) | $0.072 (**active only**) | ~$0.0274 ($20/vCPU-mo, **actual**) | per-second machine pricing |
| GiB-hour RAM | $0.0162 (provisioned) | $0.0090 (provisioned, awake) | ~$0.0137 ($10/GB-mo, **actual**) | bundled in machine size |
| GB disk | $0.000108/GB-h | $0.000252/GB-h | $0.15/GB-mo volume | $0.15/GB-mo volume |
| Egress | — | — | $0.05/GB | pay-as-you-go |
| Floor / free | $200 one-time credits | $5/mo Workers + monthly free tier | $5/mo Hobby / $20 Pro (credit counts toward usage) | pay-as-you-go, card required |

**Billing-model differences that matter:**
- **Daytona:** bills **provisioned** vCPU+RAM continuously while running. Stop → disk only.
  Default sandbox 1 vCPU/1 GB; **per-org cap 4 vCPU / 8 GB / 10 GB**.
- **Cloudflare:** memory+disk billed on **provisioned size while awake**, CPU on **active use**
  (10 ms granularity), **scales to zero** on idle. Instance types: lite(1/16,256MiB)…
  standard-3(2,8GiB,16GB), standard-4(4,12GiB,20GB).
- **Railway:** bills **actual measured** vCPU+RAM by the minute → cheap for idle-ish apps.
- **Fly:** per-second machine billing; **scale-to-zero only wakes on inbound HTTP** (a problem
  for Paperclip heartbeats — see §6).

---

## 4. Cost models

### Scenario A — host the whole always-on stack
- **Railway ~$25–60/mo** (actual-usage metering; managed Postgres + volume included in usage).
- **Render ~$25–50/mo** (fixed tiers: web $7+ / Postgres $6+ / disk $0.25/GB-mo).
- **Fly ~$15–45/mo** (always-on Machine + managed Postgres + volume).
- **Daytona ~$169–243/mo** (provisioned 24/7; architecturally off-label as an app host).
- **Cloudflare ~$83–120/mo** on rates, but **wrong tool** for a stateful always-on app.

### Scenario B — bursty per-agent execution
Example: ~100 tasks/mo × 10 min, 2 vCPU/8 GiB, ~40% CPU-active.
- **Cloudflare ~$6.5/mo** (≈ the $5 Workers floor; scale-to-zero + active-CPU make idle free).
- **Daytona ~$4/mo if the sandbox is diligently stopped between tasks**, but it balloons to
  $120–170/mo if left running (no auto-sleep). Footgun.

### Baselines
- **User's own machine:** ~$0 ongoing (sunk hardware + ~$3–5/mo electricity 24/7), but local
  setup + the machine must stay on for continuous agent work.
- **Flat VPS (Hetzner/DO):** $5–20/mo for always-on — beats every managed option if the user
  will operate it.
- **LLM/adapter spend dwarfs hosting** in all cases — compute is the small line item.

---

## 5. Recommendation

- **"Deploy the whole product in one click" → Railway.** Native **Deploy button +
  marketplace + 15–25% creator kickback**, managed Postgres + volumes, actual-usage billing
  (cheap), deploys into the *user's* account. Strictly better answer to the original ask than
  building a Daytona launcher ourselves.
- **Cheapest always-on + max runtime control → Fly.io** (CLI flow, see §6).
- **Daytona / Cloudflare are complementary, not competitors:** host the always-on Paperclip
  **control plane** on a PaaS, and offload **bursty agent execution** to a scale-to-zero
  sandbox (Cloudflare/e2b/Daytona) or run it in-container. Offer the sandbox provider plugins
  as an optional add-on.
- We can ship **multiple paths** and let operators choose by usage pattern.

---

## 6. Fly.io — detailed implementation notes (for later development)

Fly has **no web button or marketplace** — it is CLI-driven. "Controlling the template" =
**two artifacts we own and publish:**

1. **A public Docker image we control** — e.g. `ghcr.io/harperaa/paperclip-acvc:<version>` —
   the template content: steel-paperclip build + `ai-cyber-value-creator` plugin +
   `harper-agentic-company` template + adapter CLIs, with an entrypoint that runs DB
   migrations and boots the server on `$PORT`, reading `DATABASE_URL` and persisting state
   under the mounted volume. We version + push updates; users pull them.
2. **A small public "launch repo"** users clone — `fly.toml` pinning our image + a `deploy.sh`
   wrapper that runs the flyctl sequence and prompts for keys.

Updates: push a new image tag → users run `fly deploy` (or `./deploy.sh --update`).

### User steps — simplest path (the `deploy.sh` we ship)
```bash
# 1. one-time: install CLI + sign in (Fly needs a card — pay-as-you-go)
curl -L https://fly.io/install.sh | sh
fly auth signup                 # or: fly auth login

# 2. get the template
git clone https://github.com/harperaa/paperclip-fly && cd paperclip-fly

# 3. run it — prompts for app name, region, LLM keys; prints the URL
./deploy.sh
```

### What `deploy.sh` runs (manual/transparent version)
```bash
# create the app from OUR fly.toml (pins the image), don't deploy yet
fly launch --no-deploy --copy-config --name <app> --region <region>

# managed Postgres → auto-sets the DATABASE_URL secret on the app
fly mpg create --name <app>-db --region <region>     # or legacy: fly postgres create
fly postgres attach <app>-db --app <app>             # writes DATABASE_URL secret

# persistent volume for Paperclip state (companies, workspaces, instance data)
fly volumes create paperclip_data --size 10 --region <region> --app <app>

# their secrets (which keys depends on the default adapter)
fly secrets set --app <app> OPENAI_API_KEY=sk-... ANTHROPIC_API_KEY=sk-ant-...

# pull our image, run migrations, boot, open
fly deploy --app <app>
fly open --app <app>
```

### `fly.toml` essentials we bake in
```toml
app = "<set-per-user>"
[build]
  image = "ghcr.io/harperaa/paperclip-acvc:<tag>"    # our controlled template
[env]
  PORT = "8080"
  PAPERCLIP_HOME = "/data"                            # state on the volume
  PAPERCLIP_INSTANCE_ID = "default"
[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "off"
  min_machines_running = 1                            # always-on (heartbeats — see below)
[[mounts]]
  source = "paperclip_data"
  destination = "/data"
[[vm]]
  size = "performance-2x"                             # ~2 vCPU / 4 GB; size for agent runs
```

### Caveats / decisions
- **Always-on, not scale-to-zero.** Paperclip heartbeats are internal timers, not inbound
  HTTP, so Fly's scale-to-zero (wakes on request) would stop agents firing on schedule. Use
  `min_machines_running = 1`. → steady billing (~$15–45/mo).
- **Volume = one Machine.** State on a Fly volume attaches to a single Machine — correct for
  single-tenant Paperclip; do **not** horizontally scale it.
- **Agents run in-container.** codex_local/claude_local are child processes in the Machine, so
  the image must bundle those CLIs + Node; CPU/RAM spikes during runs → size the VM
  (performance-2x+), or offload execution to a sandbox provider plugin and keep the VM small.
- **Postgres options:** Fly Managed Postgres (`fly mpg`, current managed path) vs legacy
  `fly postgres` (you operate it) vs running the postgres image yourself with a volume
  (cheapest, you operate). `attach` sets `DATABASE_URL` → picked up by `runtime-config.ts`.
- **⚠️ License exposure (applies to Railway/Render too).** Our distribution model is a
  **password-protected, license-gated zip**. A **public** Docker image bundling the plugin
  puts those bytes in a registry anyone can pull — bypassing the gate. Options:
  - (a) keep the image **private** and issue pull tokens only to licensed users (adds a
    `fly`/registry-auth step), or
  - (b) have the image **entrypoint check a license key** the user sets as a secret.
  Decide this **before** publishing any hosted template.

### Fly vs Railway, for this path
- **Fly:** cheapest always-on, scriptable to ~3 commands, full runtime control — but **CLI
  only (no button), no marketplace, no kickback**, user installs flyctl + adds a card.
- **Railway:** real **button + marketplace + kickback** and managed Postgres/volumes built in
  — less runtime control, slightly higher cost, far lower friction for the user + revenue
  share for us.

---

## 7. Railway — notes (for the button path)

- **"Deploy on Railway" button** (Markdown/HTML embed) + **template marketplace**. We define a
  template (Paperclip app from our image + managed Postgres + volume + prompted env vars),
  publish it, users click → deploys into **their** Railway account.
- **Template kickback:** **15%** of usage the template generates (**25%** with active support
  via the template queue), paid as credits or cash. Railway has paid ~$1M to template authors.
- **Pricing:** Hobby $5/mo (incl. $5 usage) / Pro $20/mo, then **$20/vCPU-mo + $10/GB-RAM-mo**
  metered on **actual** usage (by the minute), volumes $0.15/GB-mo, egress $0.05/GB.
- Same **license-exposure** caveat as Fly: the plugin lives in the deployed image.

---

## 8. Open decisions / next steps

- [ ] Decide license-gate strategy for hosted images (private registry + tokens vs entrypoint
      license-key check). Blocks publishing any hosted template.
- [ ] Pick primary path: **Railway button** (lowest user friction + kickback) and/or **Fly**
      (control + cost).
- [ ] Build the controlled **Docker image** (steel-paperclip + plugin + company template +
      adapter CLIs + migration/boot entrypoint reading `DATABASE_URL`, state under `/data`).
- [ ] Build the **launch artifacts**: Railway template (services + volume + env prompts) and/or
      Fly launch repo (`fly.toml` + `deploy.sh`).
- [ ] Decide whether agent execution stays **in-container** or is **offloaded** to a sandbox
      provider plugin (affects VM sizing + cost).
- [ ] Verify Daytona/Cloudflare API **CORS** if we ever pursue the client-side "launcher
      button" for Daytona (Option 3 from the discussion).

---

## 9. Sources

- Daytona: [pricing](https://www.daytona.io/pricing), [snapshots](https://www.daytona.io/docs/en/snapshots/), [sandboxes](https://www.daytona.io/docs/en/sandboxes/), [preview URLs](https://www.daytona.io/docs/en/preview/)
- Cloudflare: [Containers pricing](https://developers.cloudflare.com/containers/pricing/), [new CPU pricing changelog](https://developers.cloudflare.com/changelog/2025-11-21-new-cpu-pricing/)
- Railway: [pricing](https://docs.railway.com/pricing), [publish/share templates](https://docs.railway.com/templates/publish-and-share), [template kickbacks](https://docs.railway.com/templates/kickbacks)
- Fly.io: [fly deploy / --image](https://fly.io/docs/flyctl/deploy/), [fly.toml reference](https://fly.io/docs/reference/configuration/), [secrets](https://fly.io/docs/apps/secrets/), [Fly Launch](https://fly.io/docs/reference/fly-launch/)
- Comparison: [Northflank AI-sandbox pricing](https://northflank.com/blog/ai-sandbox-pricing), [Railway vs Render](https://northflank.com/blog/railway-vs-render)
