# windows-vm

## What this project is

This is a small, standalone Docker Compose definition for an occasional-use Windows 11 Pro VM. It runs the pinned `dockurr/windows:6.05` container, which runs QEMU/KVM and automates the Windows download and installation. It does not configure the host OS, libvirt, host QEMU, GPU passthrough, Secure Boot, or TPM.

## Architecture

```text
Linux host
├── Docker / Docker Compose
├── /dev/kvm
├── FreeRDP (`xfreerdp3` or `xfreerdp`)
├── PC/SC service and USB CCID driver
├── `pcsc_scan` diagnostic tool
│
└── windows-vm/
    └── dockurr/windows Docker container
        └── QEMU/KVM
            └── Windows 11 Pro
```

The smart-card reader remains attached to the Linux host. FreeRDP redirects its
PC/SC interface into the Windows RDP session; the Docker container does not
receive raw access to the host USB bus.

Docker commands intentionally use `sudo`; membership in the `docker` group is not assumed. The web console and RDP ports bind only to `127.0.0.1`, and `PROTECT=Y` protects the web interface.

## Host prerequisites

The host must provide:

- Docker and Docker Compose
- working `/dev/kvm` and `/dev/net/tun`
- a PC/SC service and USB CCID driver
- FreeRDP (`xfreerdp3` or `xfreerdp`) built with PC/SC support
- `pcsc_scan` for host-side reader diagnostics

Install and configure these on the host separately. This project does not install host packages.

### NixOS host configuration

For NixOS, enable the system PC/SC service and install the diagnostic tool:

```nix
{ pkgs, ... }:

{
  services.pcscd.enable = true;

  environment.systemPackages = [
    pkgs.pcsc-tools
  ];
}
```

The NixOS `pcscd` module includes the generic `ccid` plugin and its udev rules.
This supports USB CCID readers such as the HID Global OMNIKEY 5422 without
configuring smart-card login or changing PAM authentication.

After rebuilding the host, stop the VM, reconnect the reader, and verify that
the host owns it:

```bash
pcsc_scan
```

Both the contact and contactless OMNIKEY interfaces should be listed.

### Ubuntu host setup

Install the host prerequisites, including Docker Compose v2, the X11 FreeRDP
client, and the PC/SC service and CCID driver:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 freerdp3-x11 pcscd libccid pcsc-tools
sudo systemctl enable --now pcscd.socket
```

Verify Docker Compose v2 and the required KVM and TUN devices:

```bash
docker compose version
test -e /dev/kvm && echo '/dev/kvm is available'
test -e /dev/net/tun && echo '/dev/net/tun is available'
```

If either device check fails, enable KVM virtualization in the host firmware as
needed and ensure the appropriate host kernel support is present before using
this project.

With the smart-card reader connected, verify that PC/SC can see it:

```bash
pcsc_scan
```

Press `Ctrl-C` after confirming the reader is listed. Then verify the FreeRDP
build and its PC/SC support:

```bash
xfreerdp3 /buildconfig
xfreerdp3 /buildconfig | grep -Ei '(^|[[:space:]])WITH_PCSC[[:space:]]*=[[:space:]]*(ON|TRUE|YES|Y|1)([[:space:]]|$)'
```

The second command must print a `WITH_PCSC` setting. `scripts/connect` accepts
the standard enabled CMake values `ON`, `TRUE`, `YES`, `Y`, and `1`, without
regard to case.

## Configuration

Create the private environment file:

```bash
cp .env.example .env
chmod 600 .env
```

Replace every placeholder in `.env`. The file contains the Windows username and password. It is ignored by Git and must remain mode `0600`.

The VM is explicitly configured for Windows 11 Pro with 8 CPU cores, 12 GiB RAM, and a 128 GiB raw persistent disk. Stock Dockur defaults provide the host CPU model, KVM and Hyper-V enlightenments, q35 machine, SCSI disk, and the intended non-Secure-Boot/non-TPM configuration.

## First Windows installation

Use this workflow:

```bash
cp .env.example .env
chmod 600 .env

./scripts/start
```

Open <http://127.0.0.1:8006> and observe the automated Windows download and installation. Dockur performs both automatically. Once Windows has reached the desktop, connect with:

```bash
./scripts/connect
```

Use RDP for normal daily use. Port 8006 is primarily for initial installation and recovery or troubleshooting.

## Connecting with FreeRDP

`./scripts/connect` launches FreeRDP (`xfreerdp3` when available, otherwise `xfreerdp`) against `127.0.0.1:3389` with smart-card redirection, sound, microphone, clipboard, dynamic resolution, automatic reconnect, and trust-on-first-use certificate handling. It passes only the username; FreeRDP prompts interactively for the Windows password so the password does not appear in the process list.

`scripts/connect` uses the X11 FreeRDP client, so it needs an X11 graphical
environment or XWayland session on the host.

The container must already be running. If Windows is still booting, connection can fail; retry shortly. Closing or failing the RDP client never stops the VM.

## Smart-card reader workflow

The reader uses FreeRDP PC/SC redirection:

```text
OMNIKEY reader
→ Linux PC/SC service
→ FreeRDP `/smartcard` channel
→ Windows RDP session
→ Windows application
```

Connect the reader before running `./scripts/connect`. The reader is available
inside the RDP session only while that session is connected. It is intentionally
not passed to QEMU, because raw USB passthrough and host PC/SC access would
compete for exclusive control of the same physical device.

Use the web console for installation and recovery. Applications launched there
do not receive the FreeRDP-redirected reader.

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

## Moving to another Linux host

Stop the VM cleanly, then move the same project folder—including the ignored `local/storage`, `local/shared`, and private `.env`—to the future Linux host. No project files need conversion. The new host needs Docker, Docker Compose, `/dev/kvm`, `/dev/net/tun`, FreeRDP with PC/SC support, and the applicable PC/SC/CCID host configuration described above. Keep `.env` mode `0600` after copying.

This repository intentionally contains no distro-specific host configuration. Host enablement remains a separate concern.
