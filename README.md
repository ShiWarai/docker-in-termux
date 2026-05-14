# Installing Docker in Termux (ARM64 / aarch64 guest)

This fork documents a single path: **Alpine Linux aarch64** in **QEMU** on **ARM64 Termux**, then **Docker** with **linux/arm64** images. Docker runs inside the VM, not in Termux itself.

# Prerequisites
- An **aarch64** Android device with [Termux](https://termux.com/) (e.g. from [F-Droid](https://f-droid.org/packages/com.termux/)). Check with `uname -m` — it should print `aarch64`.
- Stable internet connection.

The automated install uses [answerfile](answerfile). By default `DISKOPTS` installs to **`/dev/vda`** (typical for a single virtio disk). If `lsblk` shows a different device, edit the last line of `answerfile` before `setup-alpine`.

# Installation steps

1. Open Termux.

2. Update packages:
```bash
pkg update -y && pkg upgrade -y
```

3. Install QEMU (aarch64 system emulator) and tools. If the package name changes, run `pkg search qemu-system-aarch64`:
```bash
pkg install qemu-utils qemu-common qemu-system-aarch64-headless wget -y
```

4. Working directory:
```bash
mkdir alpine && cd alpine
```

5. Download Alpine **3.20.2** virt ISO for **aarch64** (matches v3.20 in `answerfile`):
```bash
wget http://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-virt-3.20.2-aarch64.iso
```

6. Disk image:
```bash
qemu-img create -f qcow2 alpine.img 5G
```
The qcow2 file grows as needed (often on the order of hundreds of MB, not the full 5G immediately).

7. Writable UEFI NVRAM (per-directory copy so boot variables can be stored):
```bash
cp "$PREFIX/share/qemu/edk2-aarch64-vars.fd" ./edk2-aarch64-vars.fd
```
If firmware files are missing: `ls "$PREFIX/share/qemu" | grep edk2` — see [termux-packages](https://github.com/termux/termux-packages) or the Termux wiki for your build.

8. First boot from ISO (1024 MiB RAM, 2 CPUs — tune with `nproc` and `free -m` on the host):
```bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 1024 \
  -smp cpus=2 \
  -drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-aarch64-code.fd \
  -drive if=pflash,format=raw,file=$PWD/edk2-aarch64-vars.fd \
  -netdev user,id=n1,dns=8.8.8.8,hostfwd=tcp::2222-:22 \
  -device virtio-net,netdev=n1 \
  -drive if=none,id=cd0,format=raw,media=cdrom,readonly=on,file=$PWD/alpine-virt-3.20.2-aarch64.iso \
  -device virtio-scsi-pci,id=scsi0 \
  -device scsi-cd,bus=scsi0.0,drive=cd0 \
  -drive if=none,id=hd0,file=$PWD/alpine.img,format=qcow2 \
  -device virtio-blk-pci,drive=hd0 \
  -boot order=d \
  -nographic
```
You can try `-cpu max` if QEMU accepts it. If allocating more than ~4 GiB RAM fails on some devices, lower `-m`.

9. Log in as `root` (no password).

10. Network (accept defaults for DHCP on `eth0`):
```bash
setup-interfaces
ifup eth0
```

11. Fetch [answerfile](answerfile). Replace **`YOUR_GITHUB_USER`** / **`YOUR_REPO`** with your fork (and branch if not `main`), or copy `answerfile` from a local clone:
```bash
wget -O answerfile "https://raw.githubusercontent.com/YOUR_GITHUB_USER/YOUR_REPO/main/answerfile"
```
> If DNS fails inside the installer, e.g. `wget: bad address '...'`, fix resolver first, for example:
> ```bash
> echo -e "nameserver 192.168.1.1\nnameserver 1.1.1.1" > /etc/resolv.conf
> ```

12. Confirm the install disk (see prerequisites): `lsblk`. Edit `DISKOPTS` in `answerfile` if it is not `/dev/vda`.

13. Serial console for QEMU `virt` on AArch64 (often **`ttyAMA0`**):
```bash
sed -i -E 's/(local kernel_opts)=.*/\1="console=ttyAMA0"/' /sbin/setup-disk
```
If you get no login after reboot, try `console=ttyS0` instead.

14. Install to disk:
```bash
setup-alpine -f answerfile
```

15. `poweroff`

16. Boot installed system (no ISO):
```bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 1024 \
  -smp cpus=2 \
  -drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-aarch64-code.fd \
  -drive if=pflash,format=raw,file=$PWD/edk2-aarch64-vars.fd \
  -netdev user,id=n1,dns=8.8.8.8,hostfwd=tcp::2222-:22 \
  -device virtio-net,netdev=n1 \
  -drive if=none,id=hd0,file=$PWD/alpine.img,format=qcow2 \
  -device virtio-blk-pci,drive=hd0 \
  -nographic
```

**Optional — `run_qemu.sh`** (same flags as step 16; keep in the same directory as `alpine.img` and `edk2-aarch64-vars.fd`):
```bash
#!/bin/bash
exec qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 1024 \
  -smp cpus=2 \
  -drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-aarch64-code.fd \
  -drive if=pflash,format=raw,file=$PWD/edk2-aarch64-vars.fd \
  -netdev user,id=n1,dns=8.8.8.8,hostfwd=tcp::2222-:22 \
  -device virtio-net,netdev=n1 \
  -drive if=none,id=hd0,file=$PWD/alpine.img,format=qcow2 \
  -device virtio-blk-pci,drive=hd0 \
  -nographic
```
Then: `chmod +x run_qemu.sh` and `./run_qemu.sh`.

17. In the guest — DNS and Docker:
```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 8.8.4.4" >> /etc/resolv.conf

apk update && apk add docker
service docker start
rc-update add docker
docker run hello-world
```

# Native Docker without a VM (not this fork’s focus)

Docker directly in Termux usually needs **root** and a kernel with suitable **cgroups**, **namespaces**, and often **overlay** support. Many stock phone kernels are incomplete for that. The VM route above stays predictable.

# Useful QEMU keys
- `Ctrl+a` then `x`: quit emulation
- `Ctrl+a` then `h`: QEMU console help

# Usage
See [Docker documentation](https://docs.docker.com/) for day-to-day container use.

# Contributing
Issues and pull requests are welcome.

# Acknowledgment
- Inspired by: https://gist.github.com/oofnikj/e79aef095cd08756f7f26ed244355d62  
- Upstream-style dual-arch guide: [cyberkernelofficial/docker-in-termux](https://github.com/cyberkernelofficial/docker-in-termux) (this fork is aarch64-only).

# License
[MIT License](LICENSE).
