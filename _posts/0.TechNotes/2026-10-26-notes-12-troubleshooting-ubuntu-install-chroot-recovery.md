---
title: 설치가 중간에 끊긴 Ubuntu Server를 라이브 USB에서 복구하기
description: 갤럭시북에 Ubuntu 24.04 Server를 설치하다 설치기가 중간에 멈춘 뒤, 다시 설치하지 않고 설치 USB의 셸에서 디스크로 들어가 계정과 호스트 이름, 네트워크를 채우고, 빠진 와이파이 패키지를 USB로 옮겨 설치해 살린 기록
date: 2026-10-26
series: TechNotes
tags:
  - TroubleShooting
  - Ubuntu
  - Linux
---

## 요약

> 프로젝트 클러스터의 워커 노드로 쓸 갤럭시북에 Ubuntu 24.04 Server를 설치하는데, 설치기가 마지막 단계 근처에서 멈추는 일이 두 대에서 있었습니다. 다시 설치해도 같은 자리에서 멈췄기 때문에, 설치기가 디스크에 이미 넣어 둔 파일은 그대로 두고 빠진 것만 직접 채우는 쪽으로 방향을 바꿨습니다.
>
> 설치 USB의 셸에서 `chroot`로 디스크 안의 시스템에 들어가 계정과 호스트 이름, 네트워크 설정을 만들었고, 부팅 뒤 와이파이가 붙지 않는 문제는 설치가 끊기면서 빠진 `wpasupplicant` 패키지를 Mac에서 받아 USB로 옮겨 설치하는 것으로 해결했습니다.

짧게 줄이면 이렇습니다.

1. 설치기가 중간에 멈추고, USB를 빼고 부팅하면 `localhost login:`만 나옴
2. 설치 USB로 다시 부팅해 tty2에서 계정을 만들었지만 재부팅하니 사라짐. USB 라이브 환경에 만든 것이었음
3. `chroot /target`으로 디스크 안에 들어가 계정, 호스트 이름, 임시 netplan을 작성
4. 부팅은 되지만 와이파이가 안 붙음. `wpasupplicant`가 설치되지 않은 상태
5. 인터넷이 없으므로 Mac에서 deb 파일을 받아 USB로 옮겨 `dpkg -i`로 설치
6. 와이파이가 붙은 뒤 `openssh-server`, `tailscale`을 설치해 평소대로 접속

## 1. 문제 상황

갤럭시북 세 대를 Ubuntu 24.04 Server로 설치해 클러스터 워커로 쓰려고 했습니다. 이 중 두 대에서 설치기가 끝까지 가지 못했고, 증상은 조금 달랐습니다.

| 기기 | 시점 | 증상 |
|---|---|---|
| io (처음 워커 후보) | 10/22 | 설치 마지막 GRUB 단계에서 매번 `grub-install: error: failed to register the EFI boot entry: Cannot allocate memory`로 실패 |
| terra | 10/26 | 설치 중 에러로 멈춤. USB를 빼고 부팅하면 `localhost login:` 화면만 나오고 만든 적 없는 계정으로는 들어갈 수 없음 |

terra는 부팅까지는 됐기 때문에 GRUB은 설치된 상태였고, 설치기가 계정과 호스트 이름을 넣는 단계 전에 끊긴 것으로 보였습니다. io는 GRUB 설치 자체가 실패해서 부팅부터 되지 않았습니다.

두 경우 모두 다시 설치하면 같은 자리에서 다시 멈췄기 때문에, 설치를 반복하는 대신 설치기가 어디까지 했는지 확인하고 나머지를 직접 채우기로 했습니다.

## 2. 설치기가 남긴 것과 빠진 것

Ubuntu 설치기는 설치 대상 디스크를 `/target`에 연결해 두고 그 안에 파일을 복사합니다. 설치 화면에서 `Ctrl + Alt + F2`를 누르면 tty2 셸이 열리는데, 여기서 `/target` 안을 보면 설치기가 어디까지 진행했는지 알 수 있습니다.

```bash
sudo -i
```

`sudo -i`는 다른 명령과 함께 붙여 넣지 않고 단독으로 실행합니다. 새 셸이 열리면서 뒤에 붙인 명령은 버려지기 때문입니다. tty2에 처음 붙은 사용자는 일반 사용자라 `chroot`가 `Operation not permitted`로 막히므로, root 셸로 먼저 바꿔야 합니다.

