[← README로 돌아가기](../README.md)

# pfSense 방화벽 규칙 (재현용)

pfSense 2.8.1 GUI(`Firewall → Rules`)에서 재현할 수 있는 형태로 규칙 12개를 정리했습니다. 규칙 순서는 위→아래 평가, 첫 매치에서 정지입니다. Pass는 위, Block은 아래에 둡니다. Alias 정의는 [pfsense-aliases.md](pfsense-aliases.md) 참고.

## WAN 탭

| 순서 | Action | Proto | Source | Destination | Port | Log |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP | any | `DMZ_WEB` | 443 | - |
| 2 | Pass | TCP | any | `DMZ_WEB` | * | - |
| 3 | Block | any | any | any | * | ✔ |

## DMZ 탭

| 순서 | Action | Proto | Source | Destination | Port | Log |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | DMZ net | `MON_SERVER` | 5044 | - |
| 2 | Pass | ICMP | DMZ net | DMZ address | * | - |
| 3 | Block | any | DMZ net | LAN net | * | ✔ |
| 4 | Block | any | any | any | * | ✔ |

> 2번의 Source는 `DMZ net`이어야 합니다. `DMZ address`(/32)로 두면 DMZ 내부 서버 트래픽이 매칭되지 않습니다.

## LAN(관제망) 탭

Anti-Lockout Rule은 pfSense 기본값으로 유지합니다.

| 순서 | Action | Proto | Source | Destination | Port | Log |
|---|---|---|---|---|---|---|
| 1 | Pass | TCP | LAN net | This Firewall (self) | 443 | - |
| 2 | Pass | ICMP | LAN net | This Firewall (self) | * | - |
| 3 | Pass | TCP | `MON_SERVER` | any | 443 | - |
| 4 | Pass | UDP | `MON_SERVER` | any | 53 | - |
| 5 | Block | any | any | any | * | ✔ |

## 재현 시 유의점

- **Default Deny** — 각 탭 맨 아래 `Block any → any (Log)`가 원칙을 강제합니다. 이 규칙을 지우면 존 분리가 무의미해집니다.
- **로그** — 모든 Block에 Log를 켭니다. 차단 로그가 관제 자산입니다.
- **롤백 안전망** — 초기 Allow-All 규칙은 삭제하지 말고 Disable 상태로 보존합니다.
- **self 규칙** — LAN 1·2번(`This Firewall (self)`)을 빠뜨리면 관리 경로가 잠깁니다.
