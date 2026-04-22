# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mattermost plugin that integrates Jitsi Meet video conferencing. Has two components: a Go server plugin (`server/`) and a React/TypeScript webapp (`webapp/`). The plugin provides `/jitsi` slash commands (start, persistent, settings, help), a channel header button, and embedded Jitsi meeting support within Mattermost.

## Build & Development Commands

```bash
make all              # lint + test + build bundle
make dist             # build server + webapp + create .tar.gz bundle
make server           # build Go binaries for all platforms
make webapp           # build webapp with webpack
make deploy           # build and deploy to running Mattermost instance
make watch            # rebuild webapp on file changes
make check-style      # run golangci-lint + eslint + tsc type checking
make test             # run Go tests (gotestsum) and Jest tests
make coverage         # generate Go coverage report
make clean            # remove all build artifacts
```

### Running individual tests

```bash
# Single Go test
cd server && go test -run TestFunctionName -v ./...

# Webapp tests
cd webapp && npm run test          # watch mode
cd webapp && npm run test-ci       # CI mode (no watch)

# Linting
cd webapp && npm run lint
cd webapp && npm run fix           # auto-fix lint issues
cd webapp && npm run check-types   # TypeScript type checking
```

### Build notes

- Set `MM_DEBUG=1` to build with debug symbols (Go: `-gcflags "all=-N -l"`, webapp: webpack mode=none).
- Node 18 requires `NODE_OPTIONS=--openssl-legacy-provider` for webpack 4 builds.
- `make clean` removes `build/bin/`, `node_modules/`, and dist. After clean, rebuild `build/bin/manifest` with `go build -o build/bin/manifest build/manifest/main.go` before running `make dist`, or run `npm install` in `webapp/` before building.

## Architecture

### Server (`server/`)

- **main.go** — Entry point; calls `plugin.ClientMain()`
- **plugin.go** — Core `Plugin` struct implementing Mattermost plugin hooks (`OnActivate`, `OnConfigurationChange`). Initializes bot user, telemetry, and i18n bundle
- **api.go** — HTTP API endpoints: `/api/v1/meetings` (start), `/api/v1/meetings/enrich` (enrich JWT), `/api/v1/meetings/token` (generate fresh JWT for persistent meetings), `/api/v1/config` (user config)
- **command.go** — `/jitsi` slash command handler: `start`, `persistent`, `settings`, `help`
- **configuration.go** — Plugin settings management with `OnConfigurationChange` hook
- **randomNameGenerator.go** — Generates human-readable random meeting names

The plugin uses Mattermost's `pluginapi` client library and stores user preferences in KVStore.

### Webapp (`webapp/src/`)

- **index.tsx** — Plugin registration: registers post type component, channel header button, reducer, and loads Jitsi external API
- **components/conference/** — Embedded Jitsi meeting component (floating window)
- **components/post_type_jitsi/** — Custom post type rendering for meeting posts
- **actions/** — Redux actions for starting/opening meetings
- **reducers/** — Redux state for conference visibility and browser state
- **client/** — HTTP client wrapping plugin API calls

Uses React 16, Redux, react-intl for i18n, Enzyme for testing.

### Internationalization

- Server: go-i18n JSON files in `assets/i18n/`
- Webapp: react-intl JSON files in `webapp/i18n/` (20+ languages)

### Plugin Configuration (plugin.json)

Key settings: JitsiURL (server address), JitsiEmbedded (in-app meetings), JitsiNamingScheme (random/UUID/context/ask), JWT auth (JitsiJWT, JitsiAppID, JitsiAppSecret).

### Meeting Types

- **Regular meetings** (`/jitsi start`): JWT generated at creation time with fixed expiration (JitsiLinkValidTime). Link expires after the configured time.
- **Persistent meetings** (`/jitsi persistent`): No JWT at creation time. Each "Join Meeting" click calls `/api/v1/meetings/token` to generate a fresh JWT, so the link never expires. Posts have `meeting_persistent: true` in props.

### JWT Structure

When JWT auth is enabled, tokens include a `context` object with `user` info, `group`, and `channel_id` (the Mattermost channel where the meeting was created). The `channel_id` is used by Jibri to determine the source channel for recordings. Key functions:
- `generateMeetingJwt(meetingID, user, channelID)` — creates a new JWT with user context
- `updateJwtUserInfo(jwtToken, user)` — enriches an existing JWT with the calling user's info (preserves `channel_id` and `group`)
- `signClaims` / `verifyJwt` — low-level JWT signing and verification

## Tooling

- **Go:** 1.23+, golangci-lint v1.61.0 (see `.golangci.yml` for enabled linters)
- **Node/Webapp:** TypeScript 3.9, webpack 4, ESLint (see `webapp/.eslintrc.json`), Jest 26
- **CI:** GitHub Actions using shared Mattermost community plugin workflows
- **Docker:** `./docker-make` available for containerized builds

## Known Build Issues

- `go vet` and `go build` report `undefined: manifest` — this is expected, `manifest` is generated at build time by `build/bin/manifest apply`
- `tsc --noEmit` reports `Cannot find module 'manifest'` — same reason, generated at build time
- These errors do not affect `make dist` builds
