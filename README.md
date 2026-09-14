# windows-vm

## What this project is

This is a small, standalone Docker Compose definition for an occasional-use Windows 11 Pro VM. It runs the pinned `dockurr/windows:6.05` container, which runs QEMU/KVM and automates the Windows download and installation. It does not configure Nix, libvirt, host QEMU, GPU passthrough, Secure Boot, or TPM.

## Architecture

```text
Linux host
├── Docker / Docker Compose
├── /dev/kvm
├── FreeRDP (`xfreerdp3` or `xfreerdp`)
├── usbutils (lsusb)
│
└── windows-vm/
    └── dockurr/windows Docker container
        └── QEMU/KVM
            └── Windows 11 Pro
```

Docker commands intentionally use `sudo`; membership in the `docker` group is not assumed. The web console and RDP ports bind only to `127.0.0.1`, and `PROTECT=Y` protects the web interface.

## Host prerequisites

The host must provide:

- Docker and Docker Compose
- working `/dev/kvm` and `/dev/net/tun`
- access to `/dev/bus/usb`
- FreeRDP (`xfreerdp3` or `xfreerdp`)
- usbutils (`lsusb`)

Install and configure these on the host separately. This project does not install host packages.

## Configuration

Create the private environment file:

```bash
cp .env.example .env
chmod 600 .env
```

Replace every placeholder in `.env`. The file contains the Windows username and password plus the reader VID/PID. It is ignored by Git and must remain mode `0600`.

The VM is explicitly configured for Windows 11 Pro with 8 CPU cores, 12 GiB RAM, and a 128 GiB raw persistent disk. Stock Dockur defaults provide the host CPU model, KVM and Hyper-V enlightenments, q35 machine, SCSI disk, and the intended non-Secure-Boot/non-TPM configuration.

## Finding USB VID/PID

Connect the smart-card reader temporarily and run:

```bash
lsusb
```

For example:

```text
Bus 001 Device 005: ID 076b:1234 HID Global ...
```

becomes:

```text
USB_VENDOR_ID=0x076b
USB_PRODUCT_ID=0x1234
```

Keep the `0x` prefixes. Only the configured VID/PID is matched by QEMU; this project does not pass through arbitrary storage devices.

## First Windows installation

Use this workflow:

```bash
cp .env.example .env
chmod 600 .env

lsusb
# edit .env

./scripts/start
```

Open <http://127.0.0.1:8006> and observe the automated Windows download and installation. Dockur performs both automatically. Once Windows has reached the desktop, connect with:

```bash
./scripts/connect
```

Use RDP for normal daily use. Port 8006 is primarily for initial installation and recovery or troubleshooting.

## Connecting with FreeRDP

`./scripts/connect` launches FreeRDP (`xfreerdp3` when available, otherwise `xfreerdp`) against `127.0.0.1:3389` with sound, microphone, clipboard, dynamic resolution, automatic reconnect, and trust-on-first-use certificate handling. It passes only the username; FreeRDP prompts interactively for the Windows password so the password does not appear in the process list.

The container must already be running. If Windows is still booting, connection can fail; retry shortly. Closing or failing the RDP client never stops the VM.

## Smart-card reader hotplug workflow

The reader uses raw QEMU USB passthrough, not SPICE redirection:

```text
start VM
→ Windows boots
→ physically plug reader into laptop
→ QEMU matches VID/PID
→ Windows detects the actual USB reader
```

If the reader was connected earlier, unplug and reconnect it after Windows boots. The host exposes `/dev/bus/usb` to the container and the configured QEMU `usb-host` device matches only its VID/PID.

## Shared `Z:` folder

Only `local/shared` is shared with the VM. Dockur exposes it inside Windows as drive `Z:` and as the `Shared` folder on the Windows desktop. No home directory, Documents directory, or other broad host path is exposed.

## Commands

```bash
./scripts/start    # validate prerequisites and start the VM
./scripts/connect  # open an independent RDP session
./scripts/stop     # gracefully stop and remove the container
./scripts/status   # show container state and local endpoints/paths
./scripts/logs     # follow Dockur container logs
./scripts/update   # show the intentional pin workflow, then pull that pin
```

`start` does not wait for Windows and does not open RDP. `stop` uses `docker compose down` with a two-minute grace period; it does not kill QEMU or delete storage. Container state does not prove that Windows itself is ready.

## Storage and backup implications

The layers have distinct roles:

```text
tracked Git repository = VM definition
local/storage          = actual Windows installation/data
Docker image           = Dockur/QEMU runtime
```

`local/storage` and `local/shared` are ignored runtime state. The Windows installation survives `sudo docker compose down`, container recreation, and image updates because `./local/storage` is bind-mounted at `/storage`. Deleting the container does not delete it. **Deleting `local/storage` destroys the Windows installation and its data.** Back it up separately while the VM is stopped.

## Explicit Dockur update workflow

The image is intentionally pinned to `dockurr/windows:6.05`. There is no automatic version discovery. To upgrade:

1. Intentionally edit the image tag in `compose.yml`.
2. Review that tracked change and its release implications.
3. Run `./scripts/update` to pull the selected tag.
4. Start or recreate the container when desired.

The update script never modifies `local/storage`. Host OS updates remain independent of this container image.

## Windows activation note

Activation is a separate manual post-install task inside Windows. No product key is stored in `.env`, Compose, Git, or the scripts.

## Moving from Arch/Wintarch to NixOS/Wintix

Stop the VM cleanly, then move the same project folder—including the ignored `local/storage`, `local/shared`, and private `.env`—to the future host. No project files need conversion. The Wintix host only needs Docker, Docker Compose, `/dev/kvm`, `/dev/net/tun`, FreeRDP, usbutils, and suitable USB device access. Keep `.env` mode `0600` after copying.

This repository intentionally contains no Nix or Wintix configuration. Host enablement remains a separate concern.
