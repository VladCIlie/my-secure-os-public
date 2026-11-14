# My Secure OS

A declaratively managed, git-enforced hardened Linux workstation — a thin
custom layer on top of Debian/Ubuntu/Mint that treats the endpoint the way
you would treat a production server: configuration-as-code, drift correction,
application allow-listing, and no reliance on user discipline.

## How it works

1. This repository holds the desired state of the machine
   (`ansible-workstation/playbook.yml` and the files it deploys).

2. A systemd timer (`workstation-sync.timer`) fires hourly, with a
   randomized delay. Each activation runs `workstation-sync.service`,
   which is a one-shot unit that executes `ansible-pull`.

3. `ansible-pull` clones this repo using a fine-grained, read-only
   GitHub token (stored in `/root/.netrc`) and immediately applies
   the playbook against the local machine.

4. Because the playbook re-runs on a schedule, the machine continuously
   converges back to the committed state: manual changes to protected
   files (which are locked with the immutable attribute between runs)
   are reverted within about an hour.

On a fresh install, `bootstrap-workstation-sync.sh` sets up steps 2–3:
it asks for the token, installs the service and timer, and optionally
runs the first pull immediately.

## What the playbook enforces

| Area | Mechanism |
|------|-----------|
| No Snap | purge snapd, delete state dirs, APT pin `-10` blocks reinstall |
| Curated app installs | Flatpak (system-wide) from a filtered Flathub remote |
| No user-level Flatpak | `/usr/local/bin/flatpak` wrapper rejects `--user`; polkit allows only the curated install/uninstall/update actions for active local users |
| Auto-updates | `unattended-upgrades` for APT; systemd timer + `flatpak update --system` for apps |
| Execution allow-listing | fapolicyd: only package-manager-trusted binaries and system Flatpaks may execute; AppImages and roaming scripts in $HOME denied with audit |
| Browser lockdown | Brave/Chromium enterprise policies: autofill off, password manager off, guest mode off, data cleared on exit, AI features disabled, Brave telemetry/rewards/wallet off, forced extension list |
| DNS filtering | systemd-resolved pointed at a filtering resolver (DNS-over-TLS), plus secure DoH pinned in the browser |
| Anti-tamper | enforced files marked immutable (`chattr +i`) at the end of each run |

## Repo layout


bootstrap-workstation-sync.sh # one-shot installer for fresh machines ansible-workstation/ playbook.yml # the whole convergence definition files/ block-apps.pref # APT pinning: block alt. browsers custom_policy.json # Brave/Chromium enterprise policies fapolicyd.rules # fapolicyd allow/deny rules flathub.filter # Flathub remote filter


## Usage

```bash
sudo ./bootstrap-workstation-sync.sh            # standard machine
sudo ./bootstrap-workstation-sync.sh --gaming   # adds Steam/gamemode stack
```

## Some Considerations
The bootstrap script prompts for a fine-grained GitHub read-only token, writes it to /root/.netrc (mode 600), installs the service + timer, and runs the first pull immediately.

Edit before deploying
Filtering-resolver profile IDs in ansible-workstation/playbook.yml (resolver drop-in) and files/custom_policy.json (DoH template).
mandatory_flatpaks and the cookie allowlist in custom_policy.json.
Notable engineering details
fapolicyd's control FIFO race: the playbook restarts the daemon, inspects the journal and fails with the daemon's real error (instead of a confusing missing-FIFO error), then polls for the FIFO before fapolicyd-cli --update.
Immutable attributes are unlocked at the top of each run and re-locked at the end, so every ansible-pull remains idempotent.
Legacy state (old monolithic fapolicyd.rules, stale mangohud package, snap residue) is explicitly cleaned up so first convergence succeeds on machines with prior history.
