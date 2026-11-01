# sasori

Hypervisor-level user activity injection for the
[FGNW](https://github.com/Get-Sybers/FGNW) Active Directory training range.

sasori runs inside an LXC on the Proxmox host and drives realistic workstation
behaviour across every VM in the range — keyboard input, lock/unlock cycles,
decoy file opens, and mouse jitter — without any software footprint inside the
guest OSes.

## How it works

| Layer | Mechanism | Guest artefact |
|---|---|---|
| Keyboard input | Proxmox HMP `sendkey` → QEMU emulation layer | None — indistinguishable from physical keys |
| File open | Proxmox QEMU Guest Agent `exec` (runs as SYSTEM) | Real WINWORD/EXCEL process + CIFS type-3 logon |
| Lock / unlock | `Win+L` via sendkey; password typed at lock screen | EventID 4800/4801 on the DC |
| Logoff | `rundll32 ExitWindowsEx` via QGA | EventID 4634/4647 on the DC |
| Mouse jitter | HMP `mouse_move` ±3 px net-zero | None |

## Quick start

```sh
export PVE_HOST=https://pve01:8006
export PVE_USER=emulator@pve
export PVE_TOKEN_ID=sasori
export PVE_SECRET=<token-secret>
export RANGE_NUMBER=1

# Pre-flight check
sasori verify --deep

# Run (dry-run first)
DRY_RUN=true sasori

# Run live
sasori
```

## Configuration

All configuration is via environment variables.

| Variable | Required | Default | Description |
|---|---|---|---|
| `PVE_HOST` | yes | — | Proxmox base URL, e.g. `https://pve01:8006` |
| `PVE_USER` | yes | `emulator@pve` | API token user |
| `PVE_TOKEN_ID` | yes | — | API token ID (part after `!`) |
| `PVE_SECRET` | yes | — | API token secret |
| `RANGE_NUMBER` | yes | — | Ludus range number (integer) |
| `PVE_NODE` | no | `pve` | Proxmox node name |
| `PVE_INSECURE` | no | `false` | Skip TLS verification |
| `PERSONA_TIMELINES` | no | `/etc/sasori/persona-timelines.yml` | GHOSTS persona-timelines file |
| `PERSONA_OVERRIDES` | no | `/etc/sasori/persona-overrides.yml` | GHOSTS persona-overrides file |
| `PERSONA_PASSWORDS` | no | `/etc/sasori/passwords.yml` | `hostname: password` map for unlock injection |
| `GHOSTS_API_URL` | no | — | GHOSTS 9 API URL; enables activity reporting |
| `API_LISTEN` | no | `:8080` | REST API listen address |
| `API_TOKEN` | no | — | Bearer token for API auth (empty = no auth) |
| `TZ_OFFSET` | no | `+00:00` | UTC offset of the range, e.g. `+10:00` |
| `DRY_RUN` | no | `false` | Log actions but do not inject input |

### Proxmox API token permissions

The token needs at minimum:

| Permission | Path | Purpose |
|---|---|---|
| `VM.Audit` | `/nodes/<node>` | List VMs |
| `VM.Monitor` | `/vms/<vmid>` | HMP sendkey |
| `VM.Guest` | `/vms/<vmid>` | QEMU Guest Agent exec |

## Verify subcommand

```
sasori verify [--deep] [hostname ...]
```

Runs a pre-flight check suite and prints a pass/fail table.  `--deep` adds a
Guest Agent `cmd /c echo` round-trip on top of the standard QGA ping.  Exits 0
only when every check passes — safe to use in CI/CD gates.

## REST API

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Liveness + uptime |
| `GET` | `/api/v1/personas` | All loaded personas |
| `GET` | `/api/v1/personas/{hostname}` | One persona |
| `POST` | `/api/v1/personas/{hostname}/trigger` | Immediately run an activity |
| `GET` | `/api/v1/vms` | VM map (hostname → VMID) |
| `POST` | `/api/v1/scheduler/pause` | Suspend all injection |
| `POST` | `/api/v1/scheduler/resume` | Resume injection |
| `GET` | `/api/v1/activity/recent` | Last N events (default 50) |
| `GET` | `/api/v1/activity/stream` | Server-Sent Events stream |

Trigger body: `{"activity": "lock|unlock|logoff|type|open_file|idle_jitter"}`

## Building

```sh
# Native
go build ./cmd/sasori

# All platforms
make all

# Container image
podman build -f Containerfile -t sasori:latest .
```

## Deployment

sasori is deployed as an LXC on the Proxmox host by the
`fgnw_emulator` Ansible role.  See
`blueprints/FGNW/campaign/deploy-tasks/phase1b2-emulator.yml` in the FGNW
repo for the full deployment task.

## License

See [FGNW](https://github.com/Get-Sybers/FGNW) for licensing information.
