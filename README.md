# NetBird Connect — GitHub Action

> Securely connect your GitHub Actions runners to your private NetBird network — giving your CI/CD pipelines direct access to private resources without exposing them to the public internet.

> Based on https://github.com/Alemiz112/netbird-connect

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub Actions                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                       Workflow                          │    │
│  │                                                         │    │
│  │   1. Checkout code                                      │    │
│  │   2. ► netbird-connect  ──────────────────────────────┐ │    │
│  │   3. Deploy / Test / Migrate                          │ │    │
│  └───────────────────────────────────────────────────────┼─┘    │
└──────────────────────────────────────────────────────────┼──────┘
                                                           │
                                         Encrypted WireGuard tunnel
                                                           │
┌──────────────────────────────────────────────────────────▼──────┐
│                       NetBird Network                           │
│                                                                 │
│   ┌──────────────────┐         ┌──────────────────────────────┐ │
│   │                  │         │         Private Network      │ │
│   │  NetBird Peer    │◄───────►│                              │ │
│   │                  │  Peer   │  ┌──────────┐  ┌─────────┐   │ │
│   └──────────────────┘  Mesh   │  │  Server  │  │Database │   │ │
│                                │  │          │  │         │   │ │
│                                │  └──────────┘  └─────────┘   │ │
│                                └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**In 4 steps:**
1. **Install** — Downloads and installs the NetBird client on the runner
2. **Authenticate** — Connects using your setup key and registers the runner on the network
3. **Verify** — Pings a known peer IP to confirm the tunnel is live
4. **Done** — All subsequent workflow steps have full access to your private network

---

## Usage

```yaml
- name: Connect to NetBird
  uses: Rocket.Chat/netbird-connect@main
  with:
    setup-key: ${{ secrets.NETBIRD_SETUP_KEY }}
    preshared-key: ${{ secrets.NETBIRD_PSK }}
    test-ip: ${{ vars.NETBIRD_TEST_IP }}
    management-url: https://your-netbird-management-url:443
```

### Full Example — Deploy to a Private Server

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Connect to NetBird
        uses: Rocket.Chat/netbird-connect@main
        with:
          setup-key: ${{ secrets.NETBIRD_SETUP_KEY }}
          preshared-key: ${{ secrets.NETBIRD_PSK }}    # Allows connection with other peers
          test-ip: ${{ vars.NETBIRD_TEST_IP }}          # IP of a NetBird peer to verify connectivity
          management-url: https://your-netbird-management-url:443

      # From here, the runner is on your private network
      - name: Deploy to private server
        run: |
          ssh user@10.0.0.5 "cd /app && ./deploy.sh"

      - name: Run DB migration
        run: |
          psql postgresql://10.0.0.10:5432/mydb -f migrations/latest.sql
```

---

## Inputs

| Input | Required | Default | Description |
|-------|:--------:|---------|-------------|
| `setup-key` | ✅ | — | NetBird setup key used to authenticate the runner. Store as a GitHub secret. |
| `preshared-key` | ✅ | — | NetBird preshared key that allows the runner to communicate with other peers. Store as a GitHub secret. |
| `test-ip` | ✅ | — | IP address of a peer on your NetBird network used to verify the tunnel is active. |
| `management-url` | ❌ | `https://api.netbird.io:443` | Your NetBird management server URL. |
| `hostname` | ❌ | `github-ci-runner` | Name the runner will appear as in your NetBird dashboard. |
| `args` | ❌ | `''` | Extra arguments passed directly to `netbird up` (e.g. `--log-level debug`). |

---

## Setup

### Step 1 — Generate credentials in your NetBird dashboard

1. Go to your NetBird management dashboard
2. Create a **Setup Key** scoped to the appropriate peer groups for your use case
3. Optionally configure a **preshared key** if you want peer-to-peer encryption

### Step 2 — Add Secrets to your repository

Go to **Settings → Secrets and variables → Actions → Secrets** and add:

| Secret name | Value |
|-------------|-------|
| `NETBIRD_SETUP_KEY` | Your NetBird setup key |
| `NETBIRD_PSK` | Your NetBird preshared key |

> [!WARNING]
> Never hardcode these values in a workflow file. Always reference them via `${{ secrets.* }}`.

### Step 3 — Add Variables to your repository

Go to **Settings → Secrets and variables → Actions → Variables** and add:

| Variable name | Value |
|---------------|-------|
| `NETBIRD_TEST_IP` | NetBird IP of a peer in the target network (used to verify connectivity) |

### Step 4 — Add the action to your workflow

```yaml
- name: Connect to NetBird
  uses: Rocket.Chat/netbird-connect@main
  with:
    setup-key: ${{ secrets.NETBIRD_SETUP_KEY }}
    preshared-key: ${{ secrets.NETBIRD_PSK }}
    test-ip: ${{ vars.NETBIRD_TEST_IP }}
    management-url: https://your-netbird-management-url:443
```

---

## Requirements

- Runner OS: **Linux only** (the action will fail immediately on macOS or Windows)
- The runner must have internet access to reach `pkgs.netbird.io` (to install) and your management URL (to connect)

---

## Security Notes

- Always store `setup-key` and `preshared-key` as GitHub Secrets — never hardcode them in workflow files
- The NetBird tunnel is encrypted end-to-end with **WireGuard** — traffic between the runner and your private resources is never exposed to the public internet
- The runner is automatically removed from your NetBird network once the workflow job ends (ephemeral peer)

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Linux only` error | Running on macOS/Windows runner | Switch to `ubuntu-latest` |
| `Timeout waiting for connectivity` | Wrong `test-ip`, peer is offline, or setup key is invalid | Verify the IP is reachable and the setup/preshared keys are correct |
| `Tunnel connects but services unreachable` | Firewall or routing rules | Ensure your network allows traffic from the NetBird IP range |
| `Permission denied` on `netbird up` | Missing `sudo` access | Use a standard `ubuntu-latest` runner (sudo is available by default) |

---

## Action Structure

```
netbird-connect/
  └── action.yml    ← Composite action definition
```

This is a **composite action** — no Docker image, no external dependencies. It runs directly on the runner using standard bash steps.
