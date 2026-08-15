# AGENTS.md

Guidance for AI coding agents (Claude Code, etc.) working in this repository.

> Note: this repo's `.gitignore` explicitly excludes `CLAUDE.md` ("Claude
> configuration"), so agent guidance lives here in `AGENTS.md` instead.

## What This Is

**NotHotDog** is an open-source platform for testing and evaluating AI
agents/chatbots (`AgentEvaluation/NotHotDog`, marketed as **nothotdog.ai**).
It lets a developer point NotHotDog at their conversational agent's HTTP
endpoint, auto-generate test case variations, run them through configurable
personas, and score the results — including LLM-powered hallucination
detection and custom validation rules. This repository is the **Community
Edition**; a separate proprietary Enterprise Edition adds multi-tenant orgs,
RBAC, custom personas/metrics, SSO, and CI/CD integrations (gated in this
codebase via `src/config/features.ts`'s `isFeatureEnabled`/`requireEnterprise`
helpers, driven by `NEXT_PUBLIC_EDITION=enterprise`).

Stack: Next.js 15 (App Router) + React 18 + TypeScript, Prisma ORM over
PostgreSQL, Clerk for auth (per README; not yet wired into `src/app` at the
time of writing — see Gotchas), LangChain + the Anthropic SDK for the LLM
integration, Radix UI + Tailwind CSS (shadcn "new-york" style) for the UI.

## Layout

| Path | Purpose |
|------|---------|
| `src/app/` | Next.js App Router pages (`src/app/tools/`: `runs`, `test-cases`) and API routes (`src/app/api/tools/*/route.ts`: `agent-config`, `agent-rules`, `analyze-result`, `generate-tests`, `persona-mapping`, `test-agent`, `test-runs`, `test-variations`, `validate`). |
| `src/components/` | React components: `common/`, `config/`, `navigation/`, `tools/` (incl. `metrics/`, `test-runs/`), `ui/` (shadcn primitives, incl. `ui/test-sets/`). |
| `src/services/agents/claude/` | The Claude-driven QA agent: `apiHandler.ts`, `conversationHandler.ts`, `conversationProcessor.ts`, `qaAgent.ts`, `validationService.ts`, `validators.ts`. |
| `src/services/db/` | Data-access layer over Prisma, one service per domain (`agentConfigService`, `conversationService`, `metricsService`, `personaMappingService`, `personaService`, `testRunService`, `testService`). |
| `src/services/llm/` | Model registry: `enums.ts` (`LLMProvider`, `AnthropicModel`, `OpenAIModel`), `config.ts` (`MODEL_CONFIGS`, `PROVIDER_MODELS`), `modelfactory.ts`, `types.ts`. |
| `src/services/metrics/hallucinationDetector.ts` | LLM-powered hallucination scoring. |
| `src/services/persona/` | Persona prompt generation (`promptGenerator.ts`). |
| `src/services/prompts/` | Prompt templates for runs and test-case generation. |
| `src/lib/langchain/` | LangChain glue: `memory/conversationMemory.ts`, `prompts/conversation.ts`. |
| `src/lib/` | Cross-cutting: `prisma.ts` (client singleton), `api-utils.ts` (`withApiHandler` — standardizes API route responses/error shape), `errors.ts` (`AppError` hierarchy: `ValidationError`, `AuthorizationError`, ...), `env-validation.ts` (`validateEnv`, currently only requires `ANTHROPIC_API_KEY`), `api-client.ts`, `storage.ts`, `utils.ts`, `validations.ts`. |
| `src/hooks/` | Client hooks: `useAgentConfig`, `useTestExecution`, `useTestRuns`, `useTestVariations`, `useAppError`/`useErrorContext`, `useLocalStorage`. |
| `src/types/` | Shared TypeScript types (`agent`, `chat`, `context`, `metrics`, `persona`, `runs`, `test`, `test-sets`, `variations`, ...). |
| `src/config/features.ts` | Community vs. Enterprise feature flags (`FEATURES`, `isFeatureEnabled`, `requireEnterprise`). |
| `prisma/schema.prisma` | PostgreSQL schema — agent configs/headers/outputs/descriptions, personas, persona mappings, test scenarios, test runs, validation rules, conversations/messages. |
| `prisma/migrations/*.sql` | Hand-written SQL migrations (not the standard `prisma migrate` directory format — see Gotchas). |
| `check-messages.sql` | Standalone ad hoc query, not part of the migration set. |

## Commands

```bash
npm install                      # install dependencies
npx prisma generate               # generate the Prisma client
npx prisma db push                 # push schema.prisma to the DB (no migration history)
cp .env.example .env.local         # README references this; verify it exists before relying on it

npm run dev                       # Next.js dev server
npm run build                     # production build
npm start                          # run production build
npm run lint                       # next lint
npm test                           # jest
npm run test:watch                  # jest --watch
```

