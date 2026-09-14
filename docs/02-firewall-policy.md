[← README로 돌아가기](../README.md)

# 02. 방화벽 정책 설계

## 결론

방화벽의 보안 수준은 규칙의 개수가 아니라 **기본 정책과 평가 순서**가 결정합니다. "무엇을 막을까"가 아니라 "무엇만 허용할까"에서 출발했습니다.

## 설계 원칙 5가지

### 1. Default Deny

명시적으로 허용하지 않은 트래픽은 전부 차단합니다. 허용 목록을 관리하는 편이, 금지 목록을 끝없이 늘리는 것보다 누락 위험이 작습니다. 각 탭 맨 아래에 `Block any → any (Log)`를 두어 이 원칙을 강제했습니다.

### 2. First-match를 고려한 순서

pfSense는 규칙을 위에서 아래로 평가하고 첫 매치에서 정지합니다. 그래서 Pass 규칙을 위에, Block 규칙을 아래에 배치해야 의도한 허용이 광범위한 차단에 먼저 걸리지 않습니다. 규칙의 순서 자체가 논리입니다.

### 3. Alias로 추상화

IP를 규칙에 직접 박으면, 주소가 바뀔 때마다 규칙을 전부 고쳐야 하고 규칙을 읽어도 의미가 드러나지 않습니다. 호스트를 Alias로 정의해 규칙이 IP가 아니라 역할로 읽히게 했습니다.

| Alias | 값 | 의미 |
|---|---|---|
| `MON_SERVER` | 192.168.100.10 | 관제 서버(ELK) |
| `DMZ_WEB` | 192.168.200.10 | DMZ 웹서버(DVWA) |
| `DMZ_WIN_VICTIM` | 192.168.200.20 | DMZ 피해자 Windows |

### 4. 모든 Block에 Log

차단 자체보다 "무엇이 차단됐는지"가 관제 자산입니다. 차단 로그가 쌓여야 스캔·탐색 시도를 사후에 분석할 수 있습니다. 모든 Block 규칙에 Log를 켰습니다.

### 5. 롤백 안전망 유지

정책을 조이는 과정에서 관리 경로까지 잠글 수 있습니다. 초기 Allow-All 규칙을 삭제하지 않고 **Disable로 보존**해, 잠금 사고가 나면 즉시 되돌릴 수 있게 했습니다.

## 존 간 접근 정책

```
WAN  → DMZ    : 웹 서비스 포트만 허용
DMZ  → 관제망 : 로그 전송 포트(5044)만 허용
DMZ  → 내부망 : 전면 차단 (Lateral Movement 방지)
전체 → 관제망 : 로그 트래픽만 단방향
```

핵심은 `DMZ → 내부망` 전면 차단입니다. DMZ 웹서버가 침해되더라도, 거기서 관제망·내부망으로 옆걸음질(Lateral Movement)하는 경로를 막는 것이 존 분리의 실질적 목적입니다.

## 규칙 12개와 근거

### WAN 탭 (3개)

| # | Action | Proto | Source → Destination | Port | 근거 |
|---|---|---|---|---|---|
| 1 | Pass | TCP | any → DMZ_WEB | 443 | 외부에 노출하는 것은 웹 서비스뿐 |
| 2 | Pass | TCP | any → DMZ_WEB | * | 실습용 웹 접근 확인 경로 |
| 3 | Block | any | any → any | * (Log) | Default Deny, 스캔 시도 기록 |

WAN에서 들어오는 트래픽은 DMZ 웹서버로 향하는 웹 세션만 통과합니다. 관제망·피해자 Windows로 직접 향하는 경로는 열지 않았습니다.

### DMZ 탭 (4개)

| # | Action | Proto | Source → Destination | Port | 근거 |
|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | DMZ net → MON_SERVER | 5044 | 로그 전송(Filebeat beats)만 허용 |
| 2 | Pass | ICMP | DMZ net → DMZ address | * | 게이트웨이 도달성 점검용 |
| 3 | Block | any | DMZ net → LAN net | * (Log) | Lateral Movement 차단 |
| 4 | Block | any | any → any | * (Log) | Default Deny |

DMZ가 관제망으로 낼 수 있는 것은 로그 5044 하나뿐입니다. 3번 규칙으로 관제망 대역 전체로 향하는 나머지 경로를 명시적으로 차단하고 로그를 남깁니다. Source는 반드시 `DMZ net`(대역 전체)이어야 하며, `DMZ address`(/32)는 인터페이스 IP 하나만 뜻해 서버 트래픽이 매칭되지 않습니다(트러블슈팅 ②).

### LAN(관제망) 탭 (5개, Anti-Lockout Rule 유지)

| # | Action | Proto | Source → Destination | Port | 근거 |
|---|---|---|---|---|---|
| 1 | Pass | TCP | LAN net → This Firewall (self) | 443 | 관제망에서의 방화벽 관리(HTTPS) |
| 2 | Pass | ICMP | LAN net → This Firewall (self) | * | 방화벽 도달성 점검 |
| 3 | Pass | TCP | MON_SERVER → any | 443 | 관제 서버 외부 갱신 등 |
| 4 | Pass | UDP | MON_SERVER → any | 53 | DNS 질의 |
| 5 | Block | any | any → any | * (Log) | Default Deny |

1·2번의 `This Firewall (self)` 허용을 빠뜨리면 최소 권한을 적용하는 순간 관리 경로가 끊깁니다(트러블슈팅 ③). Anti-Lockout Rule은 관리 잠금을 막는 pfSense 기본 안전장치로 유지했습니다.

## 재현용 규칙표

GUI 재현이 가능한 형태의 규칙표와 Alias 정의는 아래를 참고하세요.

- [configs/pfsense-rules.md](../configs/pfsense-rules.md)
- [configs/pfsense-aliases.md](../configs/pfsense-aliases.md)
