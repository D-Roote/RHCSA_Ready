# RHCSA 시험 대비 정리

---

## 필요시 사전 작업

### 1. SSH root 로그인 허용

각 노드로 SSH 접속하여 작업하려는 경우에만 적용한다.

```bash
vi /etc/ssh/sshd_config.d/01_RootLogin.conf

# 아래 내용 입력 후 저장
PermitRootLogin yes

systemctl restart sshd
```

---

### 2. 노드 IP 변수 등록

네트워크 설정 이후 IP 주소를 반복 입력하지 않도록 셸 변수로 등록하여 사용할 수 있다.
관리자 권한이 없어 `/etc/hosts` 수정이 불가한 경우 `export` 또는 `~/.bashrc`를 활용한다.

```bash
export NODE1=172.25.250.10
echo $NODE1
# 사용 예: ssh root@$NODE1
```

---

## 사전 설명
문제에서 명확하게 명시하지 않는 조건이 있으므로 정상 작동을 목표로 공부한다.

---

# node1

## 1. 네트워크 설정

- Hostname: `node1.network12.example.com`
- IPv4: `172.25.250.10/24`
- Gateway: `172.25.250.254`
- DNS: `172.25.250.254`

#### 풀이

> 일반적으로 `nmtui`를 사용해서 수행한다.

```bash
# [사전 확인] 연결 프로파일명 확인
nmcli connection show

# [설정 수행]
hostnamectl hostname node1.network12.example.com

nmcli connection modify "Wired Connection 1" \
  ipv4.method manual \
  ipv4.addresses 172.25.250.10/24 \
  ipv4.gateway 172.25.250.254 \
  ipv4.dns 172.25.250.254

nmcli connection up "Wired Connection 1"
```

<details>
<summary><strong>TIP</strong></summary>

- 네트워크 설정은 SSH 환경이 아닌 해당 VM을 직접 실행하여 수행한다.
- `nmcli`로 작업 시 연결 이름에 공백이 포함된 경우 따옴표를 이용하거나 이스케이프 문자를 사용한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
hostnamectl hostname
nmcli connection show "Wired Connection 1"
```

</details>

---

## 2. 리포지토리 구성

문제에서 제시한 BaseOS/AppStream URL을 사용한다.

`BaseOS`: `<BASEOS_URL>`

`AppStream`: `<APPSTREAM_URL>`

#### 풀이

```bash
# [설정 수행]
vi /etc/yum.repos.d/redhat.repo

# 아래 내용 입력 후 저장
[BaseOS]
name=BaseOS
baseurl=<BASEOS_URL>
enabled=1
gpgcheck=0

[AppStream]
name=AppStream
baseurl=<APPSTREAM_URL>
enabled=1
gpgcheck=0

dnf clean all
```

<details>
<summary><strong>TIP</strong></summary>

- `dnf repolist`에서 두 저장소가 enabled 상태로 보이는지 확인한다.  
- 패키지 설치와 연계된 문제가 있으므로 저장소 구성 방법은 반드시 숙지한다.  

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
dnf repolist
```

</details>

---

## 3. SELinux / Firewall 디버그

비표준 HTTP 포트(82)를 사용하도록 설정되어 서비스가 제대로 시작되지 않는 상황으로,  
서비스가 정상 작동하고 자동으로 시작되도록 작업한다.  

#### 풀이

```bash
# [사전 확인] SELinux / 방화벽 확인
semanage port -l | grep http
firewall-cmd --list-all
ls -lZ /var/www/html

# [설정 수행]
# 기본 웹 콘텐츠 경로의 SELinux 라벨을 정책 기본값으로 복구
restorecon -Rv /var/www/html

# 82/tcp를 http_port_t로 등록
semanage port -a -t http_port_t -p tcp 82

# 방화벽에서 82/tcp 허용
firewall-cmd --permanent --add-port=82/tcp
firewall-cmd --reload

# 웹 서비스 활성화
systemctl enable --now httpd
systemctl restart httpd
```

<details>
<summary><strong>TIP</strong></summary>

