# AGENTS.md

## What this repo is

A Docker customization layer on top of the [fazer-ai/chatwoot](https://github.com/fazer-ai/chatwoot) fork. **The Chatwoot source is not checked in** — `Dockerfile.full` clones it at build time. There is no `package.json`, `Gemfile`, test suite, or lint/typecheck config here; the only thing that runs locally is Docker.

To change Chatwoot behavior, edit files under `custom/` and/or `Dockerfile.full`. You cannot edit `app/...` directly — it only exists inside the image after the build clones upstream.

## Build & run

- Build the image (the real build command; `docker compose build` does nothing — the compose file has `image:` but no `build:` context):
  `docker build -f Dockerfile.full -t meu-chatwoot-custom:v1.10 .`
- Run locally: `docker compose up -d` (compose expects image `meu-chatwoot-custom:v1.10`).
- `docker compose exec rails bundle exec rails console` / `... rails db:migrate`
- Generate `SECRET_KEY_BASE`: `docker compose exec rails bundle exec rake secret`

The build clones upstream then runs `pnpm build:sdk` + `vite build` (pnpm pinned to `10.2.0` via corepack). It takes minutes with cache, ~20 min cold. A frontend change is validated only by a successful `vite build`; a backend Ruby change only by booting the container. There are no fast unit tests.

## Version pinning (updating Chatwoot)

The upstream version is pinned in **two places** in `Dockerfile.full` and must change together:
1. `git clone --branch v4.18.0-fazer-ai.126 ...` (Stage 1)
2. `FROM ghcr.io/fazer-ai/chatwoot:v4.18.0-fazer-ai.126` (Stage 2)

The Stage 2 tag must exist as a **published GHCR image** (`ghcr.io/fazer-ai/chatwoot`), not just a Git tag — otherwise the build fails at Stage 2. Use `:latest` for Stage 2 if the image is missing. After bumping, diff every upstream file that `custom/` overrides and merge new upstream methods/getters while preserving local logic. `AGENTE_IA.md` §3 lists the files and common failure modes (e.g. `/build/public/vite not found` → a broken import in a custom Vue file; `NoMethodError` after deploy → upstream added a method our Ruby override lacks).

## Deploy flow (no manual deploy commands)

1. Edit `custom/` or `Dockerfile.full`, commit, `git push origin master`.
2. `.github/workflows/build.yml` builds `Dockerfile.full` (`--no-cache`, `CACHEBUST=run_id`) and pushes `ghcr.io/fabricio-back/chatwoot-gc:latest`.
3. Coolify (`docker-compose.coolify.yml`) pulls `:latest` on "Force Redeploy".

Only pushes to `master` trigger a build. `docker-compose.yml` = local dev; `docker-compose.coolify.yml` = production.

## Custom file map (source of truth = COPY lines in `Dockerfile.full`)

- **Frontend** (built Stage 1): `Sidebar.vue`, `LoginIndex.vue`, `theme-colors.js`, `conversations_getters.js`, `conversations_helpers.js`, `KanbanIndex.vue`. Plus inline `sed` patches that neutralize `kanban.moveisback.com.br` URLs.
- **Backend** (copied Stage 2): `permission_filter_service.rb`, `conversation_finder.rb`, `_conversation.json.jbuilder`, `saleshub_brand.rb`, `notification_listener.rb`, `search_service.rb`, `remove_kanban_script.rb`, `account_dashboard_patch.rb`, `account_theme_initializer.rb`, `vueapp.html.erb`.
- **Not wired into the build** (present in `custom/` but no COPY line — likely stale/orphaned): `_account.json.jbuilder`, `enterprise_unlock.rb`. Never assume a `custom/` file is active; verify it has a COPY line.

## Key customizations

- **Agent access control**: agents see conversations where they are assignee *or* participant (backend `permission_filter_service.rb` + `conversation_finder.rb`; frontend `conversations_helpers.js`). Per-account toggle "Agentes veem todas as conversas" via `account_dashboard_patch.rb`.
- **Per-account theme color**: default green `#23c93e` (`theme-colors.js`), served via `GET /account_theme/:account_id` (`account_theme_initializer.rb`), injected by `vueapp.html.erb`.
- **Kanban de Etiquetas**: `KanbanIndex.vue` replaces the upstream kanban paywall.
- **WhatsApp groups filter**: behind `BAILEYS_WHATSAPP_GROUPS_ENABLED=true`, which is set only in `docker-compose.coolify.yml` (not `docker-compose.yml`).

Full feature detail is in `AGENTE_IA.md`.

## Conventions & gotchas

- Default branch is `master` (not `main`); CI triggers only on `master`.
- Docs and commit messages are written in Portuguese (pt-BR).
- `.env` is gitignored; use `.env.example` as the reference. `docker-compose.override.yml` is also gitignored.
- `AGENTE_IA.md` is the detailed (pt-BR) customization reference but is slightly stale — it references `custom/InternalChat.vue`, which no longer exists (that widget is now handled inline in `Sidebar.vue`).
- `README.md` and `.github/copilot-instructions.md` contain stale commands (`docker compose build`, `custom/ChatList.vue`); trust `Dockerfile.full` and this file over them.
