# Hermes Agent — Repository Instructions

Hermes shares an agent core across the CLI, messaging gateway, TUI, and Electron Desktop app. Plugins and skills provide most extensions.

This file contains the rules needed for everyday work. Before changing a subsystem, read its linked sections in the [development reference](docs/agent-development-reference.md) and any instructions scoped to that directory. Read references by task; no whole-document read is required. Desktop changes also follow [apps/desktop/AGENTS.md](apps/desktop/AGENTS.md).

## Core invariants

- Preserve the cached conversation prefix: do not mutate past context, swap toolsets, or rebuild the system prompt mid-conversation. Context compression is the intentional exception. Defer prompt-affecting settings to the next session; preserve explicit `--now` behavior where supported.
- Preserve strict message-role alternation. Never inject a synthetic user message mid-loop.
- Keep the core narrow. Prefer existing code → CLI command + skill → prerequisite-gated tool → plugin → MCP catalog entry → new core tool. Reuse existing infrastructure; no consumerless hooks. Large, focused refactors and product expansion at the edges are welcome.
- Reproduce a reported bug on current `main` and check the intended behavior before changing it. Fix the whole affected path, including sibling callers and configuration propagation.
- User-facing non-secret behavior belongs in `config.yaml`; API keys, tokens, and passwords belong in `.env`. Do not add new non-secret `HERMES_*` knobs or opt-out outbound telemetry.
- Native plugin APIs, wire formats, and persisted data must remain compatible: use additive changes, defaults, optional arguments, or explicit migrations. Deprecations require warnings, migration documentation, and at least two subsequent minor releases. New third-party product integrations and memory providers ship outside the core repository.
- Preserve external authorship when salvaging work by cherry-picking or rebase-merging. Automated PR closure is limited to `implemented_on_main`, `cannot_reproduce`, and `incoherent`; taste and scope decisions stay with a human maintainer.

