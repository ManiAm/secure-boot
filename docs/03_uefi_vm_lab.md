# Lab: Build a UEFI Secure Boot Test VM

> **Prerequisite:** Familiarity with the Secure Boot key hierarchy and verification flow from [UEFI Secure Boot](01_secure_boot.md).

This lab sets up a QEMU virtual machine with UEFI firmware so you can safely experiment with Secure Boot without touching your daily machine. By the end, you will have a VM with your own custom Secure Boot keys enrolled, a signed boot chain, and Secure Boot actively enforcing signature verification.

## Why a VM?

Secure Boot experiments involve clearing platform keys, enrolling custom certificates, and booting unsigned binaries. Doing this on your real machine risks locking yourself out. A VM gives you:

- **Full UEFI firmware** — the VM runs [OVMF](https://github.com/tianocore/tianocore.github.io/wiki/OVMF) (Open Virtual Machine Firmware), a UEFI implementation from the [TianoCore EDK II](https://github.com/tianocore/edk2) project. It implements the same Secure Boot specification as physical boards.

- **Instant recovery** — if you corrupt the firmware variables, simply delete the VM and start over in seconds.

- **No special hardware** — any Linux host works. KVM hardware virtualization is recommended for speed but not required. If `/dev/kvm` is not available, remove `-enable-kvm` and replace `-cpu host` with `-cpu qemu64` in the QEMU commands below.

## What You Need

| Component | Package | Purpose |
|-----------|---------|---------|
| QEMU | `qemu-system-x86` | Machine emulator and virtualizer |
| OVMF | `ovmf` | UEFI firmware for the VM |
| A Linux ISO | [Debian netinst](https://www.debian.org/distrib/netinst) (minimal, ~600 MB) | Guest OS to install |

## Step 1: Install Packages

```bash
sudo apt update
sudo apt install qemu-system-x86 ovmf
```

After installation, verify the OVMF firmware files are in place:

```bash
ls /usr/share/OVMF/
```

You should see several firmware variants:

| File | Purpose |
|------|---------|
| `OVMF_CODE_4M.fd` | Basic UEFI — no Secure Boot support |
| `OVMF_CODE_4M.secboot.fd` | UEFI with Secure Boot support, no keys pre-enrolled |
| `OVMF_CODE_4M.ms.fd` | Secure Boot with Microsoft keys baked into the firmware code |
| `OVMF_CODE_4M.snakeoil.fd` | Secure Boot with test keys (for development and CI) |
| `OVMF_VARS_4M.fd` | Empty NVRAM — blank key databases (Setup Mode) |
| `OVMF_VARS_4M.ms.fd` | NVRAM pre-loaded with Microsoft PK, KEK, db, and dbx |
| `OVMF_VARS_4M.snakeoil.fd` | NVRAM with snakeoil test keys |

OVMF splits its firmware into two files: a **code** file (the firmware logic, loaded read-only) and a **vars** file (the NVRAM that stores settings and Secure Boot key databases, writable). You choose one of each.

For this lab we use:

- **`OVMF_CODE_4M.secboot.fd`** — firmware with Secure Boot support compiled in.
- **`OVMF_VARS_4M.fd`** — blank NVRAM with no keys enrolled.

This combination starts the VM in [Setup Mode](01_secure_boot.md#setup-mode-and-user-mode) with empty key databases, giving us full control over the key hierarchy.

> **The `.ms.fd` shortcut:** To quickly test the standard Microsoft trust model (like a retail PC out of the box), pair `OVMF_CODE_4M.ms.fd` with `OVMF_VARS_4M.ms.fd`. The VM will start in User Mode with Microsoft's PK, KEK, db, and dbx already enrolled — no manual key enrollment needed.

> **Note:** File names vary across distributions. Some use `OVMF_CODE.secboot.fd` (without `4M`) or place files under `/usr/share/qemu/` or `/usr/share/edk2/`. Run `dpkg -L ovmf` (Debian/Ubuntu) or `rpm -ql edk2-ovmf` (Fedora/RHEL) to find the exact paths on your system.

## Step 2: Create the VM Disk and NVRAM Copy

Create a virtual disk for the guest OS:

```bash
qemu-img create -f qcow2 secureboot-lab.qcow2 20G
```

Copy the NVRAM template so this VM has its own writable firmware variables:

```bash
cp /usr/share/OVMF/OVMF_VARS_4M.fd my_vars.fd
```

Each VM needs its own copy of the vars file. The code file is shared read-only and does not need to be copied.

## Step 3: Boot the VM with the Installer ISO

Download the Debian netinst ISO:

```bash
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.8.0-amd64-netinst.iso
```

> **Note:** The version number changes with each Debian point release. If the URL above returns a 404, check the [current ISO directory](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/) for the latest filename.

Start the VM with the ISO attached:

```bash
QEMU_AUDIO_DRV=none qemu-system-x86_64 \
    -enable-kvm \
    -machine q35 \
    -m 2048 \
    -cpu host \
    -vga std \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.secboot.fd \
    -drive if=pflash,format=raw,file=my_vars.fd \
    -drive file=secureboot-lab.qcow2,format=qcow2 \
    -cdrom debian-13.8.0-amd64-netinst.iso \
    -boot d,menu=on,splash-time=20000 \
    -vnc :0
```

> **Note:** QEMU defaults to opening a graphical window (GTK), which requires a local display. If you are running QEMU over SSH or on a headless server, this will fail with `gtk initialization failed`. The `-vnc :0` flag starts a VNC server on port 5900 instead. Connect from your local machine using any VNC client (TigerVNC, RealVNC, Remmina) at `<host-ip>:5900`. If you are running QEMU on a machine with a local desktop, you can omit `-vnc :0` and use the default GTK window.

What each flag does:

| Flag | Purpose |
|------|---------|
| `QEMU_AUDIO_DRV=none` | Disable audio — avoids harmless PipeWire warnings on headless servers. |
| `-enable-kvm` | Use hardware virtualization for near-native speed. Remove this and use `-cpu qemu64` if `/dev/kvm` is not available. |
| `-machine q35` | Emulate the Q35 chipset. OVMF works best with Q35 — the older default (`i440fx`) may fail to initialize the display. |
| `-vga std` | Emulate a standard VGA card. Ensures OVMF can initialize the graphical display. |
| `-m 2048` | Allocate 2 GB of RAM to the guest. |
| `-cpu host` | Pass through the host CPU features to the guest. Replace with `-cpu qemu64` if KVM is not available. |
| First `-drive if=pflash,...` | Load the UEFI firmware code (read-only). This replaces the default legacy BIOS (SeaBIOS). |
| Second `-drive if=pflash,...` | Load the writable NVRAM. Secure Boot key databases are stored here. |
| `-drive file=...qcow2` | Attach the virtual disk. |
| `-cdrom ...iso` | Attach the installer ISO as a virtual CD-ROM drive. |
| `-boot d,menu=on,splash-time=20000` | Boot from CD-ROM first. Show the TianoCore logo for 20 seconds so you can press **F2** to enter the firmware setup. |
| `-vnc :0` | Start a VNC server on port 5900 for headless access. Omit this flag if you have a local desktop. |

The TianoCore splash screen should appear briefly, followed by the Debian installer.

## Step 4: Install the Guest OS

The Debian UEFI installer menu presents several options. Select **Install** (the text-mode option). It is faster over VNC and provides the same installation steps.

Follow the installer prompts with the default options. A minimal installation is sufficient — no desktop environment needed. The installer will automatically create a GPT partition table, an EFI System Partition (ESP), and install GRUB as a UEFI bootloader.

When the installation finishes, shut down the VM instead of rebooting. The installer ISO must be removed from the QEMU command before the next boot (Step 5), otherwise the VM will boot into the installer again. If the VM does reboot into the installer, simply power it off — close the VNC window and press `Ctrl+C` in the terminal where QEMU is running.

## Step 5: Boot from Disk

Now boot the VM without the installer ISO. The `-cdrom` flag is removed entirely and the `-boot` flag no longer includes `d` (CD-ROM first):

```bash
QEMU_AUDIO_DRV=none qemu-system-x86_64 \
    -enable-kvm \
    -machine q35 \
    -m 2048 \
    -cpu host \
    -vga std \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.secboot.fd \
    -drive if=pflash,format=raw,file=my_vars.fd \
    -drive file=secureboot-lab.qcow2,format=qcow2 \
    -boot menu=on,splash-time=20000 \
    -nic user,hostfwd=tcp::2222-:22 \
    -vnc :0
```

> **Tip:** Save this command as a shell script (e.g., `start-vm.sh`) so you do not have to retype it each time.

The `-nic user,hostfwd=tcp::2222-:22` flag forwards host port 2222 to the guest's SSH port. Once the VM is running, you can SSH in from the host instead of using VNC:

```bash
ssh -p 2222 <user>@localhost
```

This gives you a normal terminal with full copy-paste support. VNC is still needed for the firmware setup menu (F2), but all command-line work is easier over SSH.

The TianoCore logo appears briefly, followed by the GRUB menu and the Linux login prompt. Verify that the guest OS boots successfully before continuing.

## Step 6: Enter the OVMF Firmware Setup

Before proceeding to key enrollment, verify that the firmware recognizes Secure Boot. Restart the VM — run `sudo reboot` inside the guest or power-cycle it — and press **F2** during the TianoCore logo (the 20-second splash screen). This enters the OVMF setup menu, the equivalent of the BIOS/UEFI setup on a physical machine.

Navigate to **Device Manager → Secure Boot Configuration**:

- **Current Secure Boot State:** should show **Disabled** — Secure Boot is not enforced.
- **Secure Boot Mode:** should show **Setup Mode** — no PK is enrolled.

This confirms the VM has Secure Boot capability but is not yet enforcing it. The firmware is waiting for you to enroll keys.

> **Why is there no "Install default keys" option?** Unlike a physical motherboard (see [Secure Boot in Practice](02_secure_boot_in_practice.md)), the OVMF `secboot` firmware does not ship with embedded default keys. It was built with Secure Boot *capability* (the verification logic) but without any vendor certificates to install. To get pre-enrolled Microsoft keys in a VM, use the `.ms.fd` firmware and NVRAM variants described in [Step 1](#step-1-install-packages). This lab intentionally starts with empty databases so you can enroll your own keys manually.

> **What if you do not see the Secure Boot option?** You may have used `OVMF_CODE_4M.fd` instead of `OVMF_CODE_4M.secboot.fd`. The non-secboot variant does not include Secure Boot support. Switch to the `secboot` firmware file and restart the VM.

## Step 7: Verify from Inside Linux

Boot into the guest OS, install the Secure Boot tools, and check the firmware state from the command line:

```bash
sudo apt install mokutil efitools sbsigntool uuid-runtime
```

| Tool | Purpose |
|------|---------|
| `mokutil` | Query Secure Boot state and manage Machine Owner Keys (MOK) |
| `efitools` | Read and manipulate UEFI Secure Boot variables (PK, KEK, db, dbx) |
| `sbsigntool` | Sign and verify EFI binaries using Authenticode signatures |
| `uuid-runtime` | Provides `uuidgen` for generating GUIDs (needed during key enrollment) |

Check whether Secure Boot is enforced:

```bash
mokutil --sb-state
```

Expected output:

```
SecureBoot disabled
Platform is in Setup Mode
```

This confirms that the firmware supports Secure Boot but is not currently enforcing signature verification. This is the expected state — no keys have been enrolled yet.

Check which keys are enrolled:

```bash
efi-readvar
```

Expected output (with blank NVRAM):

```
Variable PK has no entries
Variable KEK has no entries
Variable db has no entries
Variable dbx has no entries
Variable MokList has no entries
```

All four Secure Boot databases are empty. The firmware is in [Setup Mode](01_secure_boot.md#setup-mode-and-user-mode), ready for you to enroll your own keys.

## Step 8: Back Up the NVRAM

Before making any changes to the firmware variables, back up the NVRAM file on the **host** (not inside the VM). This allows you to restore the VM to its current state if anything goes wrong.

```bash
cp my_vars.fd my_vars.fd.bak
```

> **Tip:** Create a fresh backup before each experiment. If you need to start over, shut down the VM and run `cp my_vars.fd.bak my_vars.fd`.

## Step 9: Generate Custom Secure Boot Keys

Inside the guest VM, create three key pairs — one for each level of the [Secure Boot key hierarchy](01_secure_boot.md#who-controls-db-and-dbx-the-key-hierarchy):

```bash
mkdir -p ~/sb-keys && cd ~/sb-keys

# Platform Key (PK) — the top-level key; establishes platform ownership
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=Lab Platform Key/" \
    -keyout PK.key -out PK.crt -days 3650 -nodes -sha256

# Key Exchange Key (KEK) — authorizes updates to db and dbx
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=Lab Key Exchange Key/" \
    -keyout KEK.key -out KEK.crt -days 3650 -nodes -sha256

# Signature Database key (db) — signs EFI binaries that the firmware should trust
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=Lab Signing Key/" \
    -keyout db.key -out db.crt -days 3650 -nodes -sha256
```

Each command generates an RSA 2048-bit key pair and a self-signed X.509 certificate. The `.key` file is the private key (used for signing — never stored in firmware) and the `.crt` file is the public certificate (this is what gets enrolled in the firmware).

> **Lab keys only.** This lab uses RSA-2048, SHA-256, and unencrypted private keys on disk (`-nodes`) for simplicity. In a production environment, Secure Boot keys should use stronger cryptography (RSA-4096, SHA-384 or higher) and private keys should never exist as files — they are generated and stored inside a Hardware Security Module (HSM) and accessed via PKCS#11.

Next, convert the certificates into the binary format that UEFI firmware stores internally — the **EFI Signature List** (`.esl`):

```bash
# Generate a GUID to identify us as the owner of these entries
GUID=$(uuidgen)

# Convert certificates to EFI Signature List format
cert-to-efi-sig-list -g "$GUID" PK.crt PK.esl
cert-to-efi-sig-list -g "$GUID" KEK.crt KEK.esl
cert-to-efi-sig-list -g "$GUID" db.crt db.esl

# PK enrollment requires a signed authenticated variable update (.auth),
# even in Setup Mode. Create one by signing the PK ESL with the PK itself:
sign-efi-sig-list -g "$GUID" -k PK.key -c PK.crt PK PK.esl PK.auth
```

The file formats at each stage:

| File | Format | Purpose |
|------|--------|---------|
| `.key` | PEM private key | Signs updates and EFI binaries. Never stored in firmware. |
| `.crt` | PEM X.509 certificate | The public certificate in human-readable format. |
| `.esl` | EFI Signature List | The certificate in the binary format that firmware stores internally. |
| `.auth` | Authenticated Variable Update | The `.esl` wrapped with a digital signature and timestamp — the format required for PK enrollment. |

> **Note:** In [User Mode](01_secure_boot.md#user-mode), modifying any key database requires [authenticated variable updates](01_secure_boot.md#authenticated-variables) — `.auth` files signed by the appropriate private key. For db and KEK, we can skip that here because the platform is still in Setup Mode. The PK is the exception — `efi-updatevar` requires a signed `.auth` file for PK enrollment regardless of the platform mode.

## Step 10: Enroll the Keys

Write the keys into the firmware's NVRAM. The order matters — **enroll db and KEK first, then PK last**. Enrolling the PK transitions the platform from [Setup Mode to User Mode](01_secure_boot.md#setup-mode-and-user-mode), after which further changes require properly signed authenticated updates.

The Linux kernel sets the **immutable** file attribute on EFI variable files in `/sys/firmware/efi/efivars/` to prevent accidental modification. If a variable already exists, this attribute must be cleared before writing. Variables that do not exist yet (no file in efivarfs) do not need this step.

```bash
cd ~/sb-keys

# Enroll db (clear the immutable flag first — db exists in efivarfs even when empty)
sudo chattr -i /sys/firmware/efi/efivars/db-d719b2cb-3d3a-4596-a3bc-dad00e67656f
sudo efi-updatevar -e -f db.esl db

# Enroll KEK (no chattr needed — KEK does not exist in efivarfs yet)
sudo efi-updatevar -e -f KEK.esl KEK

# Enroll PK last (uses the signed .auth file; triggers Setup Mode → User Mode)
sudo efi-updatevar -f PK.auth PK
```

Verify that all keys are enrolled:

```bash
efi-readvar
```

You should see your three certificates listed under PK, KEK, and db with the subject names you chose (`Lab Platform Key`, `Lab Key Exchange Key`, `Lab Signing Key`). The dbx and MokList variables should still have no entries.

Check the platform mode:

```bash
mokutil --sb-state
```

Expected output:

```
SecureBoot disabled
```

Notice that the "Platform is in Setup Mode" line from [Step 7](#step-7-verify-from-inside-linux) is gone. `mokutil` only prints that line when the platform is in Setup Mode — its absence confirms the platform has transitioned to **User Mode** (PK is enrolled, key databases are protected). Secure Boot **enforcement** is still off. This matches the [important distinction](01_secure_boot.md#an-important-distinction-user-mode--secure-boot-on) covered in the theory: User Mode protects the databases, but image verification is a separate setting that must be enabled explicitly.

## Step 11: Sign the Bootloader

Before enabling Secure Boot enforcement, sign the boot chain so the firmware will accept it.

The Debian installer set up [shim](01_secure_boot.md#the-shim-bootloader) as the first-stage bootloader. The firmware's boot entry points to `shimx64.efi`, which is currently signed only by Microsoft — a certificate that is **not** in our custom db. We need to add our own signature so the firmware will accept it.

```bash
cd ~/sb-keys

# Back up the original shimx64.efi
sudo cp /boot/efi/EFI/debian/shimx64.efi /boot/efi/EFI/debian/shimx64.efi.bak

# Add our signature (sbsign appends alongside any existing signatures)
sudo sbsign --key db.key --cert db.crt \
    --output /boot/efi/EFI/debian/shimx64.efi \
    /boot/efi/EFI/debian/shimx64.efi
```

Verify that our signature is present and valid:

```bash
sudo sbverify --cert db.crt /boot/efi/EFI/debian/shimx64.efi
```

Expected output:

```
Signature verification OK
```

You can also list all signatures on the binary:

```bash
sudo sbverify --list /boot/efi/EFI/debian/shimx64.efi
```

The output shows the original Microsoft signatures plus your `Lab Signing Key` as an additional signature. A PE/COFF binary can carry [multiple Authenticode signatures](01_secure_boot.md#what-is-inside-the-attribute-certificate-table) — the firmware checks each one and authorizes the binary if **any** signature chains to a certificate in db.

> **Why sign shim instead of GRUB?** The firmware only verifies the **first** EFI binary it loads — in this case, `shimx64.efi`. After that, shim takes over and verifies GRUB and the kernel using the [distro's certificate embedded inside shim](01_secure_boot.md#what-is-inside-shim). By signing only shim with our db key, the entire existing verification chain continues to work: the firmware trusts shim (our signature), and shim trusts GRUB and the kernel (Debian's embedded signature).

## Step 12: Reboot and Verify

OVMF automatically enables Secure Boot enforcement when a PK is enrolled — no manual step is needed. This is a platform policy choice by the OVMF firmware. Some physical boards require you to enable Secure Boot separately (as shown in [Secure Boot in Practice](02_secure_boot_in_practice.md)); OVMF does not.

> **Note:** `mokutil --sb-state` still showed "disabled" in the previous step because it reflects the boot session that was running at the time. The firmware reads the updated NVRAM on the **next** boot.

Reboot the VM:

```bash
sudo reboot
```

The firmware now verifies every EFI binary before executing it. On this boot:

1. The firmware loads `shimx64.efi` from the ESP.
2. It checks the Authenticode signature against db — our `Lab Signing Key` certificate matches. ✅
3. shim runs, loads `grubx64.efi`, and verifies it using Debian's embedded certificate. ✅
4. GRUB loads the kernel, and shim verifies it using Debian's certificate. ✅
5. Linux boots.

Once logged in, confirm that Secure Boot is active:

```bash
mokutil --sb-state
```

Expected output:

```
SecureBoot enabled
```

Verify that your keys are in the firmware:

```bash
efi-readvar
```

The output should show your `Lab Platform Key`, `Lab Key Exchange Key`, and `Lab Signing Key` in PK, KEK, and db respectively.

Secure Boot is now fully operational with your own keys. The firmware trusts only EFI binaries signed by your `Lab Signing Key` — no Microsoft certificates, no third-party trust.

## Step 13: Test Rejection

To confirm that Secure Boot actually blocks unauthorized code, run the following **inside the VM** to restore the original `shimx64.efi` (which carries only Microsoft's signatures, without ours) and reboot:

```bash
sudo cp /boot/efi/EFI/debian/shimx64.efi.bak /boot/efi/EFI/debian/shimx64.efi
sudo reboot
```

The firmware loads `shimx64.efi`, extracts the PKCS#7 signatures, and checks whether any signing certificate chains to an entry in db. None do — the original shim is signed only by Microsoft, and our db contains only `Lab Signing Key`. The firmware rejects the binary with a message like:

```
BdsDxe: failed to load Boot0008 "debian" from ...\EFI\debian\shimx64.efi:
Access Denied -- rejected probably by Secure Boot
```

The VM then falls through to the next boot option (typically PXE network boot) and eventually stops at the TianoCore logo. This is exactly the [authorization flow](01_secure_boot.md#how-authorization-works) described in the theory.

> **Recovery:** Press **F2** during the TianoCore splash screen to enter the firmware setup. Disable Secure Boot under **Device Manager → Secure Boot Configuration** to regain access to the guest OS. Alternatively, shut down the VM and restore the NVRAM backup on the host: `cp my_vars.fd.bak my_vars.fd` (this resets the firmware to Step 7, before key enrollment).

After observing the rejection, recover access to the guest by pressing **F2** to disable Secure Boot (as described in the recovery note above), then boot into Linux and re-sign shim to restore normal boot:

```bash
cd ~/sb-keys
sudo sbsign --key db.key --cert db.crt \
    --output /boot/efi/EFI/debian/shimx64.efi \
    /boot/efi/EFI/debian/shimx64.efi.bak
```

Re-enable Secure Boot in the firmware setup (if you disabled it to recover), and verify that the VM boots successfully again.

---

[← Previous: Secure Boot in Practice](02_secure_boot_in_practice.md)