- 문제에 firewalld가 명시되어 있지 않더라도 외부에서 82/tcp로 접근해야 한다면 방화벽 허용이 필요하다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
# 문제 페이지의 하이퍼링크를 이용해 확인한다.
curl http://172.25.250.10:82/file1
```

</details>

> 시험에서는 `/var/www/html`의 `file1`의 fcontext가 잘못 적용되어 있어 `curl`이 정상 수행되지 않았음.
> `/var/www/html`은 기본 HTTP 콘텐츠 경로이므로 보통 `restorecon`만으로 올바른 타입이 복구된다.
> `semanage fcontext -a`는 사용자 지정 경로처럼 기본 라벨 규칙이 없는 경우에만 추가한다.
> `semanage fcontext -a -t http_sys_content_t "/var/www/html(/.*)?"`

---

## 4. 사용자 계정 생성

조건:

- 그룹 `manager` 생성
- `natasha`, `harry`는 `manager`의 보조 그룹 구성원
- `sarah`는 대화형 로그인 불가
- 세 사용자의 비밀번호는 `rateable`

#### 풀이

```bash
# [설정 수행]
groupadd manager

useradd -G manager natasha
useradd -G manager harry
useradd -s /sbin/nologin sarah

for user in natasha harry sarah; do
  echo "${user}:rateable" | chpasswd
done
```

<details>
<summary><strong>TIP</strong></summary>

- `-G manager`는 보조 그룹 지정이며 기본 그룹을 지정하는 `-g`와 다르다.
- `sarah`는 manager 그룹에 넣지 않고 로그인 셸만 `/sbin/nologin`으로 지정한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
id natasha harry sarah
tail /etc/passwd
```

</details>

---

## 5. cron 작업 구성

`natasha` 사용자로 2분 마다 `logger "natasha text"`를 실행한다.

#### 풀이

```bash
# [설정 수행]

crontab -e -u natasha

# 아래 한 줄을 추가하고 저장
*/2 * * * * logger "natasha text"
```

<details>
<summary><strong>TIP</strong></summary>

- cron 필드 순서는 `분 시 일 월 요일`이다. `12:34`는 `34 12 * * *`이다.
- 특정 주기를 요구하는 경우 `/숫자`를 사용한다. `5분마다`는 `*/5 * * * *`이다.
- 문제에서 요구한 사용자가 `natasha`인지 확인하고 다른 사용자의 crontab에 등록하지 않는다.
- `crontab -l`로 등록 상태를 검증한다.
- 문제가 요구하는 CMD에 따라 `watch` 명령어를 활용해 실제 작동을 확인할 수 있다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
crontab -l -u natasha
```

</details>

---

## 6. 파일 권한 설정

`/etc/fstab`을 `/var/tmp/fstab`으로 복사하고 다음 조건을 만족한다.

- 소유자/그룹: `root:root`
- 누구도 실행 불가
- `natasha`: 읽기/쓰기
- `harry`: 접근 불가
- 기타 사용자: 읽기 가능

#### 풀이

```bash
# [설정 수행]
cp /etc/fstab /var/tmp/fstab
chown root:root /var/tmp/fstab
chmod 0644 /var/tmp/fstab

setfacl -m u:natasha:rw- /var/tmp/fstab
setfacl -m u:harry:--- /var/tmp/fstab
```

<details>
<summary><strong>TIP</strong></summary>

- ACL 설정 후에도 기본 파일 모드에 실행 비트가 남아 있지 않은지 확인한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
ls -l /var/tmp/fstab
getfacl /var/tmp/fstab
```

</details>

---

## 7. 협업 디렉터리 만들기

`/home/contrib`의 그룹 소유권을 `manager`로 지정하고, manager 구성원만 사용할 수 있게 하며 새 파일이 자동으로 `manager` 그룹을 상속하도록 한다.

#### 풀이

```bash
# [설정 수행]
mkdir -p /home/contrib
chown root:manager /home/contrib
chmod 2770 /home/contrib
```

<details>
<summary><strong>TIP</strong></summary>

- `chmod 2770`의 앞자리 `2`를 빠뜨리면 새 파일이 manager 그룹을 자동 상속하지 않는다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
ls -ld /home/contrib
```

</details>

---

## 8. NTP 구성

문제에서 지정한 NTP 서버를 `chrony`에 등록한다.

#### 풀이

```bash
# [설정 수행]
vi /etc/chrony.conf

# 문제에서 지정한 서버를 추가
server 3.kr.pool.ntp.org iburst

systemctl enable --now chronyd
systemctl restart chronyd
```

<details>
<summary><strong>TIP</strong></summary>

- `iburst` 옵션을 누락하지 않도록 확인한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
chronyc sources
```

</details>

---

## 9. UID를 지정한 사용자 계정 구성

UID `4332`의 사용자 `jean`을 만들고 비밀번호를 `rateable`로 설정한다.

#### 풀이

