# voussoir

My personal Arch Linux setup: an opinionated installer plus the dotfiles I
actually run, for a ThinkPad T470. The installer also detects an X1 Carbon
Gen 7 and generic Intel/AMD machines and adjusts packages and kernel
parameters accordingly.

The whole thing is LUKS2 (argon2id) + Btrfs + NetworkManager (with iwd as the
Wi-Fi backend) + systemd-resolved, with Hyprland as the desktop.

## The name

A *voussoir* is a wedge-shaped stone in an arch: individually load-bearing,
but only stable once the whole arch is assembled. That is roughly how this
setup works — installer, packages, and dotfiles only hold together as one
structure.

## Layout

Every top-level directory is a [GNU Stow](https://www.gnu.org/software/stow/)
package whose contents mirror `$HOME`:

```
hypr/.config/hypr/        Hyprland configuration — Lua
scripts/.local/bin/       wifi-menu + protonvpn-signin-pty
waybar/.config/waybar/    Waybar bar: config.jsonc, style.css, scripts/
```

Supporting files:

| File | Purpose |
| --- | --- |
| `install.sh` | Runs from the Arch ISO. Partitions, encrypts, pacstraps, chroots. **Destroys the target disk.** |
| `post-install.sh` | Runs after first boot. Installs paru + AUR packages, configures snapper and TLP, stows the dotfiles. Idempotent. |
| `pkglist.txt` | Explicit native packages (`pacman -Qqen`). |
| `aur.txt` | AUR / foreign packages (`pacman -Qqem`). |

## Hyprland configuration

The config is **Lua** and requires **Hyprland 0.55 or newer**.

`hyprland.lua` is a small entry point that `require()`s one module per
section, all in the same directory:

```
hyprland.lua     entry point — just the require() calls
monitors.lua     output, mode, scale
env.lua          environment variables
autostart.lua    exec-once processes
input.lua        keyboard, mouse, touchpad
appearance.lua   general, decoration, animations, dwindle, misc
binds.lua        keybindings
rules.lua        window / workspace rules
```

To change something, edit the module that owns that section. To add a
section, drop in a new module and `require()` it from `hyprland.lua`.

`hyprland.conf.bak` is the previous `.conf` configuration, kept for history
only — Hyprland does not load it, and it should not be edited.
`hyprpaper.conf` stays a plain `.conf` because hyprpaper has no Lua support.

### Verifying config changes

```bash
luac -p hypr/.config/hypr/*.lua                                     # syntax
bash -n install.sh post-install.sh scripts/.local/bin/wifi-menu     # shell syntax
python3 -c 'compile(open("scripts/.local/bin/protonvpn-signin-pty").read(), "protonvpn-signin-pty", "exec")'  # helper syntax
hyprland --verify-config -c hypr/.config/hypr/hyprland.lua          # the real check
shellcheck install.sh post-install.sh scripts/.local/bin/wifi-menu  # if installed
```

`hyprland --verify-config` executes the config, follows `require()` into the
modules, and rejects unknown configuration keys. It needs no running session,
so it is the safest and strongest check available.

## Networking

ProtonVPN's official client requires **NetworkManager**, so NetworkManager is
the control plane. **iwd** stays as the 802.11 backend and is started on demand
by NetworkManager; DNS is handed to **systemd-resolved**. `networkctl` is not
used here — it only fronts systemd-networkd, which is not part of this setup.

The Waybar network icon opens `wifi-menu --wifi`, the VPN icon opens
`wifi-menu --vpn`, and `Super + N` opens the combined menu
(`scripts/.local/bin/wifi-menu`). The menu is entirely fuzzel-driven — no
action launches another GUI.

The Wi-Fi side can:

- show Wi-Fi and VPN state in a status header;
- scan from the cached list and rescan only on request;
- deduplicate multi-band APs, keeping the strongest BSSID;
- connect saved profiles directly, or prompt for a passphrase inline;
- disconnect/reconnect the active network, join hidden networks, and toggle
  the radio;
- manage saved profiles in fuzzel: connect/disconnect, auto-connect on/off,
  auto-connect priority and retries, metered state, Wi-Fi powersave, MAC
  address policy, IPv6 method, WPA/802.1X credentials, and forget network.

The VPN side can:

- run ProtonVPN connects in the background and report the server/IP in a
  notification;
- quick-connect the fastest/random/P2P/Tor/Secure Core server, pick a country
  from the CLI's list, or enter a specific server;
- show VPN status plus tunnel/public IP on demand;
- activate imported NetworkManager VPN / WireGuard profiles;
- sign in or switch accounts with fuzzel username/password/2FA prompts. The
  official CLI is driven through a PTY helper, so no terminal window opens.

The official CLI cannot run while the ProtonVPN desktop app is open. When the
menu detects the app, it offers to close it and then exposes the full CLI
actions; it never launches the app itself.

The ProtonVPN packages are in Arch's `extra` repo and are installed by the
installer/post-install:

```bash
sudo pacman -S proton-vpn-gtk-app proton-vpn-cli
```

Sign in once with `protonvpn signin` (or the GUI); credentials are stored in
gnome-keyring, which ProtonVPN pulls in automatically.

The old `iwmenu-bin` package is no longer used; remove it with
`paru -Rns iwmenu-bin`.

## Installing

`install.sh` wipes the disk it is pointed at. There is no undo.

```bash
# From the Arch ISO
./install.sh --dry-run    # walk the questionnaire, print the plan, change nothing
./install.sh              # for real
```

Then, after reboot and connecting to Wi-Fi:

```bash
git clone <this repo> ~/voussoir && cd ~/voussoir
./post-install.sh --dry-run   # shows every change, including stow actions
./post-install.sh             # applies them
```

Both scripts support `--dry-run` (`-n`) and `--help`. Use `--dry-run` first —
it is the only supported way to test them.

`post-install.sh` never deletes a file to resolve a conflict: anything in the
way is moved to `.stow-backups/<timestamp>/` with its original path preserved.

## Packages

`pkglist.txt` and `aur.txt` describe what the current machine has installed:

```bash
pacman -S --needed - < pkglist.txt          # native packages
paru -S --needed - < aur.txt                # AUR packages (needs paru)
```

Regenerate them after changing what you use:

```bash
pacman -Qqen > pkglist.txt
pacman -Qqem > aur.txt
```

## License

GPL-3.0 — see [LICENSE](LICENSE).
