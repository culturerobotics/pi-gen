# Culture Pi Image Build

Custom Raspberry Pi OS Lite (64-bit, Trixie) image built with pi-gen.

## Prerequisites

Build machine must be Debian-based with the following installed:

```bash
apt install coreutils quilt parted qemu-user-static debootstrap zerofree zip \
  dosfstools e2fsprogs libarchive-tools libcap2-bin grep rsync xz-utils file git curl bc \
  gpg pigz xxd arch-test bmap-tools kmod wpasupplicant
```

`wpasupplicant` is required for WiFi password hashing during the build.

## Quick Start

```bash
sudo ./build.sh
```

The build will prompt for your WiFi SSID and password. The password is hashed
via `wpa_passphrase` and never stored in plain text.

To skip the prompt, pass them as environment variables:

```bash
sudo WIFI_SSID="Culture" WIFI_PASSWORD="yourpassword" ./build.sh
```

The output image lands in `deploy/`.

## What the Image Includes

- **User**: `culture` with passwordless sudo and SSH enabled
- **SSH keys**: pulled from GitHub (`jmeed`, `k0s`) at image creation, then
  refreshed from S3 every 30 minutes via a systemd timer
- **Random hostname**: each Pi gets a unique `pi-XXXXXX` hostname on first boot
- **Boot info display**: hostname and IP are printed to the console and login
  prompt after networking is up
- **Plymouth disabled**: boot splash is disabled to avoid a known hang
- **WiFi + Ethernet**: both configured via cloud-init (netplan), WiFi is
  required for boot, Ethernet is optional

## Config Reference

All options are in the `config` file:

| Variable | Current Value | Purpose |
|---|---|---|
| `IMG_NAME` | `reactoros_pi64` | Output image name |
| `RELEASE` | `trixie` | Debian release |
| `ENABLE_SSH` | `1` | SSH server enabled |
| `STAGE_LIST` | `stage0 stage1 stage2` | Lite image (no desktop) |
| `TIMEZONE_DEFAULT` | `UTC` | System timezone |
| `WPA_COUNTRY` | `US` | WiFi regulatory domain |
| `FIRST_USER_NAME` | `culture` | Default user |
| `FIRST_USER_PASS` | `culture` | Build-time password (SSH key auth is primary) |
| `DISABLE_FIRST_BOOT_USER_RENAME` | `1` | Skips the setup wizard on first boot |

## Rebuilding a Single Stage

After a full build, you can skip earlier stages to iterate faster:

```bash
# Skip stage0 (debootstrap) and rebuild from stage1 onward
touch ./stage0/SKIP
sudo CLEAN=1 ./build.sh

# Skip stage0 and stage1, rebuild only stage2
touch ./stage0/SKIP ./stage1/SKIP
sudo CLEAN=1 ./build.sh
```

Remove the SKIP files when you want a full rebuild:

```bash
rm -f ./stage0/SKIP ./stage1/SKIP
```

## Changing the WiFi Network

The WiFi config lives in `stage2/04-cloud-init/files/network-config`. After a
build, the placeholders are replaced with real values. To change the network,
restore the placeholders:

```yaml
access-points:
  WIFI_SSID_PLACEHOLDER:
    password: hash:WIFI_HASH_PLACEHOLDER
```

Then rebuild and the build will prompt for the new credentials.

## Re-running Cloud-Init on a Live Pi

If you need to re-apply cloud-init without reimaging:

```bash
sudo cloud-init clean --reboot
```

This wipes cloud-init's state and reboots. On the next boot it runs the full
first-boot sequence again.

## Key Files

| File | Purpose |
|---|---|
| `config` | Build-time variables |
| `stage1/00-boot-files/files/cmdline.txt` | Kernel command line (plymouth disabled here) |
| `stage2/04-cloud-init/files/user-data` | Cloud-init: user, SSH keys, systemd services |
| `stage2/04-cloud-init/files/network-config` | Cloud-init: WiFi and Ethernet (netplan v2) |
| `stage2/04-cloud-init/files/meta-data` | Cloud-init: instance metadata |

## Debugging a Booted Pi

```bash
# Check cloud-init status
cloud-init status --long

# View cloud-init log
cat /var/log/cloud-init-output.log

# Check WiFi
ip addr show wlan0
journalctl -u netplan-* --no-pager

# Check SSH key update
journalctl -u ssh-key-update.service --no-pager
cat /var/log/ssh_key_update.log

# Check hostname service
journalctl -u set-random-hostname.service --no-pager
```