```bash
# [설정 수행]
useradd -u 4332 jean
echo 'jean:rateable' | chpasswd
```

<details>
<summary><strong>TIP</strong></summary>

- `useradd jean`만 실행하면 자동 UID가 할당되므로 반드시 `-u 4332`를 사용한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
id jean
```

</details>

---

## 10. 파일 찾기

`natasha`가 소유한 모든 일반 파일을 찾아 `/root/found`에 복사한다.

#### 풀이

```bash
# [설정 수행]
mkdir -p /root/found
find / -type f -user natasha -exec cp -p {} /root/found \;
```

<details>
<summary><strong>TIP</strong></summary>

- 문제에 파일 옵션 유지를 명시한 경우 `-p` 옵션을 사용한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
ls -l /root/found
```

</details>

---

## 11. 문자열 찾기

`/usr/share/dict/words`에서 `adm` 문자열을 포함하는 모든 행을 원래 순서 그대로 `/root/lines.txt`에 저장한다.

#### 풀이

```bash
# [설정 수행]
grep 'adm' /usr/share/dict/words > /root/lines.txt
```

<details>
<summary><strong>TIP</strong></summary>

- `grep adm`은 대소문자를 구분한다. 문제에 조건이 없다면 임의로 `-i`를 추가하지 않는다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
cat /root/lines.txt
```

</details>

---

## 12. gzip 압축 아카이브 생성

`/usr/local`의 내용을 포함하는 `/root/data.tar.gz`를 생성한다.

#### 풀이

```bash
# [설정 수행]
tar -cvzf /root/data.tar.gz /usr/local
```

<details>
<summary><strong>TIP</strong></summary>

- `gzip`은 `z` `bzip2`는 `j` `xz`는 `J`를 사용한다.
- bzip2, xz를 사용하도록 명시한 경우 해당 패키지가 설치되어 있는지 확인한다.
- 생성 후 `tar -tvzf`로 실제 내용을 확인한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
tar -tvzf /root/data.tar.gz
```

</details>

---

## 13. Flatpak 리포지토리 구성

지정 사용자 계정에 Flatpak remote를 추가하고 문제에서 요구한 애플리케이션을 설치한다.

#### 풀이

```bash
# [설정 수행]
# root에서 패키지가 없다면 먼저 설치
dnf install -y flatpak

# 문제에서 지정한 사용자로 전환
su - <USER>

flatpak --user remote-add --if-not-exists <REMOTE_NAME> <REMOTE_URL>
flatpak --user search <APP_NAME>
flatpak --user install <REMOTE_NAME> <APP_ID> -y
```

<details>
<summary><strong>TIP</strong></summary>

- 특정 사용자에게만 설치하라는 조건이면 root 전역 설치와 `--user` 설치를 혼동하지 않는다.
- remote 이름과 App ID는 문제에서 제시된 값을 그대로 사용한다.
- `flatpak search` 결과가 애매하면 `remote-ls`로 App ID를 다시 확인한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
flatpak --user remotes
flatpak --user list
```

</details>

---

## 14. 기본 권한(umask) 설정

`harry` 사용자가 새로 만드는 파일은 `-rw-r-----` (`640`), 디렉터리는 `drwxr-x---` (`750`)가 되도록 한다.

기본 생성 권한 `666/777`에서 계산하면 `umask 027`이다.

#### 풀이

```bash
# [설정 수행]
su - harry

vi ~/.bashrc

# 아래 내용 추가 후 저장
umask 027

source ~/.bashrc
```

<details>
<summary><strong>TIP</strong></summary>

- 파일 기본 권한은 `666`, 디렉터리는 `777`에서 umask를 적용한다.
- `umask 027`은 반드시 `harry`의 로그인 환경에 설정한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
umask
```

</details>

---

## 15. 파일 찾기 스크립트 작성

`/usr/local/bin/mysearch` 스크립트를 만들고, `/usr` 아래에서 다음 조건의 일반 파일을 찾는다.

- 크기: `10M`보다 작음
- SGID 비트 설정
- 결과: `/root/myfiles`

#### 풀이

```bash
# [설정 수행]
vi /usr/local/bin/mysearch

# 아래 내용 입력 후 저장
#!/bin/bash
find /usr -type f -size -10M -perm -2000 > /root/myfiles

chmod +x /usr/local/bin/mysearch
/usr/local/bin/mysearch
```

<details>
<summary><strong>TIP</strong></summary>

