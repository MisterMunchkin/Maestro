# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## What This Is

A fork of [Maestro](https://github.com/mobile-dev-inc/maestro), an open-source mobile UI testing framework. This fork adds two features on top of upstream Maestro that allow AI-powered assertions to run locally via a Claude Code subscription instead of requiring a Maestro Cloud API key.

## Fork-Specific Changes

### 1. Claude Code CLI as AI Prediction Engine

**PR #1** — `MisterMunchkin/claude-code-engine`

Added `ClaudeCodePredictionEngine` (`maestro-ai/.../claudecode/ClaudeCodePredictionEngine.kt`), which shells out to the locally installed `claude` CLI to evaluate AI assertions. It implements the `AIPredictionEngine` interface (same as the existing cloud provider) and supports all three AI operations: `findDefects`, `performAssertion`, and `extractText`.

How it works:
- Writes the screenshot to a temp file, builds a prompt asking Claude to analyze the image and respond with JSON, then invokes `claude -p <prompt> --allowedTools Read --max-turns 2 --output-format text`.
- Sets `ENABLE_CLAUDEAI_MCP_SERVERS=false` to keep prompts clean.
- Parses the JSON response (handling markdown code-block wrapping).
- `isAvailable()` checks whether the `claude` CLI exists on `PATH`.

The `Orchestra` constructor (`maestro-orchestra/.../Orchestra.kt`) was changed so that `AIPredictionEngine` defaults to `ClaudeCodePredictionEngine` when the CLI is available, falling back to `CloudAIPredictionEngine` when a `MAESTRO_CLOUD_API_KEY` is set:

```kotlin
private val AIPredictionEngine: AIPredictionEngine? =
    if (ClaudeCodePredictionEngine.isAvailable()) ClaudeCodePredictionEngine()
    else apiKey?.let { CloudAIPredictionEngine(it) },
```

Error messages were also updated to mention both providers.

### 2. `aiFallback` for Deterministic Assertions

**PR #2** — `MisterMunchkin/ai-fallback-assert`

Added an optional `aiFallback` field to deterministic assertion commands (`assertVisible`, `extendedWaitUntil`). When the deterministic check fails, Maestro automatically retries the assertion using the AI engine before reporting a failure.

Files changed:
- `Commands.kt` — `AssertConditionCommand` gained `aiFallback: String?`, evaluated through the JS engine.
- `YamlElementSelector.kt` — `aiFallback: String?` added to the YAML selector model.
- `YamlFluentCommand.kt` — Extracts `aiFallback` from the selector and passes it to `AssertConditionCommand` for both `assertVisible` and `extendedWaitUntil`.
- `Orchestra.kt` — Wraps `assertConditionCommand` in a try/catch: on `AssertionFailure`, if `aiFallback` is set and an AI engine exists, calls `assertWithAICommand` with the fallback string. If the AI assertion also fails, the **original** deterministic exception is re-thrown.

YAML usage:
```yaml
- assertVisible:
    text: "Welcome"
    aiFallback: "A welcome message is visible on screen"
```

## Using AI Assertions in CI

AI assertions (`assertWithAI`, `aiFallback`) require the `claude` CLI on PATH and valid authentication. Locally, `claude login` handles this interactively. In CI (GitHub Actions, Blacksmith, etc.), use an OAuth token instead.

### One-Time Setup

1. Run `claude setup-token` locally (requires a Claude Pro or Max subscription).
2. Copy the token it outputs (valid for ~1 year).
3. Store it as a CI secret named `CLAUDE_CODE_OAUTH_TOKEN` (e.g., GitHub repo > Settings > Secrets > Actions).

### CI Workflow Steps

Add these to your existing Maestro test job:

1. Install the Claude CLI (it's an npm package)
2. Set the `CLAUDE_CODE_OAUTH_TOKEN` environment variable from your secret
3. Run Maestro tests as normal — `aiFallback` and `assertWithAI` work automatically

### Example (GitHub Actions)

```yaml
steps:
  # ... your existing setup (checkout, Java, emulator/simulator, app build) ...

  - name: Install Claude CLI
    run: npm install -g @anthropic-ai/claude-code

  - name: Verify Claude CLI
    run: claude --version

  # ... install Maestro CLI (build from source or download) ...

  - name: Run Maestro tests
    env:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    run: maestro test .maestro/
```

### How It Works

`ClaudeCodePredictionEngine.isAvailable()` checks whether `claude` exists on `PATH`. When the CLI is found, Maestro uses it for all AI operations. The `CLAUDE_CODE_OAUTH_TOKEN` env var provides authentication — the Claude CLI reads it automatically, no interactive login needed.

Without the CLI or token, AI assertions are skipped: `aiFallback` assertions fall through to the original deterministic error, and `assertWithAI` throws `CloudApiKeyNotAvailable`.

### Notes

- The token uses your **subscription quota** (Pro/Max), not API pay-as-you-go billing.
- Renew the token with `claude setup-token` before it expires (~1 year).
- For automatic token refresh, see the [claude-code-login](https://github.com/grll/claude-code-login) GitHub Action, which stores and rotates OAuth credentials via a GitHub PAT.

## Build & Test

This is a Gradle monorepo. Key commands:

```sh
# Build the CLI (produces a jar in maestro-cli/build/libs/)
./gradlew :maestro-cli:installDist

# Run unit tests
./gradlew test

# Run a specific subproject's tests
./gradlew :maestro-orchestra:test
```

## Project Structure (Key Modules)

| Module | Purpose |
|---|---|
| `maestro-ai` | AI prediction engines (Anthropic cloud client, Claude Code CLI engine) |
| `maestro-orchestra` | Command execution engine — `Orchestra.kt` is the main orchestrator |
| `maestro-orchestra-models` | Data classes for commands, conditions, selectors |
| `maestro-cli` | CLI entry point and configuration |
| `maestro-client` | Platform driver interface (communicates with devices) |
| `maestro-ios` | iOS-specific driver |
| `e2e/demo_app` | Flutter demo app used as the E2E test target (has its own `CLAUDE.md`) |

## Syncing With Upstream

This fork tracks `mobile-dev-inc/maestro` as a remote. To pull in upstream changes:

```sh
git fetch upstream
git merge upstream/main
```

Fork-specific commits sit on top of upstream's `main` branch.
