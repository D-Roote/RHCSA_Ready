# RHCSA 시험 대비 정리

## 사전 작업

### 1. SSH 루트 로그인 허용

```bash
grep -ir permitrootlogin /etc
echo "PermitRootLogin yes" > /etc/ssh/sshd_config.d/01_RootLogin.conf
systemctl restart sshd
```

> 굳이 필요한가? 가급적 `sudo -i` 사용 권장

### 2. hosts 파일 등록

```bash
vi /etc/hosts
```
```
VM1_IP  node1
VM2_IP  node2
```

---

## node1

### 1. 네트워크 설정

기존 연결 수정 또는 새 연결 생성 후 나머지 비활성화. `nmtui` 사용이 가장 간단.

```bash
hostnamectl hostname node1.network12.example.com
nmcli con mod (NAME) \
  ipv4.method manual \
  ipv4.addresses 172.25.250.10/24 \
  ipv4.gateway 172.25.250.254 \
  ipv4.dns 172.25.250.254
nmcli con up (NAME)
```

확인
```bash
nmcli connection show
ip addr
ip route
```

**Tip.** `nmtui` 사용 시 반드시 **manual**로 설정했는지 재확인할 것 (DHCP로 남아있는 경우가 종종 있음)

---

### 2. 리포지토리 구성 (★★★)

```bash
vi /etc/yum.repos.d/Name.repo
```
```ini
[NAME]
name=NAME
baseurl=URL
enabled=1
gpgcheck=0
```

또는
```bash
dnf config-manager --add-repo=URL
```

확인
```bash
dnf repolist all
```

**Tip.** 리포지토리 파일 위치·구성 형식은 암기 필요

---

### 3. SELinux Debug (★★★)

```bash
sealert -a /var/log/audit/audit.log

ls -lZ /var/www/html          # 파일 보안 컨텍스트 확인
ls -dZ /var/www/html          # 디렉토리 자체의 컨텍스트 확인

# 특정 파일/디렉토리 하위 전체에 컨텍스트 규칙 지정이 필요한 경우
semanage fcontext -a -t httpd_sys_content_t "/var/www/html(/.*)?"
restorecon -Rv /var/www/html

# 포트 등록
semanage port -a -t http_port_t -p tcp 82
semanage port -l | grep http

# boolean 확인/설정
getsebool -a | grep NAME
setsebool -P NAME on

systemctl restart httpd
systemctl enable httpd
```

확인
```bash
semanage port -l | grep http
systemctl status httpd
```

**Tip.** SELinux 관련 명령어는 `--help`로 옵션 확인하는 습관이 실전에서 유용

---

### 4. 사용자 계정 생성

```bash
groupadd manager
useradd natasha -G manager
useradd harry -G manager
useradd sarah -s /sbin/nologin
```

> `nologin` 경로 확인이 필요하면 `which nologin` 실행 → 결과가 `/sbin/nologin`인지 확인

```bash
for usr in natasha harry sarah; do
  echo "$usr:flectrag" | chpasswd
done
```

확인
```bash
id natasha harry sarah
tail /etc/passwd
tail /etc/group
```

---

### 5. cron 작업 구성

**방법 A** — 해당 사용자로 전환 후 편집
```bash
su - harry
crontab -e
```

**방법 B** — root에서 사용자 지정 편집 (더 간단, 권장)
```bash
crontab -e -u harry
```

편집 내용
```
23 14 * * * /usr/bin/echo hello
```

확인
```bash
crontab -l -u harry
```

---

### 6. 파일 권한 설정

```bash
cp /etc/fstab /var/tmp/fstab
chown root:root /var/tmp/fstab
chmod 644 /var/tmp/fstab
setfacl -m u:natasha:rw- /var/tmp/fstab
setfacl -m u:harry:--- /var/tmp/fstab
```

확인
```bash
getfacl /var/tmp/fstab
```

---

### 7. 협업 디렉토리 만들기

```bash
mkdir -p /home/contrib
chown root:manager /home/contrib
chmod 2770 /home/contrib
```

확인
```bash
ls -ld /home/contrib
```

> `2770`: setgid(2) + rwxrwx---(770) → manager 그룹만 접근 가능, 새로 생성되는 파일도 자동으로 manager 그룹 소유가 됨