- `-perm -2000`은 SGID 비트가 포함된 파일을 찾는 조건이다.
- 스크립트 파일에 실행 권한을 주는 것을 빠뜨리지 않는다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
/usr/local/bin/mysearch
cat /root/myfiles
```

</details>

---

## 16. alias 설정

`natasha` 사용자가 로그인할 때마다 `hello` 명령으로 `natasha user alias`를 출력하도록 한다.

#### 풀이

```bash
# [설정 수행]
su - natasha

vi ~/.bashrc

# 아래 내용 추가 후 저장
alias hello='echo "natasha user alias"'

source ~/.bashrc
hello
```

<details>
<summary><strong>TIP</strong></summary>

- alias를 현재 셸에서만 선언하면 재로그인 후 사라진다. `~/.bashrc`에 저장한다.
- 따옴표가 꼬이면 출력 문자열이 달라질 수 있으므로 `hello` 실행 결과를 확인한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
hello
```

</details>

---

## 17. 반복 작업 스크립트 / systemd timer

> 시험에서는 `dnf`로 특정 패키지를 설치한 뒤 스크립트를 작성하거나 수정하고, 해당 스크립트를 실행하는 service/timer를 구성하는 형태로 출제될 수 있다.  
> 스크립트 내용과 `OnCalendar` 조건은 달라질 수 있으므로 기본 구조를 숙지한다.

#### 풀이

```bash
# [설정 수행]
vi /usr/local/bin/notify.sh

# 아래 내용 입력 후 저장
#!/bin/bash
logger "Time for Break!"

chmod +x /usr/local/bin/notify.sh

vi /etc/systemd/system/notify.service

# 아래 내용 입력 후 저장
[Unit]
Description=Notify Service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/notify.sh

vi /etc/systemd/system/notify.timer

# 아래 내용 입력 후 저장
[Unit]
Description=Notify Timer

[Timer]
OnCalendar=*-*-* *:00:00
Persistent=true

[Install]
WantedBy=timers.target

systemctl daemon-reload
systemctl enable --now notify.timer
```

<details>
<summary><strong>TIP</strong></summary>

- service와 timer 파일명, `ExecStart` 스크립트 경로를 서로 다르게 적지 않는다.
- unit 파일 생성 후 `systemctl daemon-reload`를 반드시 수행한다.
- `OnCalendar` 조건은 회차마다 달라질 수 있으므로 문제 시간을 정확히 변환한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
systemctl list-timers --all | grep notify
```

</details>

---

## 18. sudo 권한 지정

특정 사용자/그룹이 sudo를 사용할 수 있도록 설정한다.

#### 풀이

```bash
# [설정 수행]
vi /etc/sudoers.d/harry

# 아래 내용 입력 후 저장
harry ALL=(ALL) NOPASSWD: ALL

chmod 0440 /etc/sudoers.d/harry
visudo -c
```

<details>
<summary><strong>TIP</strong></summary>

- 문제에서 `/etc/sudoers`를 수정하지 않도록 명시할 수 있으니 문제를 잘 확인한다.
- `/etc/sudoers.d/harry` 문법 오류는 sudo 사용에 영향을 줄 수 있으므로 `visudo -c`로 검사한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
su - harry -c 'sudo -n id'
```

</details>

---

# node2

## 1. root 암호 재설정

부팅 메뉴에서 대상 커널을 선택하고 `e`를 눌러 `linux` 행 끝에 `init=/bin/bash`를 추가한 뒤 `Ctrl+X`로 부팅한다.

#### 풀이

```bash
# [설정 수행]
mount -o rw,remount /
echo 'root:rateable' | chpasswd
touch /.autorelabel
exec /sbin/init
```

<details>
<summary><strong>TIP</strong></summary>

- 커널 편집 시 다른 노드나 다른 부팅 항목을 수정하지 않도록 확인한다.
- 루트 파일시스템을 `rw`로 remount하지 않으면 암호 변경이 저장되지 않을 수 있다.
- `/.autorelabel` 철자를 틀리지 않는다.
- relabel 과정은 정상이어도 다음 부팅 시간이 길어질 수 있다.

</details>

<details>
<summary><strong>수행 후 확인</strong></summary>

재부팅 및 SELinux relabel 완료 후 **root로 로그인되는지만 확인**한다.

</details>

> `touch /.autorelabel` 철자를 특히 주의한다. SELinux relabel이 정상적으로 수행되면 다음 부팅이 평소보다 오래 걸릴 수 있다.

---

## 2. 리포지토리 구성

