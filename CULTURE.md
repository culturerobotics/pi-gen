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

On macOS, or anywhere with Docker:

```bash
WIFI_SSID="Culture" WIFI_PASSWORD="yourpassword" ./build-docker.sh
```

Natively on Linux:

```bash
sudo ./build.sh
```

`./build.sh` from a terminal prompts for the SSID and password if you don't set
them, and offers to add further networks. `./build-docker.sh` runs the container
without a tty, so it cannot prompt — pass the variables as above and it forwards
them in. See [Configuring WiFi](#configuring-wifi) for multiple access points
and for supplying a pre-hashed PSK.

The password is hashed via `wpa_passphrase` and never stored in plain text; the
committed `network-config` holds only placeholders.

The output image lands in `deploy/` as a `.zip`, which Raspberry Pi Imager can
flash directly without unpacking.

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

## Configuring WiFi

WiFi credentials are supplied at build time via environment variables and
substituted into `stage2/04-cloud-init/files/network-config`. Nothing secret is
committed — the file in git only holds placeholders.

```bash
WIFI_SSID=Culture WIFI_PASSWORD=... ./build-docker.sh
```

For more than one access point, add numbered variables. The device will
associate with whichever is in range:

```bash
WIFI_SSID=Culture   WIFI_PASSWORD=... \
WIFI_SSID_2=culture WIFI_PASSWORD_2=... \
./build-docker.sh
```

`build-docker.sh` forwards every `WIFI_*` variable into the container using
`-e NAME`, so the values are passed from your environment rather than appearing
in the container's process arguments.

To avoid putting a passphrase in your shell at all, supply a pre-hashed PSK with
`WIFI_PSK` / `WIFI_PSK_n` instead of `WIFI_PASSWORD`:

```bash
WIFI_PSK=$(wpa_passphrase MySSID | grep -v '#' | grep psk= | cut -d= -f2)
```

Running `./build.sh` directly from a terminal prompts interactively instead, and
offers to add further networks. The container has no tty, so `./build-docker.sh`
requires the environment variables.

To change networks after a build, restore the placeholders in `network-config`:

```yaml
access-points:
  WIFI_SSID_PLACEHOLDER:
    password: WIFI_HASH_PLACEHOLDER
```

Note the PSK is written as a bare 64-character hex string. A `hash:` prefix is
**not** valid netplan syntax — it is passed through to NetworkManager verbatim,
producing a value that is neither valid hex nor a legal passphrase, and the
interface then fails to associate with `reason 'no-secrets'`.

## Iterating on a Live Pi

Rather than rebuilding and reflashing for every change, you can edit the
cloud-init inputs on a running Pi and re-run the first-boot sequence. The files
on the boot partition map 1:1 to the repo:

| Repo | On the Pi |
|---|---|
| `stage2/04-cloud-init/files/user-data` | `/boot/firmware/user-data` |
| `stage2/04-cloud-init/files/network-config` | `/boot/firmware/network-config` |
| `stage2/04-cloud-init/files/meta-data` | `/boot/firmware/meta-data` |

Edit those, then reset:

```bash
sudo cloud-init clean --logs
sudo rm -f /etc/netplan/90-NM-*.yaml
sudo reboot
```

Expect **two** reboots — `runcmd` ends with `[reboot]`, so the first boot
reboots itself partway through. That is normal, not a hang.

Three things will waste your time if you don't know them:

**Editing `/boot/firmware/*` does nothing on its own.** cloud-init reads those
files only on first boot. Without wiping `/var/lib/cloud` the edit looks applied
but isn't — this is why `network-config` changes appear to be ignored.

**`cloud-init clean` does not touch `/etc/netplan/`.** The generated
`90-NM-*.yaml` files survive and keep their old NetworkManager connection UUIDs
and names, so a re-render can pair a new SSID with a stale profile identity.
Delete them as part of the reset, as above.

**cloud-init only ever writes files, never removes them.** Deleting a
`write_files` entry or a `runcmd` leaves the file and any enabled unit in place
on a device you have been iterating on. Remove those by hand:

```bash
sudo systemctl disable --now <unit>
sudo rm -f /etc/systemd/system/<unit> /usr/local/sbin/<script>
sudo systemctl daemon-reload
```

Useful checks after a reset:

```bash
cloud-init status --long
sudo journalctl -t display-ip-info -b       # the boot banner
nmcli -t -f DEVICE,STATE,CONNECTION device  # did wifi associate?
systemctl list-units --state=failed
```

Note `cloud-init status` reports `degraded done` on this image regardless of
success: `/etc/cloud/cloud.cfg` lists a `netplan_nm_patch` module that the
installed `rpi-cloud-init-mods` does not ship. Judge a boot by the checks above,
not by that word.

When you are done, fold the working changes back into the repo files and rebuild
— the image should always be reproducible from git, never captured from a card
you have been iterating on.

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
```
