 NetBird Connect — GitHub Action

> Securely connect our GitHub Actions runners to our private NetBird network — giving our CI/CD pipelines direct access to AWS/Cloud resources without exposing them to the public internet.

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
│   │                  │         │            AWS VPC           │ │
│   │ vpn.rocket.chat  │◄───────►│                              │ │
│   │                  │  Peer   │  ┌──────────┐  ┌─────────┐   │ │
│   └──────────────────┘  Mesh   │  │    EC2   │  │   RDS   │   │ │
│                                │  │ Instance │  │Database │   │ │
│                                │  └──────────┘  └─────────┘   │ │
│                                │                              │ │
│                                │  ┌──────────┐  ┌─────────┐   │ │
│                                │  │   EKS    │  │ Elastic │   │ │
│                                │  │ Cluster  │  │  Cache  │   │ │
│                                │  └──────────┘  └─────────┘   │ │
│                                └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**In 4 steps:**
1. **Install** — Downloads and installs the NetBird client on the runner
2. **Authenticate** — Connects using our setup key and registers the runner on the network
3. **Verify** — Pings a known IP to confirm the tunnel is live
4. **Done** — All subsequent workflow steps have full access to our private network

---

## Usage

```yaml
- name: Connect to NetBird
  uses: Rocket.Chat/netbird-connect@main
  with:
    setup-key: ${{ secrets.NETBIRD_SETUP_KEY }}
    preshared-key: ${{ secrets.NETBIRD_PSK }}
    test-ip: ${{ vars.NETBIRD_TEST_IP }}
```

### Full Example — Deploy to a Private EC2 Instance

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
          preshared-key: ${{ secrets.NETBIRD_PSK }} # Allows connection with other peers
          test-ip: ${{ vars.NETBIRD_TEST_IP }}  # IP of our NetBird peer relevant to the workflow

      # From here, the runner is on our private network
      - name: Deploy to private EC2
        run: |
          ssh ubuntu@10.128.0.5 "cd /app && ./deploy.sh"

      - name: Run DB migration
        run: |
          psql postgresql://10.128.0.10:5432/mydb -f migrations/latest.sql
```

---

## Inputs

| Input | Required | Default | Description |
|-------|:--------:|---------|-------------|
| `setup-key` | ✅ | — | NetBird setup key used to authenticate the runner. Store as a GitHub secret. |
| `preshared-key` | ✅ | — | NetBird preshared key that allows the runner to communicate with the network. Store as a GitHub secret. |
| `test-ip` | ✅ | — | IP address of a peer on our NetBird network. Used to verify the tunnel is active (e.g. a bastion/server in the target AWS network) |
| `management-url` | ❌ | `https://vpn.rocket.chat:443` | Our NetBird management server URL. |
| `hostname` | ❌ | `github-ci-runner` | Name the runner will appear as in our NetBird dashboard. |
| `args` | ❌ | `''` | Extra arguments passed directly to `netbird up` (e.g. `--log-level debug`). |

---

## Setup

> [!IMPORTANT]
> **`setup-key` and `preshared-key` are managed exclusively by the Security Team.**
> Do not attempt to generate these yourself — contact **security@rocket.chat** to request them.

---

### Step 1 — Request credentials from the Security Team

Reach out to **security@rocket.chat** and provide:
- The **repository** that needs access
- The **use case** (e.g. "deploy to production EC2", "run DB migrations")

The Security Team will provision a scoped setup key mapped to the correct NetBird groups and routes for your use case.

---

### Step 2 — Add Secrets to the repository

Once you receive the credentials, go to **Settings → Secrets and variables → Actions → Secrets** and add:

| Secret name | Value |
|-------------|-------|
| `NETBIRD_SETUP_KEY` | Setup key provided by the Security Team |
| `NETBIRD_PSK` | Preshared key provided by the Security Team |

> [!WARNING]
> Never hardcode these values in a workflow file. Always reference them via `${{ secrets.* }}`.

---

### Step 3 — Add Variables to the repository

Go to **Settings → Secrets and variables → Actions → Variables** and add:

| Variable name | Value |
|---------------|-------|
| `NETBIRD_TEST_IP` | NetBird IP of a peer in the target network (provided by the Security Team) |

---

### Step 4 — Add the action to the workflow

Use the action in any workflow step after the above secrets and variables are configured:

```yaml
- name: Connect to NetBird
  uses: ./.github/actions/netbird-connect
  with:
    setup-key: ${{ secrets.NETBIRD_SETUP_KEY }}
    preshared-key: ${{ secrets.NETBIRD_PSK }}
    test-ip: ${{ vars.NETBIRD_TEST_IP }}
```

---

## Requirements

- Runner OS: **Linux only** (the action will fail immediately on macOS or Windows)
- The runner must have internet access to reach `pkgs.netbird.io` (to install) and our management URL (to connect)

---

## Security Notes

- `setup-key` and `preshared-key` are **provisioned and rotated by the Security Team** — never generate or share them outside of GitHub Secrets
- Always reference credentials via `${{ secrets.* }}` — never hardcode them in a workflow file
- The NetBird tunnel is encrypted end-to-end with **WireGuard** — traffic between the runner and our AWS resources is never exposed to the public internet
- The runner is automatically removed from our NetBird network once the workflow job ends (ephemeral peer)

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Linux only` error | Running on macOS/Windows runner | Switch to `ubuntu-latest` |
| `Timeout waiting for connectivity` | Wrong `test-ip`, peer is offline, or setup key is invalid | Verify the IP is reachable and the setup/preshared keys are valid |
| `Tunnel connects but services unreachable` | Firewall/security group rules | Ensure the AWS security groups allow traffic from the NetBird IP range |
| `Permission denied` on `netbird up` | Missing `sudo` access | Use a standard `ubuntu-latest` runner (sudo is available by default) |

---

## How the Action Is Structured

```
netbird-connect/
  └── action.yml    ← Composite action definition
```

This is a **composite action** — no Docker image, no external dependencies. It runs directly on the runner using standard bash steps.
