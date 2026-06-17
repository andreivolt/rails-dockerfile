# the-best-rails-dockerfile

A production-grade, multi-stage Dockerfile for Rails (with Tailwind) that builds a ~140MB Alpine final image. It strips the build toolchain from runtime, tunes jemalloc + YJIT, and assumes Caddy serves static assets as a reverse proxy (a dedicated `caddy` stage bakes the precompiled assets).

## Stack
- **Base image:** `ruby:${RUBY_VERSION}-alpine` (RUBY_VERSION passed as build ARG, no default).
- **DB:** PostgreSQL (`postgresql-dev` to build, `postgresql-client` at runtime).
- **Assets:** Node/npm + Tailwind (`tailwindcss` gem, used at build then deleted).
- **Runtime tuning:** jemalloc via `LD_PRELOAD`, `MALLOC_CONF=dirty_decay_ms:1000,narenas:2,background_thread:true`, `RUBY_YJIT_ENABLE=1`, `BOOTSNAP_READONLY=true`.
- **Web server:** `caddy:2.10-alpine` stage.
- Uses BuildKit `dockerfile:1.7-labs` (`--exclude` on COPY) + apk/bundle cache mounts.

## Architecture (multi-stage)
1. **base** — `WORKDIR /app`; apk-installs `gcompat jemalloc tzdata`.
2. **development** — base + `build-base git nodejs npm postgresql-dev yaml-dev`; copies `src/Gemfile*`, `bundle install`; `EXPOSE 3000`, entrypoint `./bin/docker-entrypoint`, `CMD bin/rails server`.
3. **builder** — base + build toolchain; `BUNDLE_WITH=assets`, `BUNDLE_WITHOUT=development:test`, `RAILS_ENV=production`. Copies `src/`, then in one heredoc: `bundle install` → `bundle exec bootsnap precompile --gemfile app/ lib/` → `SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile` → reconfigures bundler frozen-off + reinstalls runtime-only (`--without development test assets`) → `bundle clean --force`; deletes `~/.bundle`, bundle cache, gem `.git` dirs, `*.c/*.o/*.log/*.h/gem_make.out`; `strip --strip-unneeded` on `*.so`; removes the tailwindcss gem/bin entirely.
4. **production** — base + `postgresql-client`; creates `rails` user/group (uid/gid 1000). `COPY --from=builder` the bundle and `/app` (`--exclude=public/assets`, chown rails). Runs as `USER 1000:1000`, `EXPOSE 3000`, entrypoint `./bin/docker-entrypoint`, `CMD bin/rails server`. Env: `BUNDLE_DEPLOYMENT=true`, `BUNDLE_WITHOUT=development:test:assets`.
5. **caddy** — `caddy:2.10-alpine`; copies builder's `/app/public/assets` + `caddy/Caddyfile`.

## Run / Build
```bash
docker build --build-arg RUBY_VERSION=3.4.4 --target production -t myapp .
docker build --build-arg RUBY_VERSION=3.4.4 --target caddy -t myapp-caddy .
docker run -p 3000:3000 -e DATABASE_URL=... -e SECRET_KEY_BASE=... myapp
```
`docker-entrypoint` runs `db:prepare` on server start. RUBY_VERSION is required (no default).

## Key files
- `/home/andrei/dev/the-best-rails-dockerfile/Dockerfile` — entire project: the 5-stage build.
- `/home/andrei/dev/the-best-rails-dockerfile/README.md` — one-line pitch; ~140MB, Caddy-as-proxy.

## Gotchas
- Rails app must live under `src/`; a `caddy/Caddyfile` is expected; `bin/docker-entrypoint` must exist in the app.
- No build tools in production stage — toolchain confined to builder/development.
- Tailwind gem is deliberately removed post-precompile; assets live only in `public/assets` (served by Caddy, excluded from the Rails runtime image).
