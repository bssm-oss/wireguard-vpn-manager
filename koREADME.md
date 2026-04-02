# vpn - 간단 WireGuard 유저 관리 스크립트

이 스크립트는 단일 WireGuard 서버에서 **VPN 유저 관리 + 접근 제어(iptables)**를 편하게 하기 위한 Bash 스크립트입니다.

기능:

- WireGuard 서버 초기 세팅 (선택적, init)
- VPN 유저 추가/삭제
- 클라이언트 `.conf` 파일 생성
- 유저별 LAN 접근 제어 (full / single / multi, iptables 기반)
- WireGuard 재시작
- 유저 리스트 및 클라이언트 설정 출력

---

## 사용 전에 (Before Starting)

다음 조건이 충족되어야 합니다:

1. **OS / 권한**
   - Ubuntu / Debian 계열 서버.
   - `sudo` 권한 보유 (root 또는 sudoer).

2. **네트워크**
   - 서버에서 outbound 인터넷 접속 가능 (HTTPS).
   - 공유기/방화벽에서 **UDP 51820 포트가 이 서버 IP로 포트포워딩** 되어 있어야 함.
   - NAT 환경이면, 클라이언트는 포워딩된 공인 IP로 접속하게 됨.

3. **기존 WireGuard 설정**
   - 이미 `wg0`를 다른 용도로 쓰고 있다면, `./vpn init` 실행 전에 스크립트 내용을 꼭 확인할 것.
   - `init`는 `/etc/wireguard/wg0.conf`가 없으면 새로 만들고, 있을 경우에는 해당 파일을 기준으로 동작합니다.

4. **패키지**
   - `init` 사용 시:
     - `apt` 사용 가능해야 하며, 스크립트가 `wireguard`, `wireguard-tools`, `curl`을 설치합니다.
   - 이미 WireGuard가 설치된 환경에서 `start`만 쓸 수도 있습니다.

---

## 빠른 시작 (v1, 새 서버 기준)

1. 리포지토리 클론 및 권한 설정:

   ```bash
   git clone <your_repo_url> vpn-manager
   cd vpn-manager
   chmod +x vpn
   ```

2. 풀 초기화 실행 (WireGuard + 기본 wg0.conf + iptables ACL + 스크립트 설치):

   ```bash
   ./vpn init
   ```

   이 명령은 다음을 수행합니다:

   - `wireguard`, `wireguard-tools`, `curl` 설치
   - `/etc/wireguard/wg0.conf` 생성:
     - 인터페이스 `wg0`
     - 주소 `10.0.0.1/24`
     - 포트 `51820`
     - 기본 NAT/포워드 iptables 규칙
   - `/home/wireguard/clients`, `/home/wireguard/keys`, `/home/wireguard/users.txt` 생성
   - `WG_ACCESS` iptables 체인 생성 및 `FORWARD -i wg0`에 연결
   - `/usr/local/bin/vpn`에 스크립트 설치

3. 사용법 확인:

   ```bash
   vpn guide
   ```

---

## 명령어

### 1. `init` (처음 서버 세팅용, v1에만 존재)

```bash
./vpn init
```

- 새 서버에서 **처음 한 번만** 실행하는 것을 가정.
- WireGuard 및 기본 설정, iptables ACL, `/usr/local/bin/vpn`까지 한 번에 설치.

### 2. `start` (v1에만 존재)

```bash
./vpn start
```

- 의존성 설치 없이, **현재 디렉터리의 vpn 스크립트만 `/usr/local/bin/vpn`으로 복사**.
- 이미 WireGuard가 설치된 서버에서 스크립트만 배포하고 싶을 때 사용.

---

### 3. 유저 추가

```bash
vpn add <username> <full|single|multi> [target]
```

- `full`  
  - 다음 접근 허용:
    - `10.0.0.0/24` (VPN 대역)
    - `192.168.219.0/24` (예시 집 LAN 대역)
  - 예:
    ```bash
    vpn add alice full
    ```

