# AGENTS.md

## Cursor Cloud specific instructions

### Project overview

AiToEarn is an AI-powered content marketing platform with three main sub-projects:

| Service | Directory | Tech | Port | Notes |
|---|---|---|---|---|
| Backend (Nx monorepo) | `project/backend/` | NestJS 11, pnpm, Nx 21 | 3002 (server), 7001 (channel) | Two apps: `aitoearn-server` and `aitoearn-channel` |
| Web Frontend | `project/web/` | Next.js 14, React 18, pnpm | 6060 (dev) | |
| Electron Desktop | `project/aitoearn-electron/` | Electron 33, npm | — | Not needed for web dev |

### Infrastructure (Docker)

MongoDB 7.0 and Redis 7 must be running locally. Start them with:

```bash
sudo dockerd &>/tmp/dockerd.log &
sudo docker start aitoearn-mongodb aitoearn-redis 2>/dev/null || \
  (sudo docker run -d --name aitoearn-mongodb -p 27017:27017 -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=password mongo:7.0 && \
   sudo docker run -d --name aitoearn-redis -p 6379:6379 redis:7-alpine redis-server --requirepass password)
```

### Backend local config

The backend apps use `local.config.js` files (gitignored) loaded via the `--configuration=local` flag in `nx serve`. These must be created at:
- `project/backend/apps/aitoearn-channel/config/local.config.js`
- `project/backend/apps/aitoearn-server/config/local.config.js`

Key settings: MongoDB at `127.0.0.1:27017` (user: `admin`, password: `password`), Redis at `127.0.0.1:6379` (password: `password`). The `logger.feishu` section must be omitted (not set to empty strings) since `z.url()` validation rejects empty strings. The `aliGreen` credentials must be non-empty placeholder strings (e.g. `'placeholder'`) for the Alibaba Cloud SDK credential check.

### Running services

```bash
# Backend channel service (start first)
cd project/backend && NX_NO_CLOUD=true npx nx serve aitoearn-channel --configuration=local --skip-nx-cache

# Backend main server (in separate terminal)
cd project/backend && NX_NO_CLOUD=true npx nx serve aitoearn-server --configuration=local --skip-nx-cache

# Web frontend
cd project/web && pnpm dev
```

**Important**: Always use `NX_NO_CLOUD=true` and `--skip-nx-cache` for backend services to avoid Nx cloud cache permission issues in Cloud Agent environments.

### @nestjs/mongoose @Prop decorator caveat

When adding `@Prop()` to a property typed as `any` in Mongoose schemas, always specify `type: mongoose.Schema.Types.Mixed` explicitly. Without it, `@nestjs/mongoose@11` throws `Cannot determine a type` at runtime. See `libs/mongodb/src/schemas/publishing-task-meta.schema.ts` and `publishRecord.schema.ts` for examples.

### Lint

- **Backend**: `cd project/backend && pnpm lint` — runs Nx lint across all 14 backend projects (0 errors, ~196 warnings)
- **Web**: `cd project/web && pnpm lint` — has pre-existing formatting errors; use `pnpm lint:fix` for auto-fixable issues

### Build

- **Backend**: `cd project/backend && NX_NO_CLOUD=true npx nx build <project-name> --skip-nx-cache` (e.g. `aitoearn-server`, `aitoearn-channel`)
- **Web**: `cd project/web && pnpm build`

### Git hooks

The backend has `simple-git-hooks` with `lint-staged` pre-commit hook that runs `pnpm lint-staged` from the backend directory. Use `--no-verify` when committing from the repository root since the hook expects a `package.json` in the current directory.

See `project/backend/DEVELOPER_GUIDE.md` for comprehensive backend development documentation and `project/backend/CLAUDE.md` for coding conventions.
