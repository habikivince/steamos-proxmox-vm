# SteamOS in a Proxmox VE virtual machine

[🇫🇷 Français](README.fr.md) · 🇬🇧 English

A step-by-step guide to installing SteamOS 3 (the Steam Deck operating system) in a Proxmox VE virtual machine with no dedicated graphics card, ending with a usable KDE Plasma desktop and SSH access.

> **Disclaimer.** Valve does not support running SteamOS in a virtual machine. The method comes from community guides (see [Sources](#sources)), adapted here to Proxmox. It documents an installation actually carried out on 10 November 2026, and may stop working with other versions.

## What you get

- SteamOS boots into **Desktop Mode** (KDE Plasma) in the Proxmox console.
- The VM can be managed over **SSH** from the local network.
- The firewall is restricted to the LAN, and the settings survive system updates.

**Important limitation:** **Gaming Mode does not work.** It requires the Steam Deck's AMD GPU and only shows a black screen in a VM without a graphics card. This VM is for exploring and testing SteamOS, not for playing games.

## Tested environment

| Item | Version |
| --- | --- |
| Hypervisor | Proxmox VE 9.2 |
| Host | Intel NUC, Core i7-1165G7 |
| Recovery image | `steamdeck-oobe-repair-20260707.10-3.8.14.img.bz2` |
| SteamOS after updating | 3.9.5 |

## Requirements

| Item | Value used | Note |
| --- | --- | --- |
| Image | Steam Deck recovery image (`.img.bz2`) | This is not an ISO. Download it from [Valve's official page](https://store.steampowered.com/steamos/download/?ver=steamdeck&snr=) |
| Firmware | UEFI (OVMF), Secure Boot disabled | SteamOS does not boot with legacy BIOS |
| Machine | q35, `host` CPU, 2 cores | |
| Memory | 8 GB | Updates may fail with less |
| Disk | 64 GB, thin-provisioned | About 15 to 20 GB actually used |
| Guest agent | None (`--agent 0`) | |
| Temporary space | About 8 GB in the host's `/root` | For the decompressed image, deleted at the end |

## Conventions

- Commands **on the Proxmox host** are run as `root`.
- Commands **inside SteamOS** are run as the `deck` user.
- All commands use **bash** syntax, the default shell on both Proxmox and SteamOS.
- The examples use the following values. Adapt them to your setup:

| Value | Example used |
| --- | --- |
| VM ID | `216` |
| Storage | `local-lvm` (volume group `pve`) |
| Network bridge | `vmbr0` |
| Local network | `192.168.1.0/24` |

With these values, Proxmox names the disks as follows: `vm-216-disk-0` (EFI disk), `vm-216-disk-1` (imported recovery image), `vm-216-disk-2` (SteamOS target disk). Check the actual names with `qm config 216`.

## Installation

### 1. Download and decompress the image

On the host, in `/root` (not in the ISO folder). Copy the download link from Valve's page:

```bash
cd /root
wget '<Valve download link>'
bunzip2 -k steamdeck-oobe-repair-20260707.10-3.8.14.img.bz2
```

### 2. Create the VM and import the image

```bash
qm create 216 --name steamos --ostype l26 --machine q35 --bios ovmf \
  --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=0 \
  --sockets 1 --cores 2 --cpu host --memory 8192 \
  --scsihw virtio-scsi-single --net0 virtio,bridge=vmbr0 --agent 0 && \
qm disk import 216 /root/steamdeck-oobe-repair-20260707.10-3.8.14.img local-lvm
```

The `pre-enrolled-keys=0` option disables Secure Boot. The imported image shows up as the `unused0` disk (here `vm-216-disk-1`).

### 3. Attach the image and expose the target disk as NVMe

Valve's install script looks for the disk `/dev/nvme0n1`. Proxmox does not offer NVMe disks in its interface, so the disk is declared temporarily through the `args` option, which passes raw arguments to QEMU.

```bash
qm set 216 --virtio1 local-lvm:vm-216-disk-1 --boot order=virtio1 && \
pvesm alloc local-lvm 216 vm-216-disk-2 64G && \
lvchange -ay pve/vm-216-disk-2 && \
qm set 216 --args '-drive file=/dev/pve/vm-216-disk-2,if=none,id=nvm,format=raw,cache=none -device nvme,drive=nvm,serial=badbeef' && \
qm start 216
```

The VM boots from the recovery image. Open its console in the Proxmox web interface.

### 4. Install SteamOS

On the recovery desktop:

1. If you do not use a US QWERTY keyboard, add your layout first (System Settings > Keyboard). You will need it to type a password in the next step.
2. Run **"Wipe Devices & Install SteamOS"**, then **"Proceed"**.
3. When it finishes, answer **"Cancel"** to the reboot prompt.

### 5. Enable SSH before the first boot

Without this step, the VM boots into Gaming Mode on a black screen, with no way to get in.

SteamOS has two sets of system partitions, A and B, each with its own configuration. SSH must be enabled in both.

In the recovery desktop's terminal, open a chroot on **set B** with Valve's script:

```bash
sudo ~/tools/repair_device.sh chroot
```

Then, inside the chroot, set a password for `deck` and enable SSH:

```bash
steamos-readonly disable
passwd deck
systemctl enable sshd
steamos-readonly enable
exit
```

Do the same on **set A**:

```bash
sudo steamos-chroot --disk /dev/nvme0n1 --partset A --no-overlay
```

```bash
steamos-readonly disable
passwd deck
systemctl enable sshd
steamos-readonly enable
exit
```

Finally, power off the VM:

```bash
sudo systemctl poweroff
```

### 6. Remove the NVMe device and reattach the disk as virtio

On the host, once the VM is off. The first command aborts the rest if the VM is still running:

```bash
qm status 216 | grep -q stopped && \
qm set 216 --delete args && \
qm set 216 --virtio0 local-lvm:vm-216-disk-2,discard=on,iothread=1 --boot order=virtio0 && \
qm set 216 --delete virtio1 && \
qm start 216
```

**The console stays black: this is expected** (Gaming Mode). You now need the VM's IP address. Get its MAC address with `qm config 216 | grep net0`, then sweep the network from the host:

```bash
(for i in $(seq 2 254); do ping -c1 -W1 192.168.1.$i >/dev/null & done; wait)
ip neigh | grep -i <VM MAC address>
```

Then log in:

```bash
ssh deck@<ip>
```

### 7. Switch to the desktop permanently

Over SSH:

```bash
steamosctl set-default-login-mode desktop && \
steamosctl set-default-desktop-session plasma.desktop
sudo reboot
```

After the reboot, the KDE Plasma desktop appears in the Proxmox console.

> **Do not click the "Return to Gaming Mode" icon on the desktop**: it makes Gaming Mode the default again. If that happens, repeat this step over SSH.

### 8. Restrict the firewall to the local network

By default, SteamOS opens SSH and ports 1024 to 65535 to everyone. The commands below create a `lan` zone limited to your network, allow SSH in it, then close the public zone. Adapt `192.168.1.0/24`.

```bash
sudo firewall-cmd --permanent --new-zone=lan && \
sudo firewall-cmd --permanent --zone=lan --add-source=192.168.1.0/24 && \
sudo firewall-cmd --permanent --zone=lan --add-service=ssh && \
sudo firewall-cmd --permanent --zone=public --remove-service=ssh && \
sudo firewall-cmd --permanent --zone=public --remove-port=1024-65535/tcp --remove-port=1024-65535/udp && \
sudo firewall-cmd --reload
```

Removing the service and removing the ports must stay two separate commands: this firewalld version refuses to combine them.

### 9. Set the keyboard layout and time zone

Example for a French keyboard and Paris time. Adapt to your own:

```bash
sudo localectl set-x11-keymap fr pc105 oss_nodeadkeys && \
sudo timedatectl set-timezone Europe/Paris
```

### 10. Make these settings survive updates

**Do this before the first update.** SteamOS rewrites `/etc` on every update and keeps only the files named in a keep list. The `/etc/atomic-update.conf.d/` folder itself is kept, so that is where your own list goes.

```bash
printf '%s\n' /etc/firewalld/zones/lan.xml /etc/firewalld/zones/public.xml /etc/vconsole.conf /etc/X11/xorg.conf.d/00-keyboard.conf /etc/localtime | sudo tee /etc/atomic-update.conf.d/homelab-keep.conf
```

The system ships an example of this format in `/etc/atomic-update.conf.d/example-additional-keep-list.conf`.

> In the original installation, this list was created after the first update: the `lan` zone and the keyboard layout were lost, and steps 8 and 9 had to be redone.

### 11. Update SteamOS

The update is atomic: it installs to the other partition set (A/B), and the system then reboots into it.

```bash
sudo steamos-update check && sudo steamos-update && sudo reboot
```

### 12. Clean up and take a snapshot

On the host. A clean shutdown did not work during this installation (see [Troubleshooting](#troubleshooting)), so `qm stop` was needed:

```bash
qm stop 216 && \
qm disk unlink 216 --idlist unused0 --force && \
qm snapshot 216 steamos-propre
```

`qm disk unlink ... --force` permanently deletes the recovery image, which became `unused0` in step 6. Finally, delete the downloaded files:

```bash
rm -v /root/steamdeck-oobe-repair-20260707.10-3.8.14.img /root/steamdeck-oobe-repair-20260707.10-3.8.14.img.bz2
```

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Black screen after installation | The `gamescope` session (Gaming Mode) is started by SDDM | Enable SSH in the chroot (step 5), then switch with `steamosctl` (step 7) |
| `switch-to-desktop-mode` fails with "TransactionIsDestructive" | The `galileo-mura-setup` user service (Deck OLED screen calibration) had a pending start job | The default mode was already saved: a reboot is enough |
| `firewall-cmd` rejects `--remove-service` and `--remove-port` together | Limitation of this firewalld version | Two separate commands (step 8) |
| `lan` zone and keyboard layout lost after `steamos-update` | `/etc` is rewritten on every update, except for listed files | Create the keep list (step 10), then redo steps 8 and 9 |
| `qm shutdown` times out, VM still running | Cause not identified: ACPI is ignored, and `sudo systemctl poweroff` over SSH hung after the network went down | `qm stop` as a last resort. On the next boot, read `journalctl -b -1 -n 50` |
| Back in Gaming Mode after clicking "Return to Gaming Mode" | The icon makes Gaming Mode the default again | Repeat step 7 over SSH |

## Not yet verified

- **Keep list.** It still has to be confirmed, at a future update, that it really preserves the firewall, keyboard and time zone settings.
- **VM shutdown.** The reason `qm shutdown` and `systemctl poweroff` fail has not been found.

## Sources

- [Igalia: Running the Steam Deck's OS in a virtual machine using QEMU](https://blogs.igalia.com/berto/2022/07/05/running-the-steam-decks-os-in-a-virtual-machine-using-qemu/) (updated in July 2025 for the 3.7.7 image)
- [MusaBase: How to virtualize SteamOS](https://www.musabase.com/2025/04/how-to-virtualize-steamos-test-its.html)
- [GamingOnLinux: HOWTO SteamOS 3 on a VM](https://www.gamingonlinux.com/forum/topic/5200/)
- [Reddit r/SteamOS: installing on RX 9060 XT / 9070 XT](https://www.reddit.com/r/SteamOS/comments/1o2bk9b/how_to_install_steamos_on_amd_rx_9070xt_9060xt/) (source of the `repair_device.sh chroot` tip)
- [Valve's SteamOS download page](https://store.steampowered.com/steamos/download/?ver=steamdeck&snr=)
- On the system itself: `/usr/bin/steamos-session-select` and `/etc/atomic-update.conf.d/example-additional-keep-list.conf`