```bash
lsblk -f | grep -v loop
cat /target/etc/hostname
grep orbit /target/etc/passwd
ls /target/boot/efi/EFI
```

- `lsblk -f` : `/target`과 `/target/boot/efi`에 연결된 파티션이 보여야 합니다. `grep -v loop`은 라이브 환경의 squashfs 장치를 숨깁니다.
- `/target/etc/hostname` : `localhost.localdomain`이면 호스트 이름을 넣기 전에 끊긴 것입니다.
- `/target/etc/passwd` : 만들려던 계정이 없으면 계정 단계 전에 끊긴 것입니다.
- `/target/boot/efi/EFI` : `BOOT`와 `ubuntu`가 있으면 GRUB은 설치된 것입니다.

terra는 OS 파일과 GRUB까지 있었고 계정, 호스트 이름, `/etc/netplan` 설정이 비어 있었습니다. 설치가 끊기면서 패키지 설치도 끝까지 가지 못해, 당연히 있을 것으로 생각한 `wpasupplicant`와 `openssh-server`도 빠져 있었는데 이것은 부팅한 뒤에야 알았습니다.

## 3. 첫 실수: 라이브 환경에 계정을 만듦

tty2에서 바로 `adduser orbit`을 실행하고 재부팅했는데, 로그인 화면은 그대로 `localhost`였고 만든 계정으로 들어갈 수 없었습니다. tty2의 `/`는 디스크가 아니라 설치 USB의 라이브 환경이기 때문에, 거기서 만든 계정은 USB 메모리의 임시 영역에만 생기고 재부팅하면 사라집니다.

지금 어느 환경에 있는지는 루트 파일시스템을 보면 알 수 있습니다.

```bash
df -h /
```

`/`가 `/cow`로 나오면 USB 라이브 환경입니다. 여기에 설치하거나 만든 것은 전부 재부팅과 함께 없어집니다. io 때는 같은 이유로 라이브 환경에 tailscale을 설치했다가 재부팅 뒤 사라지는 일도 있었습니다.

> **디스크를 고칠 때 반드시 chroot를 쓰는 이유**
>
> 라이브 환경에서 `adduser`나 `apt install`을 실행하면 명령은 성공하지만 대상이 USB입니다. 디스크에 설치된 시스템을 바꾸려면 그 시스템의 루트 디렉터리를 현재 루트로 바꾼 뒤 명령을 실행해야 하고, 그 역할을 `chroot`가 합니다. 그래서 설치 USB에서 디스크를 손보는 작업은 전부 `chroot /target` 안에서 하기로 했습니다.

## 4. 디스크 안으로 들어가기

설치기가 `/target`을 연결해 둔 상태를 그대로 쓰되, 설치기를 다시 부팅해서 `/target`이 비어 있으면 루트 파티션을 찾아 직접 연결합니다.

```bash
lsblk -f | grep -v loop
mount /dev/nvme0n1p2 /target
mount /dev/nvme0n1p1 /target/boot/efi
```

- terra는 LVM 없이 `nvme0n1p2` ext4 하나와 `nvme0n1p1` EFI 파티션으로 설치했습니다.
- LVM으로 설치했다면 `vgchange -ay`로 볼륨 그룹을 켠 뒤 `/dev/ubuntu-vg/ubuntu-lv`를 연결합니다.

다음으로 커널과 장치 정보를 디스크 쪽에도 보이게 연결하고 `chroot`로 들어갑니다.

```bash
for d in dev proc sys run; do mount --rbind /$d /target/$d; done
chroot /target
```

- `--rbind` : `/dev`, `/proc`, `/sys`, `/run`을 하위 마운트까지 함께 `/target` 아래에 겹쳐 연결합니다. 이것이 없으면 `chroot` 안에서 장치를 찾지 못해 `adduser`나 `grub-install` 같은 명령이 실패합니다.

`chroot` 안에서 계정과 호스트 이름을 만듭니다.

```bash
adduser orbit
usermod -aG sudo orbit
echo terra > /etc/hostname
nano /etc/hosts
```

`/etc/hosts`에는 다음 내용을 넣었습니다.

```text
127.0.0.1 localhost
127.0.1.1 terra

::1 localhost ip6-localhost ip6-loopback
```

처음에는 `/etc/hosts`에 `127.0.0.1 terra`만 넣고 재부팅했는데 로그인 화면은 여전히 `localhost.localdomain`이었습니다. `/etc/hosts`는 이름을 주소로 바꾸는 조회용 파일이고, 화면에 나오는 호스트 이름은 `/etc/hostname`이 정하기 때문에 둘 다 써야 했습니다.

