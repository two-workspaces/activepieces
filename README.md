# Deploy and Host Activepieces on Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/new/template/activepieces-workflows?referralCode=sLjDXb&utm_medium=integration&utm_source=button&utm_campaign=activepieces-workflows)

[Activepieces](https://www.activepieces.com/) is the open-source alternative to Zapier, Make and n8n: a visual flow builder with hundreds of integrations ("pieces"), webhooks, schedules, branches, loops, code steps, human-in-the-loop approvals, AI agents and an MCP server for your AI tools. This template runs the official Activepieces image the way upstream runs it in production, as a separate app and worker, on Postgres and Redis.

## About Hosting Activepieces

The stack is four services: Activepieces, Worker, Postgres and Redis.

- **App and worker split, like upstream's own compose.** The Activepieces service serves the web UI, the API and incoming webhooks. Flows, triggers and code steps run on the Worker service, which talks to the app over Railway's private network. A heavy flow can't stall the UI, and you scale execution by adding Worker replicas.
- **Upstream's own image, pinned.** Both services run `ghcr.io/activepieces/activepieces` at a fixed release tag (0.92.2), not `latest`. Database migrations run on boot, so upgrading is bumping the tag on both services.
- **Sandboxed code steps that work on Railway.** `AP_EXECUTION_MODE=SANDBOX_CODE_ONLY` runs your JavaScript/TypeScript code steps in an isolated V8 sandbox: they can't read the server's environment variables (your encryption key, JWT secret, database password) or its files. It needs no privileged container, unlike Activepieces' process-sandbox modes, which Railway doesn't support.
- **Secrets generated per deploy.** The encryption key that protects your saved app connections and the JWT secret are generated for every deployment; the worker derives its own token from the JWT secret at boot.
- **Everything in Postgres.** Flows, connections, run logs and uploaded files are stored in Postgres (the `pgvector` image, which Activepieces' vector features need). Redis holds the job queue with an append-only file on its own volume. The app and worker are stateless.

## Common Use Cases

- Replacing Zapier or Make with a self-hosted automation server and no per-task pricing
- Webhook-driven backends: receive an event, transform it in a code step, call APIs and answer the caller
- Scheduled syncs between CRMs, spreadsheets, databases and chat tools
- AI workflows and agents that call OpenAI, Anthropic or other models, with human approval steps
- Exposing your flows and pieces as MCP tools to Claude, Cursor and other AI clients

## Dependencies for Activepieces Hosting

- Postgres 17 with pgvector (included, private network only)
- Redis 8 (included, private network only)

### Deployment Dependencies

- [Activepieces self-hosting documentation](https://www.activepieces.com/docs/install/overview)
- [Environment variables reference](https://www.activepieces.com/docs/install/reference/environment-variables)
- [Activepieces on GitHub](https://github.com/activepieces/activepieces)
- [Template source on GitHub](https://github.com/two-workspaces/activepieces)

### Implementation Details

**Create the owner account first.** Open the Activepieces service's Railway domain as soon as the deploy is green and sign up: the first account becomes the platform owner. Until then anyone who finds the URL could claim the instance. After that, new sign-ups are refused unless the owner invites them (Settings → Team), so the instance is closed by default.

**Webhooks and custom domains.** Webhook and OAuth redirect URLs are built from `AP_FRONTEND_URL` on the Activepieces service. After adding a custom domain, set it to `https://your-domain` and redeploy. Leave the Worker's `AP_FRONTEND_URL` alone: it points at the app's private address on purpose.

**Scaling.** Each worker runs up to `AP_WORKER_CONCURRENCY` flows at once. For more throughput add replicas to the Worker service, or raise the concurrency together with its memory. The Activepieces service can stay at one replica.

**Execution mode.** `SANDBOX_CODE_ONLY` is the mode Activepieces recommends for unprivileged containers and uses on its own cloud: each code step runs in a fresh V8 isolate with no `require`, filesystem or npm packages, while pieces (the built-in integrations) run as trusted code in the worker. If your code steps need npm packages, switch to `UNSANDBOXED` on the Activepieces service. The `SANDBOX_PROCESS` and `SANDBOX_CODE_AND_PROCESS` modes need privileged containers (`CAP_SYS_ADMIN`) and won't work on Railway.

**Back up `AP_ENCRYPTION_KEY`.** It encrypts every saved connection (API keys, OAuth tokens) in the database. If it changes, those connections can't be decrypted and have to be re-created.

**Email (optional).** The Community edition only emails team invitations; fill in the `AP_SMTP_*` variables to send them. Railway only allows outbound SMTP on the Pro plan. Without SMTP, use the invite link Activepieces shows instead. There is no password-reset email in the Community edition: keep the owner password safe, or reset it with the SQL from upstream's [reset-password guide](https://www.activepieces.com/docs/install/troubleshooting/reset-password) in the Postgres service's Data tab.

**Resources.** This stack needs the Hobby plan: on its first boot the app downloads the whole piece catalog (about 13,000 piece versions, roughly 3–4 minutes), and that takes more than the Trial plan's 512 MB. Measured on Railway: the app peaks at about 780 MB during that sync and idles at 430–600 MB afterwards (`NODE_OPTIONS` caps its heap). The worker idles at about 210 MB and adds roughly 170 MB for each warm sandbox, up to `AP_WORKER_CONCURRENCY` (default 2). Postgres needs under 100 MB and Redis about 20 MB, so the whole stack idles at about 0.8–1.1 GB. Right after the first boot Railway's memory graph shows about 2 GB instead: Postgres keeps roughly 700 MB of file cache from importing the catalog, and Railway counts it until Postgres restarts. Redeploy the Postgres service once after the first boot to drop it.

## Why Deploy Activepieces on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying Activepieces on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
