[← README로 돌아가기](../README.md)

# pfSense Alias 정의

IP를 규칙에 직접 박지 않고 Alias로 추상화했습니다. 규칙이 주소가 아니라 역할로 읽히고, 주소가 바뀌어도 Alias 한 곳만 고치면 됩니다. `Firewall → Aliases`에서 정의합니다.

| Alias 이름 | 유형 | 값 | 의미 |
|---|---|---|---|
| `MON_SERVER` | Host | 192.168.100.10 | 관제 서버(ELK) |
| `DMZ_WEB` | Host | 192.168.200.10 | DMZ 웹서버(DVWA) |
| `DMZ_WIN_VICTIM` | Host | 192.168.200.20 | DMZ 피해자 Windows |

## 사용 예

- WAN 탭 1·2번 규칙의 Destination → `DMZ_WEB`
- DMZ 탭 1번 규칙의 Destination → `MON_SERVER`
- LAN 탭 3·4번 규칙의 Source → `MON_SERVER`

## 왜 Alias인가

- **가독성** — `any → DMZ_WEB : 443`은 "외부에서 웹서버 443만 허용"으로 바로 읽힙니다. IP만 있으면 규칙을 읽어도 의미가 드러나지 않습니다.
- **유지보수** — 서버 IP가 바뀌면 규칙 여러 개가 아니라 Alias 값 하나만 수정합니다.
- **일관성** — 같은 호스트를 여러 규칙에서 참조할 때 오타로 인한 불일치를 줄입니다.

> `DMZ_WIN_VICTIM`은 피해자 Windows를 가리키는 Alias로 정의해 두었습니다. 공격 시나리오 구성은 팀원 담당이며, 본인은 네트워크·규칙 관점에서 대상 호스트를 Alias로 명시하는 데까지 담당했습니다.