`/etc/netplan`이 비어 있었으므로 부팅 뒤 인터넷에 붙을 수 있도록 와이파이 DHCP 설정을 임시로 만들었습니다.

```bash
nano /etc/netplan/50-temp.yaml
chmod 600 /etc/netplan/50-temp.yaml
```

```yaml
network:
  version: 2
  wifis:
    wlo1:
      dhcp4: true
      access-points:
        "<와이파이 SSID>":
          password: "<와이파이 비밀번호>"
```

netplan 파일에 비밀번호가 들어가므로 권한을 `600`으로 줄였습니다. 이 설정은 복구용이고, 노드의 실제 네트워크는 이후 Ansible이 관리하는 netplan으로 바꿨습니다.

`chroot`를 나와 디스크에 쓰고 재부팅합니다.

```bash
exit
sync
reboot
```

`umount -R /target`을 먼저 실행하면 `/target/run/user/1000: target is busy`가 나오는데, `/run`을 겹쳐 연결해서 생기는 경고라 무시하고 `sync` 뒤 재부팅해도 문제가 없었습니다. 재부팅할 때 USB는 뽑습니다.

## 5. 부팅 자체가 안 될 때: GRUB을 따로 설치 (io)

io는 설치기가 GRUB을 NVRAM에 등록하는 단계에서 `Cannot allocate memory`로 매번 실패했습니다. 메모리 부족이 아니라 펌웨어가 EFI 변수 쓰기를 거부했다는 뜻이었고, 기존 부팅 항목을 지우거나 BIOS 기본값으로 되돌려도 같았습니다. UEFI 셸 USB로 부팅해 `bcfg boot rm 0`을 실행하니 펌웨어에 쓰기는 됐기 때문에, 펌웨어 고장이 아니라 리눅스에서 들어오는 쓰기만 거부되는 것으로 범위를 좁힐 수 있었습니다.

NVRAM에 쓰지 않고 부팅하도록 GRUB을 기본 경로에 설치했습니다.

```bash
sudo -i
for d in dev proc sys run; do mount --bind /$d /target/$d; done
chroot /target grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable --no-nvram
chroot /target update-grub
```

- `--no-nvram` : 펌웨어의 부팅 항목 목록에 Ubuntu를 등록하지 않습니다.
- `--removable` : GRUB을 `\EFI\BOOT\BOOTX64.EFI`에 설치합니다. 펌웨어가 기본으로 갖고 있는 `UEFI OS` 항목이 이 경로를 가리키므로 등록 없이도 부팅됩니다. 부팅 메뉴 이름이 Ubuntu 대신 UEFI OS로 나오는 것 말고는 차이가 없었습니다.

이 방법으로 부팅은 됐지만 계정, 호스트 이름, 네트워크, locale이 전부 빠져 있어서 4절과 같은 작업을 거쳐야 했습니다. 설치기 쪽에서 NVRAM 쓰기를 건너뛰도록 curtin 설정을 고치는 시도도 했는데, 설치기가 쓰는 `grub-multi-install`은 curtin 설정이 아니라 설치 대상 시스템의 debconf 값을 따르기 때문에 통하지 않았습니다.

> **io를 클러스터에서 뺀 이유**
>
> 같은 모델, 같은 BIOS 버전의 다른 갤럭시북은 정상이었고 삼성 BIOS는 Windows에서만 업데이트할 수 있어 펌웨어 쪽에서 더 해 볼 것이 없었습니다. UEFI 셸의 `dmpstore -d`로 변수를 전부 지우거나 커널 옵션 `efi_no_storage_paranoia`로 안전장치를 끄는 방법은 삼성 노트북이 벽돌이 된 사례가 있어 하지 않았습니다. 재설치할 때마다 이 과정을 반복해야 하므로 io는 노드에서 제외했고, 이 이름은 이후 홈서버의 부하 발생기가 쓰고 있습니다.

## 6. 와이파이가 안 붙음: 빠진 패키지를 USB로 설치

재부팅 뒤 만든 계정으로 로그인은 됐지만 와이파이가 붙지 않았습니다.

```bash
ip -br a
systemctl status netplan-wpa-wlo1 --no-pager | head -5
```