node1과 같은 방식으로 문제에서 지정한 BaseOS/AppStream 저장소를 구성한다.

---

## 3. 논리 볼륨 크기 조정

예시 조건: `/dev/myvol/vo` 논리 볼륨과 파일시스템의 최종 크기를 약 `250MiB`로 확장한다.

#### 풀이

```bash
# [사전 확인] VG 여유 공간과 현재 LV 크기 확인
vgs
lvs

# [설정 수행]
lvextend -r -L 250M /dev/myvol/vo
```

<details>
<summary><strong>TIP</strong></summary>

- VG의 Free PE가 충분한지 확인하지 않고 확장하면 실패할 수 있다.
- `-r`을 빠뜨리면 LV만 늘고 파일시스템 크기는 그대로일 수 있다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
df -hT /mnt
```

</details>

> `-r`은 LV 확장 후 파일시스템까지 함께 리사이즈한다.  
> `-L 250M`은 **최종 크기를 250M로 지정**하는 형식이다. `+250M`과 혼동하지 않는다.

---

## 4. 스왑 파티션 추가

> **주의:** 이미 사용 중인 디스크에서 `parted ... mklabel gpt`를 실행하면 기존 파티션 테이블을 망가뜨릴 수 있다.  
> 먼저 현재 파티션 테이블과 **Free Space**를 확인하고, 빈 공간에 새 파티션만 만든다.

#### 풀이

```bash
# [사전 확인] 기존 파티션과 빈 공간 확인
parted /dev/vdb unit MiB print free

# [설정 수행]
parted /dev/vdb
# (parted) mkpart primary linux-swap <START_MiB> <END_MiB>
# (parted) quit

partprobe /dev/vdb

mkswap /dev/vdb<N>
blkid /dev/vdb<N>

vi /etc/fstab

# 아래 한 줄 추가
UUID=<SWAP_UUID>  none  swap  defaults  0 0

systemctl daemon-reload
swapon -a
```

<details>
<summary><strong>TIP</strong></summary>

- 사용 중인 디스크에 `mklabel gpt`를 실행하면 기존 파티션 테이블을 손상시킬 수 있으므로 사용하지 않는다.
- MiB 문제에서는 `parted ... unit MiB print free`로 빈 공간을 정확히 확인한다.
- `fstab` 등록 후 `swapon -a`와 `swapon --show`로 활성화를 확인한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
swapon --show
```

</details>

---

## 5. 새 논리 볼륨 만들기

조건:

- VG: `wgroup`
- VG PE 크기: `16MiB`
- LV: `wshare`
- LV 크기: `60 extents`
- 실제 LV 크기: `16MiB × 60 = 960MiB`
- 파일시스템: `ext3`
- 마운트 지점: `/mnt/wshare`
- 부팅 시 자동 마운트

#### 풀이

```bash
# [사전 확인] 기존 파티션과 빈 공간 확인
parted /dev/vdb unit MiB print free

# [설정 수행]
# 먼저 /dev/vdb의 빈 공간을 확인한 뒤 960MiB보다 넉넉한 LVM 파티션 생성
parted /dev/vdb
# (parted) unit MiB
# (parted) print free
# (parted) mkpart primary <START_MiB> <END_MiB>
# (parted) set <PARTITION_NUMBER> lvm on
# (parted) quit

partprobe /dev/vdb

pvcreate /dev/vdb<N>
vgcreate -s 16M wgroup /dev/vdb<N>
lvcreate -n wshare -l 60 wgroup

mkfs.ext3 /dev/wgroup/wshare
mkdir -p /mnt/wshare

blkid /dev/wgroup/wshare
vi /etc/fstab

# 아래 한 줄 추가
UUID=<LV_UUID>  /mnt/wshare  ext3  defaults  0 0

systemctl daemon-reload
mount -a
```

<details>
<summary><strong>TIP</strong></summary>

