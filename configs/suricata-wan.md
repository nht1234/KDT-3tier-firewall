[← README로 돌아가기](../README.md)

# Suricata WAN 인터페이스 설정

## 배치와 모드

Suricata를 pfSense의 **WAN(em0) 경계**에 배치하고 **IDS 모드**로 운영했습니다(NIDS).

| 항목 | 값 |
|---|---|
| 배치 인터페이스 | WAN (em0) |
| 운영 모드 | IDS (탐지 전용) |
| Block Offenders | 해제 |
| 룰셋 | ETOpen Emerging Threats |

## 왜 IDS 모드인가 (Block Offenders 해제)

경계에서 먼저 차단해 버리면, 뒤에 있는 호스트 계층(Snort)과 애플리케이션 계층(ModSecurity)의 탐지 성능을 검증할 수 없습니다. 이번 실습의 목적은 다계층 탐지가 실제로 겹겹이 작동하는지 확인하는 것이었으므로, 경계는 **탐지만 하고 트래픽은 통과**시키도록 IDS 모드로 뒀습니다.

즉, 차단 우선순위보다 "각 계층이 무엇을 잡아내는지"를 관측하는 것을 우선했습니다. 탐지 신뢰도를 확보한 뒤 IPS(Block Offenders)로 전환하는 것은 향후 과제로 남겼습니다.

## 룰셋 선정

ETOpen Emerging Threats 룰셋 중 이 실습 시나리오와 맞는 카테고리를 활성화했습니다.

| 카테고리 | 대상 |
|---|---|
| `emerging-scan` | 포트 스캔·정찰 탐지 |
| `emerging-dos` | 서비스 거부 시도 |
| `emerging-exploit` | 익스플로잇 시도 |
| `emerging-web_server` | 웹서버 대상 공격 |

WAN 경계에서 가장 먼저 관측되는 것은 스캔·정찰과 웹서버 대상 트래픽이므로, 해당 카테고리를 우선했습니다.

## 로그 연동

Suricata 로그는 pfSense(FreeBSD)에서 Syslog로 관제 서버에 전송합니다(5140). FreeBSD라 리눅스용 Filebeat를 쓸 수 없어 경로를 이원화한 배경은 [docs/03-log-pipeline.md](../docs/03-log-pipeline.md) 참고.

## 한계 (향후 과제)

- **탐지 로그 분석** — 탐지 로그를 기반으로 이벤트 유형을 분석하고, 정상과 오탐을 판단하는 경험을 확장해야 합니다.
- **오탐 튜닝** — 현재 ETOpen 기본 룰셋을 그대로 사용 중. 실제 트래픽 기준 Suppress 리스트를 구성해야 합니다.
- **IDS → IPS 전환** — 탐지 신뢰도 확보 후 Block Offenders를 단계적으로 적용합니다.
