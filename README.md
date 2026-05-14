# Docker в Termux (Alpine aarch64 в QEMU)

Docker крутится **внутри ВМ** (Alpine aarch64), не в самом Termux. Нужен **aarch64**-телефон (`uname -m` → `aarch64`), [Termux](https://termux.com/) ([F-Droid](https://f-droid.org/packages/com.termux/)), интернет.

Автоустановка: [answerfile](answerfile), по умолчанию диск **`/dev/vda`** (шаг 10).

Зеркала: при сообщении про репозиторий — `termux-change-repo`, затем `pkg update`.

## Установка

### 1. Пакеты

```bash
pkg update -y && pkg upgrade -y
pkg install qemu-utils qemu-common qemu-system-aarch64-headless ovmf wget -y
```

### 2. Каталог проекта

```bash
mkdir alpine_arm64 && cd alpine_arm64
```

Дальше ISO, `alpine.img` и `edk2-aarch64-vars.fd` лежат **здесь**.

### 3. ISO Alpine 3.20.x (virt, aarch64)

Ветка в [answerfile](answerfile) — **v3.20**. Имя `.iso` в шаге 6 должно совпадать с файлом из шага 3 (`ls *.iso`). Пример **3.20.10**:

```bash
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/aarch64/alpine-virt-3.20.10-aarch64.iso
```

### 4. Диск гостя

```bash
qemu-img create -f qcow2 alpine.img 5G
```

### 5. UEFI NVRAM (ovmf + 64 MiB)

В `share/qemu` часто **нет** `edk2-aarch64-vars.fd`. Берём шаблон из **ovmf**, копируем в каталог проекта и **добиваем до 64 MiB** (иначе QEMU: `pflash1 … requires 67108864 bytes`):

```bash
cp "$PREFIX/share/edk2/aarch64/QEMU_VARS.fd" ./edk2-aarch64-vars.fd
truncate -s 64M ./edk2-aarch64-vars.fd
```

Нужен файл из **`…/edk2/aarch64/`**, не из `…/edk2/arm/`. Нет файла: `find "$PREFIX/share" -name 'QEMU_VARS.fd'`.

### 6. Первый запуск (ISO + диск)

Подстройте `-m` / `-smp` под телефон; RAM гостя **> ~4 GiB** на части устройств падает. Для Huawei Y8p рабочий пример: **2048 MiB / 4 CPU**.

Вставляйте блок **целиком** (без разрыва `scsi-cd` и без отрыва `-` от `boot`):

```bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 2048 \
  -smp cpus=4 \
  -drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-aarch64-code.fd \
  -drive if=pflash,format=raw,file=$PWD/edk2-aarch64-vars.fd \
  -netdev user,id=n1,dns=8.8.8.8,hostfwd=tcp::2222-:22 \
  -device virtio-net,netdev=n1,romfile= \
  -drive if=none,id=cd0,format=raw,media=cdrom,readonly=on,file=$PWD/alpine-virt-3.20.10-aarch64.iso \
  -device virtio-scsi-pci,id=scsi0 \
  -device scsi-cd,bus=scsi0.0,drive=cd0 \
  -drive if=none,id=hd0,file=$PWD/alpine.img,format=qcow2 \
  -device virtio-blk-pci,drive=hd0 \
  -boot order=d \
  -nographic
```

Имя `.iso` в `file=$PWD/…` = как в шаге 3. `romfile=` у сети отключает EFI option ROM, из-за которого UEFI может писать `Image type X64`. Остальные сообщения UEFI (TPM, `Image …`) обычно шум — ждите GRUB/консоль установщика.

**Не грузится (Shell / Boot Manager):** пересоздать образы в **этой** папке, проверить имя ISO, вставить команду без битых переносов; в меню — **Boot Manager → CD/ISO**.

```bash
rm -f alpine.img edk2-aarch64-vars.fd
qemu-img create -f qcow2 alpine.img 5G
cp "$PREFIX/share/edk2/aarch64/QEMU_VARS.fd" ./edk2-aarch64-vars.fd
truncate -s 64M ./edk2-aarch64-vars.fd
```

### 7. В госте: `root` (без пароля)

### 8. Сеть

```bash
setup-interfaces
```
Везде **Enter** по умолчанию (eth0, dhcp). Затем:

```bash
ifup eth0
```

### 9. `answerfile`

```bash
wget -O answerfile "https://raw.githubusercontent.com/ShiWarai/docker-in-termux/main/answerfile"
```

Ошибка DNS в госте: `echo -e "nameserver 192.168.1.1\nnameserver 1.1.1.1" > /etc/resolv.conf`

### 10. Диск для `setup-alpine`

В `answerfile` уже **`DISKOPTS=… /dev/vda`**. Проверка: `cat /proc/partitions` — нужен **vda** (образ), не **sr0**/loop. Иначе поправьте последнюю строку `answerfile`.

### 11. Serial-консоль

```bash
sed -i -E 's/(local kernel_opts)=.*/\1="console=ttyAMA0"/' /sbin/setup-disk
```

Нет логина после установки — попробовать **`console=ttyS0`**.

### 12. Установка на диск

```bash
setup-alpine -f answerfile
```

### 13. Выключение гостя

```bash
poweroff
```

### 14. Запуск без ISO

ISO не подключаем. В том же каталоге создайте **`run_qemu.sh`** (`nano run_qemu.sh`):

```bash
#!/bin/bash
qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57 \
  -m 2048 \
  -smp cpus=4 \
  -drive if=pflash,format=raw,readonly=on,file=$PREFIX/share/qemu/edk2-aarch64-code.fd \
  -drive if=pflash,format=raw,file=$PWD/edk2-aarch64-vars.fd \
  -netdev user,id=n1,dns=8.8.8.8,hostfwd=tcp::2222-:22 \
  -device virtio-net,netdev=n1,romfile= \
  -drive if=none,id=hd0,file=$PWD/alpine.img,format=qcow2 \
  -device virtio-blk-pci,drive=hd0,bootindex=0 \
  -nographic
```

Сохранить: **Ctrl+X**, **Y**, **Enter**. Запуск:

```bash
chmod +x run_qemu.sh && ./run_qemu.sh
```

Дальше обычно **`./run_qemu.sh`** (или `bash run_qemu.sh`).

**Shell / «UEFI Misc Device» после установки:** в Termux **Ctrl+a** **x**, снова шаг **5** (vars), затем **`./run_qemu.sh`**. Не помогло — шаг **12** или заново шаги **3–6** (ISO/образ/vars/первый запуск).

### 15. Docker в госте

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
echo "nameserver 8.8.4.4" >> /etc/resolv.conf
apk update && apk add docker
service docker start
rc-update add docker
docker run hello-world
```

## QEMU `-nographic`

- **Ctrl+a** **x** — выход из эмулятора  
- **Ctrl+a** **h** — справка QEMU  

## Прочее

Нативный Docker в Termux без ВМ на стоковых прошивках обычно **не вариант** (ядро, root, cgroups).

Дальше — [документация Docker](https://docs.docker.com/).

Issues/PR приветствуются.

**Источники:** [gist oofnikj](https://gist.github.com/oofnikj/e79aef095cd08756f7f26ed244355d62), upstream [cyberkernelofficial/docker-in-termux](https://github.com/cyberkernelofficial/docker-in-termux) (x86_64 + aarch64; этот форк — aarch64).

**Лицензия:** [LICENSE](LICENSE), [LICENSE.RU](LICENSE.RU).
