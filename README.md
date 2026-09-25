# XDPA-XFCE-Display-Profile-Applicator

XDPA (XFCE Display Profile Applicator) is a command line tool for the XFCE desktop. It takes one
administrator's display profile and applies it to every user account on the machine, including
accounts created after the profile was set up.

## The problem it solves

On Rocky Linux 8.10, a two monitor extended desktop configured in XFCE can revert to mirroring the
same image on both screens after the machine wakes from sleep. XFCE also keeps display profiles
per user, so a layout one person configures does not carry over to other accounts on the same
machine.

XDPA works around both problems. An administrator configures the layout once, exports it as a
profile, and XDPA copies that profile's configuration into every regular user account and
registers it with XFCE's configuration store (`xfconf`). The layout then survives a sleep and wake
cycle and applies to new accounts as they are created.

## Installing XFCE

XDPA assumes XFCE is already installed. If it is not, follow the
[official Rocky Linux XFCE installation guide](https://docs.rockylinux.org/8/guides/desktop/xfce_installation/).
The key steps, run as root or with `sudo`, are below.

1. Update the system and enable the repositories XFCE needs.

   ```bash
   dnf update
   dnf install epel-release
   dnf config-manager --set-enabled powertools
   dnf copr enable stenstorp/lightdm
   dnf update
   ```

2. Install the XFCE package group and a display manager.

   ```bash
   dnf groupinstall "xfce"
   dnf install lightdm
   ```

3. Switch the display manager and the default boot target.

   ```bash
   systemctl disable gdm
   systemctl enable lightdm
   systemctl set-default graphical.target
   ```

4. Reboot. The machine comes up at an XFCE login screen.

   ```bash
   reboot
   ```

## Installing xdpa

1. Get the script onto the machine.

   ```bash
   git clone https://github.com/metalllinux/XDPA-XFCE-Display-Profile-Applicator.git
   cd XDPA-XFCE-Display-Profile-Applicator
   ```

2. Copy it to `/usr/local/bin` and make it executable.

   ```bash
   sudo cp xdpa /usr/local/bin/xdpa
   sudo chmod 755 /usr/local/bin/xdpa
   ```

3. Add a `/usr/bin/xdpa` symlink.

   ```bash
   sudo ln -sf /usr/local/bin/xdpa /usr/bin/xdpa
   ```

   On Rocky Linux and other RHEL family systems, sudo's default `secure_path` does not include
   `/usr/local/bin`. Without this symlink, `sudo xdpa` fails with "command not found" even though
   `xdpa` itself runs fine when called directly.

### One-time setup for the administrator

`xdpa --export` copies whichever display profile is currently active for the user running it.
Before that command has anything to export, log in as the administrator (or whichever user's
layout should become the system default) and configure a profile in XFCE.

1. Open Display Settings and set the monitor layout and resolution you want.
2. Go to Advanced, then Profiles, and save the current layout as a new profile.
3. Under Connecting Displays, turn on "Configure new displays when connected" and "Automatically
   enable profiles when new display is connected."
4. Close the dialog.

## Usage

xdpa takes one subcommand at a time. Run `sudo xdpa --export`, `sudo xdpa --apply`, and
`sudo xdpa --refresh-user` as root. The other subcommands can run as a regular user.

`--list` shows the display profiles XFCE already knows about for the current user, and marks the
active one.

```console
xdpa --list
Your XFCE Display Profiles:
=================================
* 0123456789abcdef0123456789abcdef01234567 - dual_monitor_extended (ACTIVE)
  Default - Default
  Fallback - Fallback
```

`--export <id> [name]` copies the calling user's active display configuration into system storage
under `/etc/xfce4-display-profiles`, so `--apply` has something to work from.

```console
sudo xdpa --export 0123456789abcdef0123456789abcdef01234567 "dual_monitor_extended"
Exporting profile from user: admin
Extracting profile properties...
Profile exported successfully: 0123456789abcdef0123456789abcdef01234567 - dual_monitor_extended
```

`--apply [id]` propagates the exported profile (or the profile named by `id`) to every regular
user account on the machine, and sets it as the system wide XFCE default for any account created
afterward.

```console
sudo xdpa --apply
Applying profile 0123456789abcdef0123456789abcdef01234567 (dual_monitor_extended) to all users...
  Configuring user: admin
  Configuring user: user1
  Configuring user: user2
  Setting system-wide XFCE default...
Profile applied to 3 users successfully!
Users will see the profile in Display settings after logging out and back in.
The profile will appear in: Display → Advanced → Profiles
```

`--list-system` shows the profile XDPA currently has recorded system wide.

```console
sudo xdpa --list-system
System-wide Display Profiles:
=================================
  0123456789abcdef0123456789abcdef01234567 - dual_monitor_extended (exported by admin) (ACTIVE)
```

`--verify` reports, for every regular user on the machine, whether the exported profile's
configuration file is in place and whether it is registered in that user's live `xfconf` session.

```console
sudo xdpa --verify
System Profile Status:
=================================
Active Profile: 0123456789abcdef0123456789abcdef01234567
Profile Name: dual_monitor_extended
✓ Master configuration exists
✓ System-wide default set

User Status:
  admin: ✓ Profile registered
  user1: ✓ Profile configured (not logged in)
  user2: ✓ Profile configured (not logged in)

Note: Users must log out and back in to see profiles in Display settings.
If a user doesn't see the profile after login, run:
  sudo xdpa --refresh-user <username>
```

`--refresh-user <username>` re-applies the exported profile to one account on demand. Use it for a
user who was not logged in the last time `--apply` ran.

```console
sudo xdpa --refresh-user user1
Refreshing profile for user: user1
Profile: 0123456789abcdef0123456789abcdef01234567 (dual_monitor_extended)
  Staging master configuration...
  Registering profile in xfconf...
  User not logged in, will apply at next login
Profile refresh complete for user1
The user should log out and back in to see the profile in Display settings.
```

`--help` prints usage and the workflow summary below. Running xdpa with an option it does not
recognize also prints this and exits with a non-zero status.

```console
xdpa --help

XDPA - XFCE Display Profile Applicator v2.1

USAGE: xdpa [OPTION]

OPTIONS:
  -l, --list              List your XFCE display profiles
  -e, --export ID [NAME]  Export profile to system storage (requires sudo)
  -a, --apply [ID]        Apply system profile to all users (requires sudo)
  -L, --list-system       List system profiles
  -v, --verify            Verify profile status
  -r, --refresh-user USER Refresh profile for specific user (requires sudo)
  -h, --help              Show this help

WORKFLOW:
  1. Configure your display settings as desired
  2. Find the profile ID:     xdpa --list
  3. Export it system-wide:   sudo xdpa --export <ID> <name>
  4. Apply to all users:      sudo xdpa --apply
  5. Verify status:           sudo xdpa --verify

  If a user doesn't see the profile:
  6. Refresh that user:       sudo xdpa --refresh-user <username>

The profile will be applied to ALL regular users on the system.
Users need to log out and back in to see the changes.
```

Every short flag (`-l`, `-e`, `-a`, `-L`, `-v`, `-r`, `-h`) works the same as its long form above.

XDPA keeps one active system-wide profile at a time. Exporting a new profile with `--export`
replaces the previous system record rather than adding to a history, so `--apply` and
`--refresh-user` always work from whichever profile was exported most recently.

## Licence

XDPA is released under the Apache License, Version 2.0. See [LICENSE](LICENSE) for the full text.

Copyright (c) 2026, Ctrl IQ, Inc. All rights reserved.

```text
Copyright © 2026 Ctrl IQ, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at:

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