See the [contribution rubric](docs/agent-development-reference.md#contribution-rubric--what-we-want--what-we-dont), [plugin contracts](docs/agent-development-reference.md#plugins), and [prompt-cache policy](docs/agent-development-reference.md#important-policies) for rationale and edge cases.

## Profiles and state isolation

- Use `get_hermes_home()` from `hermes_constants` for profile state and `display_hermes_home()` for user-facing paths. Never hardcode `~/.hermes` or `Path.home() / ".hermes"` for profile state.
- Profile registry operations are deliberately HOME-anchored. Do not change `_get_profiles_root()` to nest profiles under the active `HERMES_HOME`. Profile selection happens before module imports.
- In multiplex mode, credentials **and authorization settings** must use the active secret scope. An installed scope with a missing value returns the caller's default; it must not fall back to the default profile's `os.environ`. Preserve environment reads only for the genuinely unscoped default-profile path or single-profile mode. Adapters use the fail-closed `_get_scoped_secret()` pattern; gateway authorization uses `_auth_env()` / `_platform_gate_env()`.
- Credential-based platform adapters acquire and release scoped token locks so two profiles cannot use the same credential.
- Tests must use a temporary `HERMES_HOME`. Profile tests must also isolate `Path.home()`, because registry paths are HOME-anchored. Never write to a user's real Hermes state.

Read [Profiles: Multi-Instance Support](docs/agent-development-reference.md#profiles-multi-instance-support) before changing profile paths, secrets, admission rules, or adapters.

## Surface capability is a property of the SESSION

Gate surface-specific tools by the current session's source and toolset, never by a process environment flag such as `HERMES_DESKTOP`. `check_fn` answers reachability or explicit user opt-in and is cached process-wide; it must not become a surface gate. Desktop sessions can use remote/cloud runtimes without Desktop environment variables.

The full [session-capability contract](docs/agent-development-reference.md#surface-capability-is-a-property-of-the-session-never-of-the-process-env) covers tool selection and plugin consumers.

## Adding New Tools

Prefer a plugin for user/local tools. A core tool requires an explicit core contribution, registration in `tools/<tool>.py`, and exposure in `toolsets.py`; autodiscovery alone does not expose it. Handlers return JSON strings. Use profile-aware paths, including paths in tool descriptions.

Do not hardcode references to another toolset's tools in schema descriptions. Add conditional references in `model_tools.get_tool_definitions()` only when the referenced tool is available. Do not add pagination that encourages skipping instructional tool content.

Read [Adding New Tools](docs/agent-development-reference.md#adding-new-tools), [Toolsets](docs/agent-development-reference.md#toolsets), and [Plugins](docs/agent-development-reference.md#plugins) for registration and compatibility details.

## Configuration and dependencies

Non-secret options need defaults in `hermes_cli/config.py` and propagation through every affected loader: `load_cli_config()`, `load_config()`, or the gateway's raw YAML path. Bump the config version for migrations, not for a simple new deep-merged key.

Bound runtime PyPI dependencies with a lower floor and upper ceiling; pin git dependencies and GitHub Actions to full SHAs, and CI-only pip installs to exact versions. Update `uv.lock` after changing `pyproject.toml` dependencies. Follow the [dependency policy](docs/agent-development-reference.md#dependency-pinning-policy) and [configuration guide](docs/agent-development-reference.md#adding-configuration) for exact conventions.

## Testing

Activate `.venv` or `venv` for development. **Run Python tests through `scripts/run_tests.sh`, never direct `pytest`.** The wrapper selects the project/worktree environment, isolates credentials and HOME, sets UTC and C.UTF-8, and runs one subprocess per test file without xdist.

```bash
scripts/run_tests.sh tests/gateway/
scripts/run_tests.sh tests/agent/test_foo.py -k test_x  # replace with a real affected file/test
scripts/run_tests.sh                                  # full suite when needed
```

- Exercise real resolution, configuration, security, remote-backend, and I/O paths with real imports in a temporary `HERMES_HOME`; mocks alone do not validate these changes. Do not wire previously unused code into a live path without this validation.
- Test behavior and relationships, not snapshots of model names, config versions, enumeration counts, or source-code text. JS configuration/artifact checks belong in the JS suite so the CI change classifier runs them.
- Run platform-specific behavior on that OS using `linux_only`, `macos_only`, or `windows_only` markers. Do not fake the interpreter's host OS or use bare `skipif`/local marker aliases; read the reference for the live Windows process lane.
- Treat a `FLAKY` pass-on-retry report as a bug. Prefer event synchronization and realistic timing bounds.
- In `ui-tui/`, use `npm run typecheck`, `npm run lint`, and `npm test` as appropriate; see [TUI dev commands](docs/agent-development-reference.md#dev-commands). Follow the Desktop-local rules for Desktop validation.

The [testing guide](docs/agent-development-reference.md#testing) contains wrapper flags, CI placement, OS lanes, and examples. Documentation-only edits need content and link checks; do not add tests just to freeze instruction wording.

## Read by task

The reference preserves detailed architecture, examples, and operational contracts. These sections are required reading when their subsystem is in scope.

| Task | Relevant sections |
| --- | --- |
| Find ownership or follow dependencies | [Project Structure](docs/agent-development-reference.md#project-structure), [File Dependency Chain](docs/agent-development-reference.md#file-dependency-chain), [architecture](website/docs/developer-guide/architecture.md) |
| Agent loop or conversation state | [AIAgent Class](docs/agent-development-reference.md#aiagent-class-run_agentpy), [Important Policies](docs/agent-development-reference.md#important-policies) |
| CLI commands, menus, or terminal output | [CLI Architecture](docs/agent-development-reference.md#cli-architecture-clipy), [Known Pitfalls](docs/agent-development-reference.md#known-pitfalls) |
| TUI, web dashboard, Desktop, or TypeScript | [TUI Architecture](docs/agent-development-reference.md#tui-architecture-ui-tui--tui_gateway), [TypeScript Style](docs/agent-development-reference.md#typescript-style), [Desktop rules](apps/desktop/AGENTS.md) |
| Tools, toolsets, or provider/plugin integration | [Adding New Tools](docs/agent-development-reference.md#adding-new-tools), [Plugins](docs/agent-development-reference.md#plugins), [Toolsets](docs/agent-development-reference.md#toolsets) |
| Settings, dependencies, themes | [Adding Configuration](docs/agent-development-reference.md#adding-configuration), [Dependency Pinning Policy](docs/agent-development-reference.md#dependency-pinning-policy), [Skin/Theme System](docs/agent-development-reference.md#skintheme-system) |
| Skills, skill lifecycle, or bot sessions | [Skills](docs/agent-development-reference.md#skills), [Curator](docs/agent-development-reference.md#curator-skill-lifecycle), [Bot Mode](docs/agent-development-reference.md#bot-mode-appsdesktopsrcpluginshermes-bots) |
| Child agents, scheduled jobs, work queues | [Delegation](docs/agent-development-reference.md#delegation-delegate_task), [Cron](docs/agent-development-reference.md#cron-scheduled-jobs), [Kanban](docs/agent-development-reference.md#kanban-multi-agent-work-queue) |
| Profiles, secrets, platform admission, adapters | [Profiles](docs/agent-development-reference.md#profiles-multi-instance-support), [Known Pitfalls](docs/agent-development-reference.md#known-pitfalls) |
| Updates, process identity, shutdown, or restarts | [Update Pipeline](docs/agent-development-reference.md#update-pipeline-hermes-update), [Gateway lifecycle](docs/agent-development-reference.md#gateway-lifecycle-vs-the-desktop-app), [Known Pitfalls](docs/agent-development-reference.md#known-pitfalls) |
| Gateway control commands or streaming delivery | [Both message guards](docs/agent-development-reference.md#the-gateway-has-two-message-guards--both-must-bypass-approvalcontrol-commands), [Streaming delivery contract](docs/agent-development-reference.md#streaming-delivery-contract-stream-is-the-message-adapters--duplicate-final-class) |
| Test changes or review | [Testing](docs/agent-development-reference.md#testing), [Contribution Rubric](docs/agent-development-reference.md#contribution-rubric--what-we-want--what-we-dont) |

## Review and branch updates

Inspect the working tree before updating a branch. Preserve uncommitted work and a recovery point; merge or rebase onto the target using the project's branch conventions, resolve conflicts deliberately, and inspect the full diff for unrelated reversions. Never discard local work as a routine update step. See [branch-update guidance](docs/agent-development-reference.md#update-the-merge-base-without-discarding-work).

For contribution mechanics, use [CONTRIBUTING.md](CONTRIBUTING.md). In the completion summary, state the behavior changed, relevant validation and its result, and any remaining limitation. Keep temporary diagnostics out of version control unless the task needs a durable artifact.
