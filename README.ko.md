# vpn - 간단 WireGuard 유저 관리 스크립트

태그 기반의 유연한 접근 제어를 제공하는 WireGuard 서버 유저 관리 Bash 스크립트입니다.

기능:

- WireGuard 서버 초기 세팅
- 태그 기반 접근 제어를 통한 VPN 유저 추가/삭제
- 디렉터리 지원과 함께 태그 기반 IP 조직화
- 유연한 접근 규칙 (전체 네트워크, 단일 IP, 다중 IP, 태그 참조, 디렉터리 와일드카드)
- ipset을 사용한 자동 iptables ACL 관리
- 유저 및 태그 설명 명령어

---

## 사용 전에

다음 조건이 충족되어야 합니다:

1. **OS / 권한**
   - Ubuntu / Debian 계열 서버
   - `sudo` 권한 보유 (root 또는 sudoer)

2. **네트워크**
   - 서버에서 outbound 인터넷 접속 가능 (HTTPS)
   - 공유기/방화벽에서 **UDP 51820 포트가 이 서버 IP로 포트포워딩** 되어 있어야 함
   - NAT 환경이면, 클라이언트는 포워딩된 공인 IP로 접속하게 됨

3. **기존 WireGuard 설정**
   - 이미 `wg0`를 다른 용도로 쓰고 있다면, `./vpn init` 실행 전에 스크립트 내용을 꼭 확인할 것

4. **패키지**
   - `init` 명령어가 자동으로 설치: `wireguard`, `wireguard-tools`, `curl`, `iptables`, `ipset`, `iptables-persistent`
   - 이미 WireGuard가 설치된 경우 `start` 명령어로 스크립트만 설치 가능

---

## 빠른 시작

1. 리포지토리 클론 및 권한 설정:

   ```bash
   git clone <your_repo_url> vpn-manager
   cd vpn-manager
   chmod +x vpn
   ```

2. 풀 초기화 실행 (WireGuard + 의존성 + iptables ACL + 스크립트 설치):

   ```bash
   ./vpn init
   ```

   이 명령은 다음을 수행합니다:

   - `wireguard`, `wireguard-tools`, `curl`, `iptables`, `ipset`, `iptables-persistent` 설치
   - 디렉터리 구조 생성:
     - `/home/wireguard/clients` - 클라이언트 설정 파일
     - `/home/wireguard/keys` - 개인키/공개키
     - `/home/wireguard/users.txt` - 유저 레지스트리
     - `/home/wireguard/tags` - 태그 정의
   - `WG_ACCESS` iptables 체인 생성 및 `FORWARD -i wg0`에 연결
   - `/usr/local/bin/vpn`에 스크립트 설치

3. 사용법 확인:

   ```bash
   vpn guide
   ```

---

## 명령어

### 1. 시스템 초기화

```bash
./vpn init
```

- 새 서버에서 **처음 한 번만** 실행
- 모든 의존성 설치 (WireGuard, iptables, ipset)
- 디렉터리 구조 생성
- iptables ACL 체인 설정
- `/usr/local/bin/vpn`에 스크립트 설치

### 2. 스크립트만 설치

```bash
./vpn start
```

- 스크립트만 `/usr/local/bin/vpn`으로 복사
- 이미 WireGuard가 설치 및 설정된 경우 사용

### 3. 유저 추가

```bash
vpn add <username> <target>
```

`target` 파라미터는 다양한 형식 지원:

- **`full`** - 전체 LAN 서브넷 접근 (192.168.219.0/24)
  ```bash
  vpn add alice full
  ```

- **단일 IP** - 특정 IP 하나만 접근
  ```bash
  vpn add bob 192.168.0.10
  ```

- **다중 IP** - 콤마로 구분된 여러 IP
  ```bash
  vpn add charlie 192.168.0.10,192.168.0.11
  ```

- **태그 참조** - 미리 정의된 태그 사용
  ```bash
  vpn add david example/db
  ```

- **디렉터리 와일드카드** - 디렉터리 내 모든 태그
  ```bash
  vpn add eve example/
  ```

각 `add` 명령어는:
- `/etc/wireguard/wg0.conf`에 `[Peer]` 블록 추가
- `10.0.0.X/32` VPN IP 할당
- `/home/wireguard/keys`에 키 생성
- `/home/wireguard/users.txt`에 로그 기록
- ipset을 사용한 효율적인 필터링을 위한 iptables 규칙 생성
- 규칙 즉시 적용 (재시작 불필요)