`wlo1`이 `NO-CARRIER`였고 `netplan-wpa-wlo1` 서비스가 `203/EXEC`로 실패해 있었습니다. `203/EXEC`는 systemd가 실행할 프로그램을 찾지 못했다는 뜻이라, 확인해 보니 와이파이 인증을 맡는 `wpasupplicant`가 설치되어 있지 않았습니다. 처음에는 5GHz 대역 문제를 의심했는데 원인은 패키지 부재였습니다.

```bash
dpkg -l wpasupplicant openssh-server | grep ^ii
```

인터넷이 없어서 `apt install`을 쓸 수 없으므로 Mac에서 패키지 파일을 받아 USB로 옮겼습니다. `wpasupplicant`는 `libpcsclite1`에 의존하기 때문에 두 파일이 필요했습니다.

| 파일 | 받는 곳 |
|---|---|
| `wpasupplicant_2.10-21build4_amd64.deb` | `http://archive.ubuntu.com/ubuntu/pool/main/w/wpa/` |
| `libpcsclite1_2.0.3-1build1_amd64.deb` | `http://archive.ubuntu.com/ubuntu/pool/main/p/pcsc-lite/` |

노드에서 USB를 연결해 설치합니다.

```bash
lsblk -f | grep -v loop
sudo mkdir -p /mnt/usb && sudo mount /dev/sda1 /mnt/usb
sudo dpkg -i /mnt/usb/libpcsclite1*.deb /mnt/usb/wpasupplicant*.deb
cd ~ && sudo umount /mnt/usb
sudo netplan apply
ip -br a
```

- `lsblk -f` : USB가 어느 장치로 잡혔는지 확인합니다. 여기서는 `sda1`이었습니다.
- `dpkg -i` : 의존 패키지를 먼저 적어야 순서대로 설치됩니다.
- `cd ~` : USB 안에 있는 채로 `umount`하면 바쁘다고 거부되므로 먼저 빠져나옵니다.

`netplan apply` 뒤 `wlo1`에 `192.168.200.x` 주소가 붙으면서 인터넷이 됐습니다. 이어서 `apt update`를 실행하니 `file:/cdrom ... no longer has a Release file` 경고가 나왔는데, 설치기가 USB를 패키지 저장소로 등록해 둔 줄이 남은 것이라 지우고 나머지 패키지를 설치했습니다.

```bash
sudo sed -i '/cdrom/d' /etc/apt/sources.list /etc/apt/sources.list.d/*.sources /etc/apt/sources.list.d/*.list 2>/dev/null
sudo apt update
sudo apt install -y openssh-server tailscale
sudo tailscale up --ssh
```

tailscale을 설치하기 전에 `could not resolve tailscale.com`이 나온 적이 있는데, curl 문제가 아니라 와이파이가 아직 안 붙어 DNS 조회가 안 되는 상태였습니다. 인터넷만 되면 tailscale은 24.04 저장소에 있어서 `apt install`로 바로 설치됐습니다.

## 7. 결론

설치기가 멈췄다고 디스크가 비어 있는 것은 아니었고, 어디까지 진행됐는지 확인한 뒤 빠진 것만 채우는 쪽이 재설치보다 빨랐습니다. 이번에 확인한 것을 정리하면 다음과 같습니다.

| 확인할 것 | 방법 |
|---|---|
| 지금 라이브 환경인지 디스크인지 | `df -h /`가 `/cow`면 USB. 디스크 작업은 `chroot /target` 안에서 |
| 설치기가 어디까지 했는지 | `/target/etc/hostname`, `/target/etc/passwd`, `/target/boot/efi/EFI` |
| `/target`이 비어 있을 때 | `lsblk -f`로 루트 파티션을 찾아 직접 `mount` |
| 호스트 이름 | `/etc/hostname`이 정함. `/etc/hosts`는 조회용 |
| 부팅 항목 등록이 안 될 때 | `grub-install --removable --no-nvram` |
| 와이파이가 `203/EXEC`로 실패 | `wpasupplicant` 부재. deb를 USB로 옮겨 `dpkg -i` |
| `apt update`의 cdrom 경고 | sources에서 cdrom 줄 삭제 |

설치가 끊기면 `wpasupplicant`나 `openssh-server`처럼 당연히 있을 것 같은 패키지도 빠질 수 있다는 것을 알았기 때문에, 인터넷이 없을 때를 대비해 amd64용 `wpasupplicant`와 `libpcsclite1` deb를 설치 USB에 같이 넣어 두기로 했습니다. 다음 노트북을 설치할 때는 같은 상황이 와도 바로 넘어갈 수 있을 것 같습니다.
