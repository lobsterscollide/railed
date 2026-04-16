# MASTER CONTEXT FILE
**Machine:** thinkpad | Arch Linux (Omarchy) | kernel 6.19.10-arch1-1
**Last updated:** 2026-04-16 ~13:15 AEST
**Project:** Linux system hardening — multi-session, USB handoff workflow

---

## HOW THIS PROJECT WORKS

Work is coordinated between an **architect** (designs steps, writes handoffs) and an **agent** (Claude, executes them). Handoff files live on a USB drive (`/dev/sda1`) and in `~/Downloads`. The most recent handoff is always the highest-numbered `architect_handoff (N).txt`.

**To mount USB for writing:**
```bash
sudo mount -o uid=1000,gid=1000 /dev/sda1 /mnt/usb
```
Always unmount after: `sudo umount /mnt/usb`

**Sudo in Claude Code:** The Bash tool has no TTY. Run this once at session start if sudo is needed:
```bash
echo 'newt ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/newt-nopasswd
```
Remove when done: `sudo rm /etc/sudoers.d/newt-nopasswd`

**hardening-kit pacman repo gotcha:** `/etc/pacman.conf` has a `[hardening-kit]` entry pointing to `/mnt/kit/pkg`. This path is only available when the kit drive is mounted. If it's not mounted, ALL pacman/yay installs fail. Workaround:
```bash
sudo sed -i '/^\[hardening-kit\]/,/^$/s/^/#/' /etc/pacman.conf
# ... run installs ...
sudo sed -i '/^#\[hardening-kit\]/,/^$/s/^#//' /etc/pacman.conf
```

---

## CLAWBOT USAGE LESSONS (read before starting a session)

Clawbot (OpenClaw + Anthropic API) uses pre-paid credits — they do NOT reset. Rate limits (30k tokens/min) reset per minute but credits are consumed permanently. When credits run out, Clawbot stops working until topped up at console.anthropic.com.

**To avoid burning credits fast:**
- **Start a fresh session for each task.** The TUI accumulates context — the longer a session runs, the more tokens each message costs. One long session costs far more than several short focused ones.
- **Give tight, specific tasks.** "Set PasswordAuthentication no in sshd_config" costs far less than "sort out SSH security". Vague tasks make the model reason at length before acting.
- **Do not ask Clawbot to update herself.** The `update.run` tool runs npm, takes ~3 minutes, peaks at ~4.8GB RAM, disconnects the TUI, and costs tokens on every retry when the rate limit hits.
- **Avoid tasks that loop on failure.** If a tool call fails, Clawbot retries autonomously — each retry burns tokens against the same rate limit, compounding the problem.
- **System tasks cost more than questions.** Each shell command is a round-trip (tokens in + tokens out). A 10-step sysadmin task might cost 10–15× what a simple question costs.
- **When the TUI shows "stuck":** check `journalctl --user -u clawbot -n 20` — it's likely a rate limit retry loop or credit exhaustion, not a bug.

---

## CURRENT STATE (post-2026-04-16 afternoon session)

### ⚠ ONE OUTSTANDING ACTION BEFORE MACHINE IS FULLY HARDENED

**Secure Boot is still DISABLED in firmware.** All keys enrolled, all binaries signed, BLAKE2B bug fixed — just needs the UEFI toggle:
1. F1 at ThinkPad splash → Security → Secure Boot → **Enable** → Save & Exit
2. After reboot verify:
```bash
sbctl status                              # Must show: Secure Boot: ✓ Enabled
stat /boot/limine.conf | grep Access     # Should show 0600 (fmask/dmask took effect)
ping -c3 10.10.0.1                        # WireGuard tunnel up
sudo aa-status | head -3                  # AppArmor loaded
```
**If Limine fails to boot:** Hit F12 at splash → "Omarchy Recovery (Direct UKI)" — bypasses Limine entirely.

---

### Hardening — ALL PHASES COMPLETE ✓