Required environment variables (per README / `src/lib/env-validation.ts`):
`DATABASE_URL` (Postgres), `ANTHROPIC_API_KEY` (the only var
`validateEnv()` actually enforces at runtime), plus optionally
`OPENAI_API_KEY`, `DEEPSEEK_API_KEY`, `GOOGLE_API_KEY`,
`NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`/`CLERK_SECRET_KEY`, and
`NEXT_PUBLIC_APP_URL`. Set `NEXT_PUBLIC_EDITION=enterprise` to flip on the
Enterprise feature flags in `src/config/features.ts` (their actual
implementations are not present in this Community repo).

## Conventions

- **Path alias**: import app code via `@/*` → `src/*` (`tsconfig.json`,
  mirrored in `components.json`'s `aliases` for the shadcn CLI).
- **API routes return a standard envelope.** Wrap Next.js route handlers in
  `withApiHandler` (`src/lib/api-utils.ts`), which normalizes success to
  `{ success: true, data }` and failures to
  `{ success: false, error: { message, type, statusCode } }`, logging via
  `logError`. Throw one of the typed errors in `src/lib/errors.ts`
  (`AppError` base; `ValidationError` → 400, `AuthorizationError` → 401, …)
  rather than returning ad hoc error shapes.
- **LLM provider/model registry is centralized.** `src/services/llm/enums.ts`
  declares the supported `LLMProvider`s and model enums;
  `src/services/llm/config.ts`'s `MODEL_CONFIGS`/`PROVIDER_MODELS` is the
  single source of context-window/output-token/temperature defaults per
  model — extend there, don't hardcode model strings elsewhere. Only
  `Anthropic` and `OpenAI` are modeled as `LLMProvider` values today, even
  though the README's env example also lists `DEEPSEEK_API_KEY` and
  `GOOGLE_API_KEY`.
- **Community vs. Enterprise gating** goes through
  `isFeatureEnabled('SOME_FLAG')` / `requireEnterprise('Feature name')` from
  `src/config/features.ts`, never a raw `process.env.NEXT_PUBLIC_EDITION`
  check inline.
- **UI components**: shadcn/ui primitives ("new-york" style, `neutral` base
  color, CSS variables on) live in `src/components/ui/`; add new primitives
  via the shadcn CLI (`components.json`) rather than hand-rolling Radix
  wrappers that already exist there.
- **Styling**: Tailwind CSS (`tailwind.config.js`) with `tailwindcss-animate`
  and `tailwind-scrollbar` plugins; use `cn()`/`clsx`+`tailwind-merge`
  (`src/lib/utils.ts`) for conditional class composition, matching existing
  components.

## Testing

- `npm test` runs Jest (`jest`, `jest-environment-jsdom`,
  `@testing-library/react`, `@testing-library/jest-dom` are installed), but
  **there is no `jest.config.*` file and no `*.test.*`/`__tests__` files
  anywhere in the repo** — the test command currently has nothing to run.
  Before adding tests, add a Jest config (Next.js's `next/jest` preset is the
  natural fit given this is a Next.js app) and confirm `npm test` actually
  discovers them.
- CONTRIBUTING.md asks contributors to "include unit tests for any new
  features or bug fixes" — treat that as the standing expectation even
  though no test suite exists yet to model from.

## Gotchas

- **`CLAUDE.md` is gitignored** (`.gitignore` line under "# Claude
  configuration"). Don't expect a committed `CLAUDE.md` to exist or persist —
  this `AGENTS.md` is the durable place for agent guidance in this repo.
- **`src/lib/.env` is committed to git** (`git ls-files` includes it) and
  contains only a `validateEnv()` helper — despite the `.env`-style filename
  and location, it is source code (`src/lib/env-validation.ts` is the
  sibling that actually calls it), not a secrets file. Don't confuse it with
  a real dotenv file, and don't assume `.gitignore`'s blanket `.env` rule
  kept it out — it was added before/without matching that pattern.
- **No ESLint config file exists** at the repo root
  (`.eslintrc*`/`eslint.config.*` are both absent) despite `npm run lint`
  invoking `next lint`, which will prompt to create one on first run in an
  interactive shell.
- **Migrations aren't Prisma-managed migrations.** `prisma/migrations/`
  contains hand-written, arbitrarily-named `.sql` files (`init.sql`,
  `add_cascade_deletes.sql`, `remove_organizations.sql`, ...), not the
  timestamped-folder format `prisma migrate dev` generates. The README's
  setup flow uses `prisma db push` (schema sync, no migration history), not
  `prisma migrate deploy` — treat `schema.prisma` as the source of truth and
  apply the `.sql` files manually/via `db push` rather than expecting
  `prisma migrate` commands to work out of the box.
- **Clerk auth is referenced but not obviously wired up.** The README lists
  Clerk env vars and cites it as the auth provider in the architecture
  diagram, but no `middleware.ts`, `ClerkProvider`, or `@clerk/nextjs`
  dependency is present in `package.json` at the time of writing — verify
  current auth wiring before assuming routes are protected.
- **Package manager**: both `package-lock.json` and `yarn.lock` are
  gitignored, so there's no committed lockfile to match — pin versions
  carefully if reproducibility matters.
- CONTRIBUTING.md's clone URL and code-style link point at a fork
  (`vedhsaka/NotHotDog`); the canonical upstream per the README badges and
  issue/discussion links is `AgentEvaluation/NotHotDog`.