---

### 4. 유저 삭제

```bash
vpn remove <username>
```

다음을 수행합니다:
- 실행 중인 `wg0`에서 peer 제거
- 키와 설정 삭제
- `users.txt`에서 로그 항목 제거
- 연관된 iptables/ipset 규칙 제거

---

### 5. 유저 리스트

```bash
vpn list
```

다음 정보를 포함한 모든 유저 표시:
- 유저명
- VPN IP (10.0.0.X)
- 접근 규칙 정의
- 생성 날짜

---

### 6. 태그 관리

IP 주소를 조직화하기 위한 태그 생성:

```bash
# 태그 추가
vpn tag add <ip> <tag_name>

# 예시: 데이터베이스 서버 태그 생성
vpn tag add 192.168.0.100 production/db

# 태그 제거
vpn tag remove <tag_name>
```

태그는 디렉터리 구조 지원:
- `production/db`
- `production/web`
- `development/api`

유저 추가 시 디렉터리 와일드카드 사용: `vpn add user production/`

---

### 7. 설명 명령어

유저, 태그 또는 디렉터리에 대한 상세 정보 확인:

```bash
# 유저의 접근 권한 설명
vpn explain <username>

# 태그가 가리키는 IP 표시
vpn explain <tag_name>

# 디렉터리 내 모든 태그 나열
vpn explain <directory/>
```

---

### 8. 가이드 표시

```bash
vpn guide
```

사용 가이드 및 사용 가능한 명령어 표시

---

## 접근 제어 모델

이 스크립트는 계층화된 보안 접근 방식을 사용합니다:

### WireGuard 계층
- 서버 `wg0.conf`의 `[Peer]` 블록은 `AllowedIPs = 10.0.0.X/32`를 사용해 각 클라이언트 식별
- 이것은 LAN 접근을 **직접 제어하지 않습니다**

### iptables + ipset 계층
- 모든 접근 제어는 `WG_ACCESS` iptables 체인을 통해 강제됨
- `full` 접근: 전체 LAN 서브넷을 허용하는 직접 iptables 규칙
- 특정 IP 접근: 효율적인 다중 IP 매칭을 위해 ipset 사용
  - 각 유저는 전용 ipset 보유: `user_10_0_0_X`
  - 태그/디렉터리 참조는 IP로 해석되어 set에 추가됨
  - iptables 규칙이 ipset과 매칭
- 최종 DROP 규칙으로 기본 거부 보장

### 보안 이점
- 클라이언트가 설정 파일을 수정해도 서버가 실제 접근 제어
- ipset은 대규모 IP 리스트에 대해 O(1) 조회 제공
- 태그 시스템으로 중앙화된 IP 관리 가능
- 디렉터리 구조로 논리적 그룹화 가능

---

## 사용 예시

### 예시 1: 데이터베이스 접근

```bash
# 데이터베이스 서버용 태그 생성
vpn tag add 192.168.0.100 production/db-primary
vpn tag add 192.168.0.101 production/db-replica

# DBA에게 모든 DB 접근 권한 부여
vpn add dba production/

# 개발자에게 제한된 접근 권한 부여
vpn add dev1 production/db-replica
```

### 예시 2: 다중 환경 접근

```bash
# 환경별 태그 설정
vpn tag add 192.168.0.10 dev/api
vpn tag add 192.168.0.11 dev/web
vpn tag add 192.168.0.20 staging/api
vpn tag add 192.168.0.21 staging/web

# 개발자는 전체 개발 환경 접근
vpn add developer dev/

# QA는 개발 + 스테이징 접근
vpn add qa dev/,staging/
```

### 예시 3: 혼합 접근

```bash
# 특정 IP + 태그 참조 + 디렉터리
vpn add admin 192.168.0.1,production/db,monitoring/
```

---

## 주의 사항

- **VPN 서브넷**: `10.0.0.0/24` (스크립트에서 설정 가능)
- **기본 LAN 서브넷**: `192.168.219.0/24` (필요시 `add_acl_rules` 함수에서 변경)
- **인터페이스**: `wg0`만 지원
- **태그 저장소**: `/home/wireguard/tags/` 디렉터리 구조 지원
- 실제 서비스 환경에 적용하기 전에 반드시 스크립트를 검토하고 테스트하세요