**Phase 1–3:**
- AppArmor: enabled, 161 profiles loaded, 78 enforcing
- WireGuard: running, tunnel live (10.10.0.2 → 10.10.0.1); boot race fixed (After=network-online.target)
- 9 hardening actions via hardened-api: sysctl (log_martians, suid_dumpable), SSH hardening, PAM umask 027, fail2ban (1h ban), git shell → nologin
- sshd: `PermitRootLogin no`, `Protocol 2`, **`PasswordAuthentication no`** (key-only; authorized_keys currently empty = no remote SSH login possible)
- rkhunter: whitelisted, baseline set (174 files, 2026-04-16)
- nftables Docker forwarding applied
- hardened-api: locked to 127.0.0.1:8080 (was listening on all interfaces)

**Phase 4 — Secure Boot (pending UEFI toggle only):**
- sbctl keys enrolled, all EFIs signed
- BLAKE2B hash bug fixed; `zzz-limine-hashes.hook` keeps hashes in sync
- Boot0002 (Limine) PARTUUID corrected; Boot0003 recovery entry added
- `/boot` fstab: fmask=0177, dmask=0077 (takes effect on reboot)

**Phase 5 — Clawbot:**
- OpenClaw gateway on 127.0.0.1:18789 (Restart=always)
- LUKS2 volume at `/var/encrypted/clawbot.img` → `/mnt/clawbot`
- API key at `/mnt/clawbot/clawbot.env` (chmod 600)
- Terminal: `clawbot` (auto-unlock + gateway start + TUI)
- Default model: `anthropic/claude-sonnet-4-6`
- AIDE baseline: set 2026-04-16 (`/var/lib/aide/aide.db.gz`, 263k entries)

**Security scan timers (all active):**
- rkhunter.timer: daily, Persistent=true
- clamav-freshclam.service: continuous, 3.6M signatures
- clamav-scan.timer: weekly (/home /tmp /var/tmp)
- AIDE: manual (`sudo aide --check`) — last clean 2026-04-16

**hardened-api current posture (as of 2026-04-16 13:09):**
- `overall_severity: critical` — driven entirely by:
  - `chkrootkit not installed` (deferred — ftp.chkrootkit.org unreachable)
  - Package CVEs: djvulibre, pam, libxml2 (critical severity) — **no upstream fix available**, monitor only
  - All other findings: warnings/info, no fix available upstream
- `ssh`, `firewall`, `services`, `ports`, `fileperm`, `sysctl`, `clamscan`, `rkhunter`: all ✓ PASS

---

## SOFTWARE INSTALLED

| Package | Version | How to launch |
|---|---|---|
| Slack | 4.49.81 | `slack` |
| KeePassXC | 2.7.12 | `keepassxc` |
| Firefox | 149.0 | `firefox` |
| Brave | 1.89.132 | `brave` |
| NymVPN | 1.27.0 | `sudo systemctl enable --now nym-vpnd` then launch app |
| OpenClaw (Clawbot) | 2026.4.14 | `clawbot` |

**NymVPN:** Two packages — `nym-vpn-app-bin` (GUI) + `nym-vpnd-bin` (daemon). Daemon must run first. Requires NymVPN account credentials.

---

## WHAT TO DO NEXT (in order)

### 1. Enable Secure Boot — see above ⚠

### 2. YubiKey / pam-u2f
Tools installed (`yubikey-manager`, `pam-u2f`, `libfido2`) but PAM not configured.
**Await architect direction — do not configure PAM blind.**

### 3. Install chkrootkit (deferred)
`yay -S chkrootkit` — ftp.chkrootkit.org:21 was unreachable. Retry when available.

### 4. SSH key setup (if remote access ever needed)
Currently `PasswordAuthentication no` + empty `authorized_keys` = SSH disabled effectively.
To enable remote access: add public key to `~/.ssh/authorized_keys`.

### 5. Enable Clawbot on login (optional)
`systemctl --user enable clawbot` — decide on volume unlock flow first.