---

### 8. NTP 구성 (★★★)

```bash
vi /etc/chrony.conf
```
```
server URL iburst
pool URL iburst
```

```bash
systemctl restart chronyd
systemctl enable chronyd
```

확인
```bash
systemctl status chronyd
timedatectl        # System clock synchronized: yes 확인
chronyc sources     # 실제로 해당 서버를 쓰고 있는지 확인
```

---

### 9. 사용자 계정 구성 (UID 지정)

```bash
useradd -u 4332 jean
echo "jean:flectrag" | chpasswd
```

확인
```bash
id jean
```

---

### 10. 파일 찾기

```bash
mkdir -p /root/found
find / -type f -user natasha -exec cp -p {} /root/found \;
```

확인
```bash
ls -l /root/found
```

---

### 11. 문자열 찾기

```bash
grep adm /usr/share/dict/words > /root/lines.txt
```

확인
```bash
cat /root/lines.txt
```

---

### 12. 압축 파일 생성

```bash
tar -cvjf /root/data.tar.bz2 /usr/local
```

확인
```bash
tar tvjf /root/data.tar.bz2
```

> gz = `z` | bzip2 = `j` | xz = `J`

---

### 13. flatpak 리포지토리 구성

```bash
flatpak remote-add --if-not-exists flatdb URL
flatpak search NAME
flatpak install flatdb NAME
```

> `student` 등 실습 계정으로 로그인한 상태에서 진행 (root로 설치 시 사용자 세션에서 인식 안 될 수 있음)

---

### 14. 기본 권한(umask) 설정

```bash
su - daffy
vi ~/.bashrc
```
```
umask 027
```
```bash
source ~/.bashrc
```

확인
```bash
umask
touch test.txt && ls -l test.txt
```

---

### 15. 파일 찾기 스크립트 작성

```bash
vi /usr/local/bin/mysearch
```
```bash
#!/bin/bash
find /usr -type f -size -10M -perm -2000 > /root/myfiles
```
```bash
chmod +x /usr/local/bin/mysearch
```

확인
```bash
/usr/local/bin/mysearch
ls -l /root/myfiles
```

---

### 16. alias 설정

```bash
su - daffy
vi ~/.bashrc
```
```
alias hello='echo "daffy user alias"'
```
```bash
source ~/.bashrc
```

확인
```bash
hello
```

---

### 17. 반복 작업 스크립트 (systemd timer) (★★★★★)

```bash
vi /usr/local/bin/notify.sh
```
```bash
#!/bin/bash
logger "Time for Break!"
```
```bash
chmod +x /usr/local/bin/notify.sh
```

```bash
vi /etc/systemd/system/notify.service
```
```ini
[Unit]
Description=Notify Service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/notify.sh
```

```bash
vi /etc/systemd/system/notify.timer
```
```ini
[Unit]
Description=Notify Timer

[Timer]
OnCalendar=*-*-* *:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl daemon-reload
systemctl enable --now notify.timer
```

확인
```bash
systemctl status notify.timer
systemctl list-timers
```

**OnCalendar 표현 예시**

| 표현 | 의미 |
|---|---|
| `daily` | 매일 자정 |
| `weekly` | 매주 월요일 자정 |
| `monthly` | 매월 1일 자정 |
| `*-*-* 02:00:00` | 매일 오전 2시 |
| `Mon *-*-* 09:00:00` | 매주 월요일 오전 9시 |
| `*-*-01 00:00:00` | 매월 1일 자정 |

---

### 18. sudo 권한 지정

```bash
visudo -f /etc/sudoers.d/sudo
```
```
USER ALL=(ALL) ALL
%GROUP ALL=(ALL) NOPASSWD: ALL
```

확인
```bash
sudo -l -U USER
```

---

## node2

### 1. root 암호 설정

재부팅 → 커널 선택 화면에서 `e` → linux 라인 끝에 `init=/bin/bash` 추가 → `Ctrl+X`(또는 `F10`)로 부팅

```bash
mount -o rw,remount /
echo "root:PASSWD" | chpasswd
touch /.autorelabel
exec /sbin/init
```

확인 : 재부팅 후 root 로그인 시도

---

### 2. 리포지토리 구성

node1과 동일

