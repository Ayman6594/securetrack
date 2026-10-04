# Phase 0: Environment and Repo Setup

Goal: a reproducible Linux workspace, a clean repo, and automated guardrails *before* any feature code.
Each step includes **why**, because that reasoning is what you'll be asked about in interviews.

---

## 1. Linux environment

**Why Linux first?** Servers, containers, CI runners, and cloud VMs are overwhelmingly Linux. Doing
all work there from day one builds the muscle memory the job requires.

### Windows: WSL2 (recommended)
In an **administrator PowerShell**:
```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default-version 2
```
Reboot if asked, then open "Ubuntu" and create your user. Keep the project **inside the Linux
filesystem** (`~/securetrack`), not under `/mnt/c/`: file I/O is much faster and permissions behave correctly.

### Native Linux or a VM
Ubuntu 24.04 LTS or Debian 12 both work.

## 2. Base tooling
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl build-essential python3 python3-venv python3-pip \
                    jq tree net-tools tcpdump openssl ca-certificates

python3 --version      # want 3.11+ (3.12 preferred)
```

### Docker
- **WSL2:** install Docker Desktop for Windows and enable "WSL integration" for Ubuntu, *or* install
  Docker Engine inside Ubuntu following the official docs.
- **Linux:** install Docker Engine and Compose plugin from Docker's official apt repo.
```bash
docker --version && docker compose version
```
**Why:** every later phase (Postgres, Prometheus, Grafana, the API) runs as containers.

## 3. Git and SSH
```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase true

ssh-keygen -t ed25519 -C "securetrack-dev" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub        # add to GitHub → Settings → SSH keys
ssh -T git@github.com
```
**Why Ed25519 keys and a passphrase?** Same reasoning as ADR-0001, and a passphrase protects the key if
your laptop is stolen (threat T17). Consider also enabling **commit signing** later.

## 4. Create the repo
```bash
cd ~ && cp -r /path/to/this/securetrack . && cd securetrack
git init
python3 -m venv .venv && source .venv/bin/activate
python -m pip install --upgrade pip pre-commit ruff
pre-commit install
git add . && git commit -m "chore: phase 0 scaffold"
```
Create an empty GitHub repo named `securetrack`, then:
```bash
git remote add origin git@github.com:<you>/securetrack.git
git push -u origin main
```

## 5. Guardrails
Included in this scaffold:

| File | Purpose |
|---|---|
| `.gitignore` | Keeps `.env`, keys, venvs, and state files out of Git |
| `.pre-commit-config.yaml` | Runs `ruff` and `gitleaks` before every commit |
| `.github/workflows/ci.yml` | Same checks in CI so they can't be bypassed locally |

Run `pre-commit autoupdate` once to bump hook versions to current releases.

### Prove the guardrail works (do this, don't skip)
```bash
echo 'AWS_SECRET_ACCESS_KEY="AKIAIOSFODNN7EXAMPLE/wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"' > leak.txt
git add leak.txt && git commit -m "test: should be blocked"
# expected: gitleaks blocks the commit
rm leak.txt && git reset
```
Note: the string above is AWS's documented **example** key, not a real credential. Never test with real secrets.

## 6. GitHub settings (5 minutes, high value)
- Settings → Branches → protect `main`: require PR, require status checks, no force pushes.
- Settings → Code security: enable Dependabot alerts, Dependabot security updates, secret scanning, push protection.

## 7. Phase 0 checklist
- [ ] `lsb_release -a` shows Ubuntu/Debian; `docker run hello-world` works
- [ ] Repo pushed; CI green on `main`
- [ ] Planted fake secret blocked by pre-commit
- [ ] You can explain: trust boundaries, why Ed25519 over HMAC, why the frame is 89 bytes
- [ ] Threat model read end to end, and you've added **one threat of your own**

## 8. Study notes: be able to explain these
1. What is a trust boundary and where are they in SecureTrack?
2. Why can't a signed frame fit in a legacy BLE advertisement, and what did we do about it?
3. What can a compromised *gateway* do, and what can't it do?
4. Why do we verify the original frame bytes rather than the gateway's JSON?

## Next: Phase 1
Build the frame codec with golden test vectors, then the tracker simulator and `SimTransport`.