### 6. linux-hardened kernel (optional)
Clarify with architect.

### 7. Anthropic API top-up
console.anthropic.com → Plans & Billing. Consider setting a monthly spend cap.

---

## KEY SYSTEM REFERENCE

**Disk:**
- `/boot` (EFI): nvme0n1p1, vfat, PARTUUID=4b52f092-8636-4187-b9aa-5aaecd0d832a
- Root: nvme0n1p2, LUKS → btrfs (subvols: @, @home, @pkg, @log)
- `/boot` fstab: fmask=0177, dmask=0077 — set, takes effect on reboot

**Boot entries:**
- Boot0002: Limine → `/EFI/limine/limine_x64.efi` (first in BootOrder)
- Boot0003: Omarchy Recovery → `/EFI/Linux/omarchy_linux.efi` (F12 to select, bypasses Limine)
- Boot001B: NVMe0 generic → BOOTX64.EFI (tertiary fallback)
- Boot0000/0001 (Windows/Ubuntu): stale, harmless

**hardened-api:**
- Auth: `Authorization: Bearer localdev`
- Check: `GET http://127.0.0.1:8080/api/v1/checks`
- Harden: `POST http://127.0.0.1:8080/api/v1/harden` + `X-Harden-Confirm: yes`
- Bind: 127.0.0.1:8080 only (loopback)

**WireGuard:** wg0, 10.10.0.2/24 → server 70.34.197.213:51820

**Pacman hooks:**
- `zz-sbctl.hook` — signs EFIs after kernel update
- `zzz-limine-hashes.hook` — updates BLAKE2B hashes after signing
- `/usr/local/bin/limine-update-hashes` — hash update script

**nftables:** Type=oneshot — `inactive (dead)` with exit 0 is CORRECT.

**Clawbot:**
- Service: `~/.config/systemd/user/clawbot.service` (Restart=always)
- Config: `~/.openclaw/openclaw.json`
- Auth: `~/.openclaw/agents/main/agent/auth-profiles.json` (Anthropic token)
- Gateway: 127.0.0.1:18789
- Wrapper: `/usr/local/bin/clawbot`
- Volume: `/var/encrypted/clawbot.img` → `/mnt/clawbot`
- API key: `/mnt/clawbot/clawbot.env` (chmod 600)

---

## SESSION HISTORY

| Session | Date | Key work |
|---|---|---|
| Pre-session | 2026-04-15 AM | AppArmor cmdline, wg-quick@wg0 disabled, mkinitcpio -P |
| Session 1 | 2026-04-15 PM | Post-reboot verify, 9 hardening actions, Secure Boot Phase 4, WG fix, EFI signing, efibootmgr |
| Session 2 | 2026-04-15 evening | BLAKE2B hash bug fix, limine-hashes hook, Boot0003 recovery, timers, AIDE, /boot fstab, Slack |
| Session 3 | 2026-04-16 early AM | Phase 5: Clawbot LUKS, OpenClaw, gateway service, Anthropic auth, default model, `clawbot` cmd |
| Session 4 (Clawbot) | 2026-04-16 AM | WG boot race fix, rkhunter+AIDE baselines, hardened-api → loopback only; hit rate limit + credit exhaustion |
| Session 5 | 2026-04-16 PM | SSH password auth disabled, nobody expiry set; token efficiency lessons documented |

---

## SIGNED EFI BINARIES (all ✓)

```
/boot/EFI/limine/limine_x64.efi
/boot/EFI/BOOT/BOOTX64.EFI
/boot/EFI/BOOT/BOOTIA32.EFI
/boot/EFI/Linux/omarchy_linux.efi
/boot/vmlinuz-linux
```

**Boot chain:** UEFI → Boot0002 → omarchy_linux.efi (UKI)
**Recovery:** F12 → Boot0003 (direct UKI, bypasses Limine)

---
*This file: ~/master_context.md — update at end of each session*
