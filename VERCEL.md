# Vercel build and runtime audit

Dossier 1.9.2 is designed for one persistent Node.js server, PostgreSQL,
Chromium and a persistent `/data` volume. A successful Vercel build does not
establish that all of its features work on Vercel Functions.

## Reproducible build

- Use Node.js **24.x**, as in the Dockerfile and CI. Authentication imports
  `node:crypto`'s `argon2`; Node 22 fails at `promisify(argon2)` during page
  collection, even after compilation and TypeScript succeed.
- Use Corepack and the exact Yarn release pinned in `packageManager`.
- Install with `corepack yarn install --immutable`.
- Build with `corepack yarn build`. This explicitly generates Prisma using
  `prisma7.config.ts` before Next.js compiles. It needs no database connection
  or production secrets. Do not run migrations as part of a preview build.
- The `prisma-client` generator writes to `src/generated/prisma`, matching
  the `@/*` TypeScript alias and imports of `client` and `enums`. Keep these
  generated files ignored; regenerate rather than editing or committing them.
- `prisma7.config.ts` is detected by the locked Prisma 7.10 CLI (confirmed in
  the original deployment logs); the build also selects it explicitly.
- `.yarnrc.yml` uses the node-modules linker. Disabled dependency install
  scripts are distinct from this explicit generation command. The original
  Vercel logs show generation and compilation succeeding with scripts disabled.
  Do not enable all dependency scripts or change locked versions just to hide
  these warnings. Validate required binaries on the target platform.
- `vercel.json` records install/build commands. Root directory is the repository
  root, framework Next.js, and output directory uses the framework default.
  `output: "standalone"` remains necessary for the supported Docker image.
- The signed-in route group's layout explicitly uses dynamic rendering so
  child pages do not attempt database reads during prerendering before the
  layout's session check. Runtime validation still requires all secrets.

## Required runtime configuration

Set sensitive values through the hosting platform's environment settings:

| Variable | Requirement |
| --- | --- |
| `DATABASE_URL` | Reachable PostgreSQL URL (`postgres://` or `postgresql://`), compatible with `@prisma/adapter-pg`; an Accelerate URL is not accepted by this app. |
| `DOSSIER_PASSWORD` | Dossier's Argon2 hash format, generated with `corepack yarn hash-password`; never a plaintext password or a generic PHC hash. |
| `DOSSIER_SECRET` | Stable random secret of at least 32 characters, used for sessions and encryption of saved credentials. Preserve it across deployments. |
| `DOSSIER_PUBLIC_URL` | Canonical HTTPS origin, recommended for Google OAuth callbacks. |
| `DOSSIER_DATA_DIR` | Persistent writable storage in the supported server deployment; `/tmp` on Functions would only be temporary and instance-local. |

AI and job-search credentials are optional until those features are used;
they can be configured in Settings. The default local AI endpoints cannot reach
a user's laptop from Vercel. Google integration needs an OAuth client and the
correct `/api/google/callback` redirect. See `.env.example` for all options.
Do not expose secrets with `NEXT_PUBLIC_` prefixes or commit environment files.

## Database release steps

Before serving real requests, verify the target database and backup policy,
then apply the existing migrations and seed built-in themes:

```sh
corepack yarn prisma migrate deploy --config prisma7.config.ts
corepack yarn prisma db seed --config prisma7.config.ts
```

The first migration requires PostgreSQL's `citext` extension. The migration
role must have the needed permissions. Review existing migration history before
applying to an existing database. Do not use `db push`, reset or demo seeds on
production. Vercel does not execute `docker/start.sh`, which normally performs
these steps. Generation alone neither creates tables nor seeds themes.

## Runtime blockers in the current implementation

1. **Jobs and cancellation:** `src/lib/jobs/runner.ts` stores the queue,
   running controllers and listeners in process memory, and launches work with
   unawaited promises. Persisting job rows does not make execution durable.
   Each cold start also marks *all* queued/running jobs failed, including jobs
   another instance may still be running. A shared worker/queue with ownership,
   leases and retry rules is needed before scaling across instances.
2. **Schedules:** instrumentation starts process-local timers for backups,
   inbox checks, the job agent, startup refreshes and retention. These require
   a persistent process today; migrate them to authenticated scheduled handlers
   with distributed locking and durable execution before relying on Functions.
3. **PDFs:** `src/lib/pdf/render.ts` launches Playwright's local Chromium and
   opens `http://localhost:${PORT || 3000}`. The Dockerfile installs the browser
   and system libraries; Vercel's normal Next.js build does not. Tracing
   `playwright-core` does not install Chromium. Both browser packaging and the
   authenticated print-page origin need a tested deployment design.
4. **Files:** backups and PDF ZIP downloads are written under `/data` and read
   in later requests. Functions do not provide this shared persistent volume.
   Moving paths to `/tmp` does not solve persistence or cross-instance access.
5. **Uploads and long tasks:** audit file import/export sizes and multi-step AI
   calls against the project's request/response and execution limits. Raising a
   timeout alone does not repair the queue, timers or local-file assumptions.

The existing single-instance Docker deployment supplies the architecture the
application expects. A complete Functions port is a separate runtime change;
the build correction does not silently disable these features or claim they work.

## License

Retain the upstream AGPL-3.0-only license, copyright notices and attribution.
Operators of a modified network service must provide its corresponding source
under the upstream license; keep the fork's changes available.

## References

- [Vercel Node.js runtime](https://vercel.com/docs/functions/runtimes/node-js)
- [Vercel Function limits](https://vercel.com/docs/functions/limitations)
- [Vercel Cron Jobs](https://vercel.com/docs/cron-jobs/manage-cron-jobs)
- [Yarn script policy](https://yarnpkg.com/configuration/yarnrc/#enableScripts)
