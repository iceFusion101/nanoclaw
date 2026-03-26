# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Context

Personal Claude assistant. Single Node.js process with skill-based channel system. Channels (WhatsApp, Telegram, Slack, Discord, Gmail) are skills that self-register at startup via `src/channels/registry.ts`. Messages route to Claude Agent SDK running in isolated Linux containers. Each group has its own filesystem, memory (`groups/{name}/CLAUDE.md`), and session.

See [README.md](README.md) for philosophy. See [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) for architecture decisions. See [docs/SPEC.md](docs/SPEC.md) for full architecture spec.

## Development

Run commands directly -- don't tell the user to run them.

```bash
npm run dev              # Run with hot reload (tsx)
npm run build            # Compile TypeScript
npm test                 # Run tests (vitest)
npm run test:watch       # Run tests in watch mode
npm run typecheck        # Type-check without emitting
npm run format:check     # Check Prettier formatting
npm run format:fix       # Auto-fix formatting
./container/build.sh     # Rebuild agent container image
```

Pre-commit hook (husky) runs `npm run format:fix` automatically.

Service management:
```bash
# macOS (launchd)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # restart

# Linux (systemd)
systemctl --user start nanoclaw
systemctl --user restart nanoclaw
```

## Architecture

### Message Flow

```
Channel receives message
  -> sender allowlist check (drop or allow)
  -> storeMessage() to SQLite
  -> main loop polls every 2s (POLL_INTERVAL)
  -> trigger detection (@AssistantName regex, main group skips trigger check)
  -> formatMessages() wraps in XML context
  -> GroupQueue enqueues message check
  -> container spawned (or follow-up piped to active container via IPC)
  -> agent streams output via stdout markers (---NANOCLAW_OUTPUT_START---{json}---NANOCLAW_OUTPUT_END---)
  -> onOutput() strips <internal> tags, sends to user via channel
  -> container idles (IDLE_TIMEOUT 30min), then closes
```

### Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel registry (self-registration at startup) |
| `src/group-queue.ts` | Per-group message queue with global concurrency limit (MAX_CONCURRENT_CONTAINERS=5) |
| `src/container-runner.ts` | Spawns agent containers, manages mounts, parses streaming output markers |
| `src/container-runtime.ts` | Docker/Apple Container detection and runtime-specific mount args |
| `src/credential-proxy.ts` | HTTP proxy (port 3001) that injects API credentials into container requests |
| `src/ipc.ts` | Filesystem-based IPC: polls `data/ipc/{group}/` for messages, tasks, commands |
| `src/router.ts` | Message XML formatting and outbound routing |
| `src/task-scheduler.ts` | Cron/interval/once scheduled task execution (polls every 60s) |
| `src/db.ts` | SQLite via better-sqlite3: messages, groups, sessions, tasks, router state |
| `src/config.ts` | Trigger pattern, paths, intervals, container settings from env |
| `src/sender-allowlist.ts` | Per-chat access control (tamper-proof, outside container) |
| `src/mount-security.ts` | Mount allowlist validation for container volumes |
| `groups/{name}/CLAUDE.md` | Per-group memory (isolated) |
| `container/skills/` | Skills loaded inside agent containers (browser, status, formatting) |

### Concurrency Model

- Single Node.js process handles all I/O
- Each group runs in its own Docker/Apple Container
- `GroupQueue` enforces global container limit (default 5), FIFO waiting, tasks prioritized over messages
- Retry backoff: 5s -> 10s -> 20s -> 40s -> 80s (max 5 retries)
- Containers stay idle after output for follow-up messages; close after IDLE_TIMEOUT

### IPC Between Host and Container

Containers communicate via JSON files in `data/ipc/{groupFolder}/`:
- `input/` -- host writes follow-up messages here
- `messages/` -- agent writes outbound messages (authorization checked: main group can send anywhere, others only to own JID)
- `tasks/` -- agent writes commands: `schedule_task`, `pause_task`, `resume_task`, `cancel_task`, `update_task`, `refresh_groups`, `register_group`

Group identity is verified from the IPC directory path, not message content.

### Container Isolation

- Main group: project root (RO), own group folder (RW), IPC (RW)
- Other groups: own group folder (RW), global folder (RO), IPC (RW)
- Additional mounts validated against `~/.config/nanoclaw/mount-allowlist.json`
- Credentials never exposed to containers; injected via credential-proxy at runtime
- Per-group session data in `data/sessions/{folder}/`

### Database Schema (SQLite, `store/messages.db`)

Tables: `chats` (JID, name, channel, is_group), `messages` (content, sender, timestamp), `registered_groups` (JID, folder, trigger, container_config, is_main), `scheduled_tasks` (prompt, schedule_type, cron/interval/once, next_run), `task_run_logs`, `router_state` (key-value), `sessions` (group_folder -> session_id).

## Skills

Four types of skills exist in NanoClaw. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full taxonomy and guidelines.

- **Feature skills** — merge a `skill/*` branch to add capabilities (e.g. `/add-telegram`, `/add-slack`)
- **Utility skills** — ship code files alongside SKILL.md (e.g. `/claw`)
- **Operational skills** — instruction-only workflows, always on `main` (e.g. `/setup`, `/debug`)
- **Container skills** — loaded inside agent containers at runtime (`container/skills/`)

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time installation, authentication, service configuration |
| `/customize` | Adding channels, integrations, changing behavior |
| `/debug` | Container issues, logs, troubleshooting |
| `/update-nanoclaw` | Bring upstream NanoClaw updates into a customized install |
| `/init-onecli` | Install OneCLI Agent Vault and migrate `.env` credentials to it |
| `/qodo-pr-resolver` | Fetch and fix Qodo PR review issues interactively or in batch |
| `/get-qodo-rules` | Load org- and repo-level coding rules from Qodo before code tasks |


## Contributing

Before creating a PR, adding a skill, or preparing any contribution, you MUST read [CONTRIBUTING.md](CONTRIBUTING.md). It covers accepted change types, the four skill types and their guidelines, SKILL.md format rules, PR requirements, and the pre-submission checklist (searching for existing PRs/issues, testing, description format).


## Troubleshooting

**WhatsApp not connecting after upgrade:** WhatsApp is now a separate skill, not bundled in core. Run `/add-whatsapp` (or `npx tsx scripts/apply-skill.ts .claude/skills/add-whatsapp && npm run build`) to install it. Existing auth credentials and groups are preserved.

**Container build cache stale:** `--no-cache` alone does NOT invalidate COPY steps. Prune the builder first, then re-run `./container/build.sh`.

**TypeScript:** ESM modules (`"type": "module"` in package.json), NodeNext resolution. All imports require `.ts` extensions in source.