- `single`  
  - VPN + 특정 한 개의 내부 IP만 허용.
  - 예:
    ```bash
    vpn add bob single 192.168.219.100
    ```

- `multi`  
  - VPN + 여러 개의 내부 IP 허용 (콤마로 구분).
  - 예:
    ```bash
    vpn add charlie multi 192.168.219.100,192.168.219.101
    ```

각 `add`는:

- `/etc/wireguard/wg0.conf`에 `[Peer]` 블록 추가
- `10.0.0.X/32` VPN IP 할당
- `/home/wireguard/keys`에 키 생성
- `/home/wireguard/clients/<username>.conf` 클라이언트 설정 생성
- `/home/wireguard/users.txt`에 로그 기록
- `WG_ACCESS` 체인에 해당 유저용 iptables 규칙 추가 (TYPE/TARGET에 따라)

> 유저 추가 후에는 반드시 **WireGuard 재시작**:

```bash
vpn restart
```

---

### 4. 유저 삭제

```bash
vpn remove <username>
```

- `wg0`에서 peer 제거
- `/etc/wireguard/wg0.conf`에서 해당 `[Peer]` 블록 제거
- `keys`, `clients`, `users.txt`에서 해당 유저 데이터 제거
- `WG_ACCESS`에서 해당 VPN IP 관련 iptables 규칙 제거

삭제 후:

```bash
vpn restart
```

---

### 5. 유저 리스트

```bash
vpn list
```

- `wg0.conf` + `users.txt` 기준으로:
  - USERNAME
  - VPN_IP (AllowedIPs)
  - TYPE (full/single/multi)
  - TARGET (허용된 내부 IP들)

을 출력합니다.

---

### 6. WireGuard 재시작

```bash
vpn restart
```

- `systemctl restart wg-quick@wg0` 실행.
- 상태의 앞부분을 보여줍니다.

---

### 7. 클라이언트 설정 출력

```bash
vpn get <username>
```

- `/home/wireguard/clients/<username>.conf` 내용을 출력.
- 이 파일을 그대로 WireGuard 클라이언트 앱에 Import 하면 됩니다.

---

## 접근 제어 방식

- 서버 `wg0.conf`의 각 `[Peer]`:
  - `AllowedIPs = 10.0.0.X/32`  
  → 클라이언트 식별용 (각 peer의 VPN IP).

- 클라이언트 설정:
  - `AllowedIPs = 10.0.0.0/24,192.168.219.0/24`  
  → 어떤 목적지 트래픽을 터널로 보낼지 (라우팅)만 정의.

- 실제 접근 제어:
  - 서버 iptables `WG_ACCESS` 체인에서 강제:
    - `full`:
      - `-s 10.0.0.X/32 -d 192.168.219.0/24 -j ACCEPT`
      - 그 다음 `-s 10.0.0.X/32 -j DROP`
    - `single`:
      - `-s 10.0.0.X/32 -d 192.168.219.Y/32 -j ACCEPT`
      - 그 다음 `-s 10.0.0.X/32 -j DROP`
    - `multi`:
      - 여러 `-d IP/32 -j ACCEPT` 후 `-s 10.0.0.X/32 -j DROP`

→ 클라이언트가 자기 `.conf`를 수정해도, 서버에서 어떤 내부 IP까지 갈 수 있는지는 iptables가 최종적으로 결정합니다.

---

## 주의 사항

- 이 스크립트는 다음과 같은 환경을 가정합니다:
  - 인터페이스 이름: `wg0`
  - VPN 대역: `10.0.0.0/24`
  - 내부 LAN 예시: `192.168.219.0/24` (필요하면 스크립트에서 수정)
- 실제 서비스 환경에 적용하기 전에, 반드시 스크립트 내용을 검토하고 테스트 환경에서 충분히 검증하는 것을 권장합니다.