- 960MiB LV를 만들려면 PV 파티션은 960MiB보다 여유 있게 잡는다.
- `vgcreate -s 16M`에서 PE 크기를 빠뜨리지 않는다.
- `lvcreate -l 60`의 소문자 `-l`은 extents 수이며 대문자 `-L`과 다르다.
- `fstab` 오타는 재부팅 실패 원인이 될 수 있으므로 `mount -a`로 먼저 검증한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
vgdisplay wgroup
lvdisplay /dev/wgroup/wshare
df -hT /mnt/wshare
```

</details>

> `vgdisplay wgroup`에서 `PE Size`가 `16.00 MiB`, `lvdisplay`에서 `Current LE`가 `60`인지 확인한다.

---

## 6. autofs 구성

원격 NFS를 필요할 때 자동 마운트하도록 구성한다.

- **간접 맵(Indirect map)**: 기준 디렉터리 아래에 상대 경로로 마운트한다.
- **직접 맵(Direct map)**: 파일시스템의 특정 절대 경로에 바로 마운트한다.
- 실제 시험에서는 문제의 경로 형태를 보고 **둘 중 하나의 방식만 선택**한다.

| 구분 | Master map | Map의 key | 예시 |
| --- | --- | --- | --- |
| 간접 맵 | `/rusers /etc/auto.indirect` | 상대 경로 (`*`, `remoteuser1`) | `/rusers/remoteuser1` |
| 직접 맵 | `/- /etc/auto.direct` | 절대 경로 (`/rusers/remoteuser1`) | `/rusers/remoteuser1` |

### 방식 1. 간접 맵(Indirect map)

예시 조건:

- NFS 서버: `192.168.10.220`
- Export: `/rusers`
- 로컬 기준 경로: `/rusers`
- 사용자별 경로: `/rusers/remoteuser1`

#### 풀이

```bash
# [설정 수행]
dnf install -y autofs nfs-utils

vi /etc/auto.master.d/indirect.autofs

# 아래 내용 입력 후 저장
/rusers  /etc/auto.indirect

vi /etc/auto.indirect

# 아래 내용 입력 후 저장
*  192.168.10.220:/rusers/&

# NFS 홈 디렉터리를 사용하는 경우 SELinux 허용
setsebool -P use_nfs_home_dirs on

systemctl enable --now autofs
systemctl restart autofs
```

<details>
<summary><strong>TIP - 간접 맵</strong></summary>

- master map의 `/rusers`가 **기준 디렉터리**가 된다.
- `/etc/auto.indirect`의 key는 절대 경로가 아니라 `remoteuser1` 또는 `*` 같은 **상대 key**를 사용한다.
- wildcard `*`를 사용하면 `&`에는 접근한 key가 그대로 치환된다.
  - `/rusers/remoteuser1` 접근 → `192.168.10.220:/rusers/remoteuser1`
- `/rusers/remoteuser1`에 실제 접근해야 autofs 마운트가 트리거된다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
su - remoteuser1 -c 'touch /rusers/remoteuser1/autofs_test'
```

</details>

### 방식 2. 직접 맵(Direct map)

특정 절대 경로를 NFS export와 직접 연결해야 하는 경우 사용한다.

예시:

- 로컬 경로: `/rusers/remoteuser1`
- NFS 경로: `192.168.10.220:/rusers/remoteuser1`

#### 풀이

```bash
# [설정 수행]
dnf install -y autofs nfs-utils

vi /etc/auto.master.d/direct.autofs

# 아래 내용 입력 후 저장
/-  /etc/auto.direct

vi /etc/auto.direct

# 아래 내용 입력 후 저장
/rusers/remoteuser1  192.168.10.220:/rusers/remoteuser1

# NFS 홈 디렉터리를 사용하는 경우 SELinux 허용
setsebool -P use_nfs_home_dirs on

systemctl enable --now autofs
systemctl restart autofs
```

<details>
<summary><strong>TIP - 직접 맵</strong></summary>

- 직접 맵의 master map mount point는 `/-`를 사용한다.
- `/etc/auto.direct`의 key는 반드시 `/rusers/remoteuser1`처럼 **절대 경로**를 사용한다.

</details>

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
su - remoteuser1 -c 'touch /rusers/remoteuser1/autofs_test'
```

</details>

> **암기 포인트:**  
> 간접 맵 = `기준 경로 + 상대 key`  
> 직접 맵 = `/- + 절대 경로 key`

---

## 7. 시스템 조정 (tuned)

시스템에서 권장하는 tuned 프로필을 확인하고 해당 프로필을 활성화한다.

#### 풀이

```bash
# [설정 수행]
dnf install -y tuned
systemctl enable --now tuned

tuned-adm recommend
tuned-adm profile <RECOMMENDED_PROFILE>
```

<details>
<summary><strong>수행 후 확인 명령어</strong></summary>

```bash
tuned-adm active
```

</details>

---

# 최종 점검

각 문제에서 바로 검증을 완료했다는 전제로, 모든 노드를 재부팅한 후 정상 부팅 여부와 필요한 서비스의 자동 시작 상태를 확인한다.
