# CyberMyth Linux on AMD64

Kernel: `Linux 6.18.49-cybermyth-amd64` \
Build Repository: [CyberMyth-OS/CyberMyth-Linux](https://github.com/CyberMyth-OS/CyberMyth-Linux/tree/main/) \
Pre-built Images: [Image Archive](https://images.cybermyth.dev/amd64) \
CyberMyth Version: `1.0` \
Live Boot Creds: `cybermyth:cybermyth` \

The AMD64 image is the general-purpose desktop and laptop release: a UEFI live
ISO with the GNOME desktop and the Calamares installer.

**This is a base image, not a loaded toolkit.** It ships the CyberMyth kernel,
branding, and the `anonymity-cli` tool, and it wires up the `cybermyth-os` APT
repository — but it does **not** preinstall a penetration-testing suite. Install
the tools you want with `apt` once the system is up. If you are expecting a
Kali-style image with hundreds of tools preloaded, this is deliberately not
that.

**Current status**:
| Component | Status |
|----------|------|
|Live boot (GNOME session)|Working|
|Calamares installer|Working|
|Wi-Fi (incl. Intel Wi-Fi 7)|Working|
|Bluetooth|Working|
|Wired networking|Working|
|Graphics (Intel / AMD / NVIDIA nouveau)|Working|
|Audio (PipeWire + SOF)|Working|
|Anonymity CLI (Tor routing, DNS over HTTPS)|Working|
|Tor Browser|Working|
|Night light|Known issue — toggles but does not activate|
| Installer | Calamares |

> Secure Boot must be **disabled**. The kernel is signed by a build-generated
> key that is in no firmware database. shim + MOK enrolment is planned.

### Pre-requisites

Download the `.iso` from the
[Image Archive](https://images.cybermyth.dev/amd64/cybermyth-1.0/).

Verify it against the published checksum before writing it — this is an
anonymity distribution, and an unverified image is worth nothing:

```bash
sha256sum -c cybermyth-1.0-amd64.iso.sha256
```

Write it to a USB stick (replace `/dev/sdX` with your device — this erases it):

```bash
sudo dd if=cybermyth-1.0-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

The ISO is a hybrid image: it can be written directly to USB or burned to
optical media.

### Booting

Boot the USB stick with **UEFI x64** and **Secure Boot disabled**.

The GRUB menu offers:

| Entry | Use |
|---|---|
| `CyberMyth OS 1.0 (Live)` | normal live session, default |
| `Install CyberMyth OS` | live session with Calamares started automatically |
| `CyberMyth OS 1.0 (safe graphics)` | `nomodeset`, for machines where KMS fails |

The live user is `cybermyth` with password `cybermyth`, created at boot by
`live-config`. It has passwordless `sudo`. Both the username and the hostname
can be overridden from the GRUB command line, e.g. `username=analyst hostname=box`.

### Installing

Launch **Install CyberMyth OS** from the desktop, the applications grid, or the
GRUB `Install` entry. Calamares handles partitioning, locale, keyboard, user
creation and the bootloader.

The installed system keeps the `cybermyth-os` APT repository, so
`apt upgrade` pulls CyberMyth package updates alongside Debian's.

## Feature Reference

### Platform

| Item | Value |
|---|---|
| Base | Debian 13 trixie, amd64 |
| Kernel | `6.18.49-cybermyth-amd64` |
| Desktop | GNOME 48.7 |
| Display manager | GDM 48.0 |
| Installer | Calamares 3.3.14 |
| Init | systemd |
| Bootloader | GRUB, `/EFI/boot/bootx64.efi` |
| Firmware | linux-firmware 20260810 |
| Packages installed | ~1710 |

### Kernel

Built from upstream `linux-6.18.49`, configured from Debian 13's own amd64
config so that hardware support matches Debian, then extended.

**Hardening** (beyond Debian's defaults):

| Option | Effect |
|---|---|
| `INIT_ON_FREE_DEFAULT_ON` | zero memory on free, defeats use-after-free data reuse |
| `RANDOM_KMALLOC_CACHES` | per-callsite kmalloc cache split |
| `ZERO_CALL_USED_REGS` | scrub call-used registers on return |
| `HARDENED_USERCOPY`, `FORTIFY_SOURCE` | bounds-checked copies and string ops |
| `RANDOMIZE_BASE`, `RANDOMIZE_MEMORY` | KASLR |
| `LEGACY_VSYSCALL_NONE` | removes the fixed-address vsyscall page |
| `SECURITY_DMESG_RESTRICT` | dmesg is root-only |
| `LSM=landlock,yama,integrity,apparmor,bpf` | AppArmor + Landlock, **no lockdown** |

`lockdown` is deliberately absent from the LSM list: it restricts `/dev/mem`,
raw MSR access and unsigned module loading, which offensive-security tooling
needs. It is still compiled in, so `lsm=` on the kernel command line can
re-enable it.

Module signing is on (`MODULE_SIG`), but **not** enforced — `MODULE_SIG_FORCE`
is off and `module.sig_enforce` is not set, so DKMS modules (out-of-tree
Realtek drivers, VirtualBox, NVIDIA) still load.

`CONFIG_DEBUG_INFO_BTF` is enabled, so `bpftrace`, `bcc` and eBPF CO-RE tooling
work out of the box.

### Wireless

Debian 13 ships a 6.12 kernel, so its config predates several USB Wi-Fi
drivers. These are enabled additionally:

| Driver | Hardware |
|---|---|
| `rtw88_8812au` | RTL8812AU — Alfa AWUS036ACH |
| `rtw88_8814au`, `rtw88_8814ae` | RTL8814AU — Alfa AWUS1900 |
| `rtw88_8821au` | RTL8821AU / RTL8811AU |
| `rtw89_8851bu`, `rtw89_8852bu` | Realtek Wi-Fi 6 USB |
| `iwlmld` | Intel Wi-Fi 7 (BE200 / BE201) |

`firmware-ath9k-htc` and `firmware-carl9170` are installed, covering the
AR9271 / AR9170 monitor-mode and injection adapters.

> **Firmware note.** The 6.18 kernel requires `iwlwifi` firmware API ≥ 100 for
> Intel Wi-Fi 7, and Debian 13 ships only up to API 97 — on that firmware the
> card never initialises and there is no Wi-Fi at all (Bluetooth still works,
> which makes it easy to misdiagnose). CyberMyth ships linux-firmware 20260810,
> which provides API 100/101, from the `cybermyth-os` repository.

### Anonymity

`anonymity-cli` 1.0.0 has exactly two components. Run it as root:

```
Usage: anonymity <component> [options...]

Components:
  tor   Route all system traffic through Tor.
  dns   Configure DNS over HTTPS for the whole system.
```

| Item | Value |
|---|---|
| Tool | `anonymity-cli` 1.0.0 |
| Tor | 0.4.9.11 |
| Tor Browser | 15.0.21 |
| Transparent routing | nftables, via `anonymity-cli tor` |
| DNS over HTTPS | dnscrypt-proxy, via `anonymity-cli dns` |

That is the whole of it. There is no VPN component, no profile manager and no
GUI — transparent Tor routing for the whole system, and system-wide DoH.

> As noted on the [devices page](../devices.md): CyberMyth provides the software
> and brings the tools together, but how anonymous you are depends on your own
> practices. `anonymity-cli` routes traffic; it does not defeat browser
> fingerprinting or telemetry. Use Tor Browser if you need that.

### Included tooling

The image is intentionally close to a stock Debian desktop. The only
security-adjacent tools preinstalled are:

| Tool | Version |
|---|---|
| `nmap` | 7.95 |
| `tcpdump` | 4.99.5 |
| `macchanger` | 1.7.0 |

`cybermyth-core` additionally pulls the plumbing `anonymity-cli` needs (nftables,
Tor, torsocks, proxychains4, dnscrypt-proxy, OpenVPN, ufw, zram-tools) plus the
usual scripting and development basics — Python 3, git, tmux, jq, ripgrep,
`fd-find`, sqlite3, rsync and friends. `cybermyth-desktop` adds the GNOME
session, Tilix, Firefox ESR, Tor Browser, GParted and the CyberMyth theme.

**Everything else you install yourself:**

```bash
sudo apt update
sudo apt install aircrack-ng wireshark hydra sqlmap john hashcat radare2
```

Debian's archive covers most of the common tooling. The `cybermyth-os`
repository is enabled by default, so anything added to the CyberMyth
metapackages later arrives on `apt upgrade`.

## Known issues

- **Night light** toggles in GNOME Settings but does not visibly activate.
- **Secure Boot** is not supported; it must be disabled.
- **Legacy BIOS boot** is not available in 1.0 — the ISO is UEFI x64 only.

## Support

- Report bugs: [GitHub Issues](https://github.com/CyberMyth-OS/CyberMyth-Linux/issues)
- Email: <cybermyth@mystichackers.com>
