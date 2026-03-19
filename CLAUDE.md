# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mattermost plugin that integrates Jitsi Meet video conferencing. Has two components: a Go server plugin (`server/`) and a React/TypeScript webapp (`webapp/`). The plugin provides a `/jitsi` slash command, channel header button, and embedded Jitsi meeting support within Mattermost.

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

### Debug builds

Set `MM_DEBUG=1` to build with debug symbols (Go: `-gcflags "all=-N -l"`, webapp: webpack mode=none).

## Architecture

### Server (`server/`)

- **main.go** — Entry point; calls `plugin.ClientMain()`
- **plugin.go** — Core `Plugin` struct implementing Mattermost plugin hooks (`OnActivate`, `OnConfigurationChange`). Initializes bot user, telemetry, and i18n bundle
- **api.go** — HTTP API endpoints served by the plugin
- **command.go** — `/jitsi` slash command handler with meeting creation logic
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

## Tooling

- **Go:** 1.23+, golangci-lint v1.61.0 (see `.golangci.yml` for enabled linters)
- **Node/Webapp:** TypeScript 3.9, webpack 4, ESLint (see `webapp/.eslintrc.json`), Jest 26
- **CI:** GitHub Actions using shared Mattermost community plugin workflows
- **Docker:** `./docker-make` available for containerized builds
