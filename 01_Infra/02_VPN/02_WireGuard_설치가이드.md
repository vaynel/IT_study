# 🔒 WireGuard 설치 및 설정 가이드

## 📋 목차
1. [서버 설정 (Linux)](#1-서버-설정-linux)
2. [클라이언트 설정 (Linux)](#2-클라이언트-설정-linux)
3. [클라이언트 설정 (Windows)](#3-클라이언트-설정-windows)
4. [연결 테스트](#4-연결-테스트)
5. [문제 해결](#5-문제-해결)

---

## 1. 서버 설정 (Linux)

### 1.1 WireGuard 설치

#### Ubuntu/Debian
```bash
# 패키지 업데이트
sudo apt update

# WireGuard 설치
sudo apt install wireguard wireguard-tools -y
```

#### CentOS/RHEL/Rocky Linux
```bash
# EPEL 저장소 추가 (필요한 경우)
sudo yum install epel-release -y

# WireGuard 설치
sudo yum install wireguard-tools -y
```

#### Fedora
```bash
sudo dnf install wireguard-tools -y
```

### 1.2 키 생성

```bash
# 개인키 생성 (서버용)
sudo wg genkey | sudo tee /etc/wireguard/server_private.key | sudo wg pubkey | sudo tee /etc/wireguard/server_public.key

# 권한 설정
sudo chmod 600 /etc/wireguard/server_private.key
sudo chmod 644 /etc/wireguard/server_public.key

# 생성된 공개키 확인
sudo cat /etc/wireguard/server_public.key
```

### 1.3 서버 설정 파일 생성

```bash
sudo nano /etc/wireguard/wg0.conf
```

다음 내용을 입력:

```ini
[Interface]
# 서버의 개인키
PrivateKey = <서버_개인키_내용>

# 서버의 VPN IP 주소 (예: 10.8.0.1)
Address = 10.8.0.1/24

# WireGuard가 리스닝할 포트
ListenPort = 51820

# 서버 재시작 후 자동 시작 설정
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# 클라이언트 추가 예시 (나중에 추가)
# [Peer]
# PublicKey = <클라이언트_공개키>
# AllowedIPs = 10.8.0.2/32
```

**참고**: `eth0`는 서버의 실제 네트워크 인터페이스 이름으로 변경해야 합니다. 확인 방법:
```bash
ip addr show
```

### 1.4 IP 포워딩 활성화

```bash
# 현재 세션에서 활성화
sudo sysctl -w net.ipv4.ip_forward=1

# 영구적으로 활성화
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 1.5 방화벽 설정

#### UFW 사용 시
```bash
sudo ufw allow 51820/udp
sudo ufw allow 22/tcp  # SSH 접속 유지
sudo ufw reload
```

#### firewalld 사용 시
```bash
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --reload
```

#### iptables 직접 사용 시
```bash
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -A OUTPUT -p udp --sport 51820 -j ACCEPT
```

### 1.6 WireGuard 서비스 시작

```bash
# 서비스 시작
sudo systemctl start wg-quick@wg0

# 부팅 시 자동 시작 설정
sudo systemctl enable wg-quick@wg0

# 상태 확인
sudo systemctl status wg-quick@wg0

# 인터페이스 확인
sudo wg show
```

---

## 2. 클라이언트 설정 (Linux)

### 2.1 WireGuard 설치

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install wireguard wireguard-tools -y
```

#### CentOS/RHEL/Rocky Linux
```bash
sudo yum install wireguard-tools -y
```

#### Fedora
```bash
sudo dnf install wireguard-tools -y
```

### 2.2 클라이언트 키 생성

```bash
# 개인키 생성
wg genkey | tee client_private.key | wg pubkey > client_public.key

# 권한 설정
chmod 600 client_private.key

# 공개키 확인 (서버 설정에 필요)
cat client_public.key
```

### 2.3 클라이언트 설정 파일 생성

```bash
sudo nano /etc/wireguard/wg0.conf
```

다음 내용을 입력:

```ini
[Interface]
# 클라이언트의 개인키
PrivateKey = <클라이언트_개인키_내용>

# 클라이언트의 VPN IP 주소 (서버와 다른 주소 사용)
Address = 10.8.0.2/24

# DNS 서버 (선택사항)
DNS = 8.8.8.8

[Peer]
# 서버의 공개키
PublicKey = <서버_공개키_내용>

# 서버의 공용 IP 주소 또는 도메인
Endpoint = <서버_IP_또는_도메인>:51820

# 허용할 IP 범위
# 0.0.0.0/0 = 모든 트래픽을 VPN으로 라우팅 (Full Tunnel)
# 10.8.0.0/24 = VPN 네트워크만 사용 (Split Tunnel)
AllowedIPs = 0.0.0.0/0

# NAT 뒤에 있는 클라이언트를 위한 Keepalive
PersistentKeepalive = 25
```

### 2.4 서버에 클라이언트 추가

서버에서 다음 명령어 실행:

```bash
sudo wg set wg0 peer <클라이언트_공개키> allowed-ips 10.8.0.2/32
```

또는 서버 설정 파일에 직접 추가:

```bash
sudo nano /etc/wireguard/wg0.conf
```

서버 설정 파일에 다음 추가:

```ini
[Peer]
PublicKey = <클라이언트_공개키>
AllowedIPs = 10.8.0.2/32
```

그 후 서버 재시작:

```bash
sudo systemctl restart wg-quick@wg0
```

### 2.5 클라이언트 연결

```bash
# VPN 연결
sudo wg-quick up wg0

# VPN 연결 해제
sudo wg-quick down wg0

# 상태 확인
sudo wg show

# 인터페이스 확인
ip addr show wg0
```

### 2.6 자동 시작 설정 (선택사항)

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

---

## 3. 클라이언트 설정 (Windows)

### 3.1 WireGuard 설치

1. **공식 웹사이트에서 다운로드**
   - https://www.wireguard.com/install/ 접속
   - Windows용 설치 파일 다운로드
   - 또는 직접 다운로드: https://download.wireguard.com/windows-client/wireguard-installer.exe

2. **설치 실행**
   - 다운로드한 설치 파일 실행
   - 설치 마법사 따라 진행

### 3.2 클라이언트 키 생성

#### 방법 1: WireGuard GUI 사용
1. WireGuard 프로그램 실행
2. "Add Tunnel" → "Add empty tunnel..." 클릭
3. 자동으로 키가 생성됨

#### 방법 2: 명령줄 사용 (PowerShell)
```powershell
# WireGuard 설치 경로로 이동
cd "C:\Program Files\WireGuard"

# 키 생성
.\wg.exe genkey | Out-File -Encoding ASCII private.key
.\wg.exe pubkey < private.key | Out-File -Encoding ASCII public.key

# 공개키 확인
Get-Content public.key
```

### 3.3 클라이언트 설정 파일 생성

1. **WireGuard GUI에서 생성**
   - WireGuard 실행
   - "Add Tunnel" → "Add empty tunnel..."
   - 이름 입력 (예: "MyVPN")
   - 설정 편집 창 열기

2. **설정 내용 입력**

```ini
[Interface]
# 클라이언트의 개인키 (GUI에서 자동 생성됨)
PrivateKey = <클라이언트_개인키>

# 클라이언트의 VPN IP 주소
Address = 10.8.0.2/24

# DNS 서버 (선택사항)
DNS = 8.8.8.8

[Peer]
# 서버의 공개키
PublicKey = <서버_공개키>

# 서버의 공용 IP 주소 또는 도메인
Endpoint = <서버_IP_또는_도메인>:51820

# 허용할 IP 범위
AllowedIPs = 0.0.0.0/0

# Keepalive
PersistentKeepalive = 25
```

3. **저장**

### 3.4 서버에 클라이언트 추가

서버에서 클라이언트를 추가해야 합니다 (2.4절 참조).

### 3.5 연결

1. WireGuard GUI에서 생성한 터널 선택
2. "Activate" 버튼 클릭
3. 연결 상태 확인 (녹색 표시 = 연결됨)

---

## 4. 연결 테스트

### 4.1 서버에서 확인

```bash
# 연결된 피어 확인
sudo wg show

# 인터페이스 상태 확인
ip addr show wg0

# 트래픽 통계 확인
sudo wg show wg0 transfer
```

### 4.2 클라이언트에서 확인

#### Linux
```bash
# 연결 상태 확인
sudo wg show

# VPN IP 확인
ip addr show wg0

# 라우팅 테이블 확인
ip route

# 서버와 통신 테스트
ping 10.8.0.1
```

#### Windows
```powershell
# PowerShell에서 확인
Get-NetAdapter | Where-Object {$_.Name -like "*WireGuard*"}

# IP 확인
ipconfig

# 서버와 통신 테스트
ping 10.8.0.1
```

### 4.3 외부 IP 확인

클라이언트에서 외부 IP를 확인하여 VPN이 제대로 작동하는지 확인:

```bash
# Linux
curl ifconfig.me

# Windows (PowerShell)
Invoke-WebRequest -Uri "https://ifconfig.me" -UseBasicParsing | Select-Object -ExpandProperty Content
```

VPN이 연결되면 서버의 공용 IP가 표시되어야 합니다.

---

## 5. 문제 해결

### 5.1 연결이 안 될 때

#### 서버 측 확인사항
```bash
# WireGuard 서비스 상태 확인
sudo systemctl status wg-quick@wg0

# 방화벽 포트 확인
sudo ufw status
# 또는
sudo firewall-cmd --list-all

# IP 포워딩 확인
sysctl net.ipv4.ip_forward

# 로그 확인
sudo journalctl -u wg-quick@wg0 -f
```

#### 클라이언트 측 확인사항
```bash
# 설정 파일 문법 확인
sudo wg-quick strip wg0

# DNS 확인
nslookup <서버_도메인>

# 포트 연결 테스트
nc -u -v <서버_IP> 51820
```

### 5.2 일반적인 문제

#### 문제: "Handshake did not complete"
- **원인**: 방화벽이 UDP 51820 포트를 차단
- **해결**: 서버 방화벽에서 포트 열기

#### 문제: "Connection refused"
- **원인**: 서버의 WireGuard 서비스가 실행되지 않음
- **해결**: `sudo systemctl start wg-quick@wg0`

#### 문제: "No route to host"
- **원인**: 서버에 클라이언트가 등록되지 않음
- **해결**: 서버에 클라이언트 공개키 추가

#### 문제: 인터넷 연결이 안 됨
- **원인**: IP 포워딩이 비활성화됨 또는 NAT 설정 오류
- **해결**: 
  ```bash
  sudo sysctl -w net.ipv4.ip_forward=1
  # PostUp/PostDown의 eth0가 올바른 인터페이스인지 확인
  ```

### 5.3 로그 확인

#### 서버 로그
```bash
# 실시간 로그 확인
sudo journalctl -u wg-quick@wg0 -f

# 최근 로그 확인
sudo journalctl -u wg-quick@wg0 -n 50
```

#### 클라이언트 로그 (Linux)
```bash
# 시스템 로그 확인
sudo journalctl -f | grep wireguard
```

#### 클라이언트 로그 (Windows)
- WireGuard GUI에서 "View Log" 버튼 클릭

### 5.4 설정 파일 재로드

서버 설정 변경 후:

```bash
# 설정 재로드
sudo wg syncconf wg0 <(wg-quick strip wg0)

# 또는 서비스 재시작
sudo systemctl restart wg-quick@wg0
```

---

## 📝 추가 팁

### 여러 클라이언트 추가

각 클라이언트마다:
1. 고유한 VPN IP 주소 할당 (10.8.0.2, 10.8.0.3, 10.8.0.4 등)
2. 서버 설정 파일에 `[Peer]` 섹션 추가
3. 서버 재시작 또는 설정 재로드

### 보안 강화

1. **키 관리**
   - 개인키는 절대 공유하지 않기
   - 정기적으로 키 로테이션

2. **방화벽 규칙**
   - 특정 IP에서만 접속 허용
   ```bash
   sudo ufw allow from <특정_IP> to any port 51820 proto udp
   ```

3. **모니터링**
   - 정기적으로 연결 상태 확인
   - 의심스러운 연결 차단

### 성능 최적화

1. **MTU 크기 조정** (필요한 경우)
   ```ini
   [Interface]
   MTU = 1420
   ```

2. **Keepalive 간격 조정**
   ```ini
   [Peer]
   PersistentKeepalive = 25  # 초 단위
   ```

---

## 📚 참고 자료

- WireGuard 공식 문서: https://www.wireguard.com/
- WireGuard 빠른 시작 가이드: https://www.wireguard.com/quickstart/
- Linux 설정 예제: https://github.com/pirate/wireguard-docs
- Windows 클라이언트: https://www.wireguard.com/install/

---

## ✅ 체크리스트

### 서버 설정
- [ ] WireGuard 설치 완료
- [ ] 서버 키 생성 완료
- [ ] 설정 파일 생성 완료
- [ ] IP 포워딩 활성화
- [ ] 방화벽 포트 열기
- [ ] 서비스 시작 및 자동 시작 설정

### 클라이언트 설정
- [ ] WireGuard 설치 완료
- [ ] 클라이언트 키 생성 완료
- [ ] 설정 파일 생성 완료
- [ ] 서버에 클라이언트 등록 완료
- [ ] 연결 테스트 성공