---

### 3. 논리 볼륨 크기 조정 (★★★)

```bash
vgs        # 볼륨 그룹에 여유 공간 있는지 먼저 확인
lvextend -r -L 250M /dev/VG/LVM
```

> `-r` 옵션을 쓰면 파일시스템 리사이즈까지 자동으로 처리됨. `-r` 없이 확장했다면 파일시스템 종류에 맞춰 별도로 `resize2fs`(ext) 또는 `xfs_growfs`(xfs)를 실행해야 함

확인
```bash
lvs
df -hT /마운트경로
```

---

### 4. 스왑 파티션 추가

```bash
# 1. 여유 공간 확인 (MiB 단위로 정확히 계산하려면 unit 지정 필수)
parted /dev/sdX unit MiB print

# 2. 파티션 생성 (label이 없는 새 디스크라면 먼저 mklabel 필요)
parted -s /dev/sdX mklabel gpt
parted -s /dev/sdX mkpart primary linux-swap 1MiB 961MiB

# 3. swap 플래그 지정 (mkpart에서 linux-swap 타입을 안 썼다면 필요)
parted -s /dev/sdX set 1 swap on

# 4. swap 포맷
mkswap /dev/sdX1

# 5. UUID 확인
blkid /dev/sdX1
```

fstab 등록
```bash
vi /etc/fstab
```
```
UUID=xxxx-xxxx  swap  swap  defaults  0 0
```

```bash
systemctl daemon-reload
swapon -a
```

확인
```bash
swapon --show
```

---

### 5. 논리 볼륨 만들기

**LV 용량 = PE 크기 × 개수 = 32MiB × 20 = 640MiB** (파티션은 이보다 넉넉하게 생성)

```bash
pvs
vgs
```
```bash
pvcreate /dev/sdX
vgcreate wgroup -s 32M /dev/sdX
lvcreate -n wshare -l 20 wgroup
mkfs.ext3 /dev/wgroup/wshare
```
```bash
mkdir -p /mnt/wshare
blkid /dev/wgroup/wshare
```
```bash
vi /etc/fstab
```
```
UUID=xxxx-xxxx  /mnt/wshare  ext3  defaults  0 0
```
```bash
systemctl daemon-reload
mount -a
```

확인
```bash
df -hT /mnt/wshare
vgdisplay wgroup     # PE Size(32MiB) 확인
lvdisplay /dev/wgroup/wshare   # LV Size(640MiB), Current LE(20) 확인
```

---

### 6. autofs 구성

**간접 맵(indirect map) 방식**

```bash
dnf -y install autofs nfs-utils
```
```bash
vi /etc/auto.master.d/indirect.autofs
```
```
/rusers  /etc/auto.indirect
```
```bash
vi /etc/auto.indirect
```
```
*  192.168.10.220:/rusers/&
```

```bash
firewall-cmd --add-service=nfs --permanent
firewall-cmd --reload
setsebool -P use_nfs_home_dirs on
```

```bash
systemctl restart autofs
systemctl enable autofs
```

확인
```bash
systemctl status autofs
ls /rusers/remoteuser1     # 자동 마운트 트리거
df -hT
```

---

### 7. 시스템 조정 (tuned)

```bash
dnf install -y tuned
systemctl enable --now tuned
tuned-adm recommend
tuned-adm profile <추천된_프로필명>
```

확인
```bash
tuned-adm active
```

---

## 종합 체크리스트

- [ ] 모든 서비스 `enable` 여부 재확인 (문제 풀이 후 `systemctl is-enabled`로 일괄 점검)
- [ ] `find -exec` 사용 시 `\;` 이스케이프 습관화
- [ ] `find -perm`의 `MODE` / `-MODE` / `/MODE` 차이 명확히 구분
- [ ] 쉘 변수는 항상 `$변수명`, 명령어 치환은 `$(명령어)` — 혼동 금지
- [ ] `chpasswd`는 표준입력으로 `user:pass`를 받는 명령 (인자로 직접 전달 불가)
- [ ] LVM/파티션 단위 표기는 `M`, `G` 등으로 통일해서 연습
- [ ] 모든 작업 후 재부팅하여 정상 부팅 확인 (재부팅 실패 시 전체 0점 위험)