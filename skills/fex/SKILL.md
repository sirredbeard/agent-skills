---
name: fex-rootfs-recovery
description: Diagnose and repair FEX-Emu RootFS selection on AArch64 Ubuntu hosts, including extracted RootFS setup for installing persistent x86-64 applications.
---

# FEX RootFS recovery

Treat the FEX package and the x86-64 RootFS as two separate parts. A clean package install does not prove FEX has a selected, usable RootFS.

The useful failure is:

```text
Invalid or Unsupported elf file.
Current RootFS path set to ''
RootFS path doesn't exist. This is required on AArch64 hosts
```

That usually means the RootFS is missing or no RootFS is selected. If a `.sqsh` file already exists, do not download it again until configuration and integrity have been checked.

## Inspect before changing anything

```bash
uname -m
. /etc/os-release
printf '%s\n' "$PRETTY_NAME"

dpkg -l | grep -E 'fex-emu|squashfs'
command -v FEX FEXBash FEXConfig FEXRootFSFetcher

find \
  "$HOME/.local/share/fex-emu" \
  "$HOME/.config/fex-emu" \
  "$HOME/.fex-emu" \
  -maxdepth 3 -print 2>/dev/null
```

Current packages generally use:

- RootFS data: `~/.local/share/fex-emu/RootFS/`
- Configuration: `~/.config/fex-emu/Config.json`

Older installs may use `~/.fex-emu/RootFS/` and `~/.fex-emu/Config.json`. Follow the directories the installed build actually created.

## Pick the right RootFS form

FEX can use a SquashFS image directly. That image is read-only, which is fine for running binaries already in the image.

Extract the image when packages or applications must be installed into the guest filesystem. A present `.sqsh` file is not a writable x86-64 environment.

The fetcher can download and extract in one pass:

```bash
FEXRootFSFetcher \
  --assume-yes \
  --extract \
  --distro-name=Ubuntu \
  --distro-version=24.04 \
  --distro-list-first
```

Use the `--option=value` form for distro selection. Verify the result instead of assuming the non-interactive options selected the intended entry.

If the archive is already present, extract it directly:

```bash
mkdir -p "$HOME/.local/share/fex-emu/RootFS/Ubuntu_24_04"
unsquashfs -f \
  -d "$HOME/.local/share/fex-emu/RootFS/Ubuntu_24_04" \
  "$HOME/.local/share/fex-emu/RootFS/Ubuntu_24_04.sqsh"
```

`unsquashfs` is provided by `squashfs-tools`.

## Test selection before making it permanent

Point one invocation at the RootFS explicitly:

```bash
FEX_ROOTFS="$HOME/.local/share/fex-emu/RootFS/Ubuntu_24_04" \
  FEX /usr/bin/uname -m
```

The expected guest architecture is:

```text
x86_64
```

If direct selection works, make it persistent with `FEXConfig` or the installed build's `Config.json`. Preserve existing keys. A new minimal current-style config is:

```json
{
  "Config": {
    "RootFS": "Ubuntu_24_04"
  }
}
```

The named value is resolved inside the FEX RootFS data directory. An absolute directory or `.sqsh` path can also be used.

## Install guest packages

Do not try to modify a SquashFS image in place. Use the extracted RootFS and the chroot helper shipped with that RootFS when available:

```bash
cd "$HOME/.local/share/fex-emu/RootFS/Ubuntu_24_04"
./chroot.py chroot
```

Inside the x86-64 chroot:

```bash
apt-get update
apt-get install ./package_amd64.deb
```

Exit cleanly when installation finishes. If the RootFS does not contain `chroot.py`, follow the FEX RootFS documentation for `unbreak_chroot.sh` and cross-architecture chroot setup rather than improvising mounts.

## Verify the application, not just FEX

```bash
FEX /usr/bin/uname -m
FEX /absolute/guest/path/to/application --version
```

For a desktop launcher, wrap the guest executable with `FEX` and use an absolute guest path:

```ini
[Desktop Entry]
Type=Application
Name=x86-64 Application
Exec=FEX /opt/example/application %U
Terminal=false
Categories=Utility;
```

Test the exact `Exec=` command in a terminal before creating the launcher.

## The rule

Check three separate facts: the RootFS exists, FEX has selected it, and the RootFS is writable when guest packages must be installed. Package presence alone proves none of them.

## References

- [FEX RootFS setup](https://wiki.fex-emu.com/index.php/Development:Setting_up_RootFS)
- [FEX configuration](https://wiki.fex-emu.com/index.php/Development:Configuring_FEX)
