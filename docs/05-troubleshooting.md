[← README로 돌아가기](../README.md)

# 05. 트러블슈팅

구축 중 총 13건을 해결했습니다. 배운 점이 분명했던 3건을 상세히 기록하고, 나머지 10건은 요약합니다. 공통된 교훈은 하나입니다 — **증상을 보고 설정을 바꾸지 말고, 양끝에서 가능성을 지워 원인을 좁힌다.**

---

## ① WAN 게이트웨이조차 닿지 않음 — 어댑터 오결선

**증상**
기본 게이트웨이 라우트를 넣었는데도 `ping 192.168.111.254`가 실패. `arp -n` 전 항목이 `incomplete`.

**원인 격리 (3단계)**
1. pfSense 콘솔에서 `.200.20`은 응답하고 `.111.100`은 `Host is down` → 방화벽 자체는 살아 있음.
2. WAN(em0) status `active`, MAC 정상 → pfSense 쪽 문제 아님으로 확정.
3. 남은 건 Kali. 어댑터 2장의 MAC을 대조하니, WAN IP가 DMZ에 물린 `eth1`에 할당돼 있었음.

**조치**
```bash
ip addr flush dev eth1
ip addr add 192.168.111.100/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.111.254 dev eth0
```

**결과**
Kali → pfSense → Windows 10 전 구간 통신 성공 (손실 0%).

**배운 점**
통신 불통은 "어느 쪽 문제인가"를 먼저 배제해야 한다. 방화벽 규칙을 계속 고치는 대신 양끝에서 좁혀 들어간 것이 정답이었다. L3 설정이 맞아도 L1·L2가 어긋나면 의미가 없다.

---

## ② DMZ 서버에서 게이트웨이 ping 실패 — net vs address

**증상**
DVWA(200.10) → 게이트웨이(200.254) ping 100% loss.

**원인**
DMZ 규칙의 Source를 `DMZ address`로 지정. 이건 pfSense 자신의 DMZ 인터페이스 IP 한 개(/32)를 뜻하고, DMZ 안 서버에서 나가는 트래픽은 매칭되지 않음.

**조치**
Source를 `DMZ net`(대역 전체, /24)으로 변경.

**배운 점**
방화벽 UI의 한 단어가 /32와 /24를 가른다. 규칙을 넣고 끝내지 않고, 실제 트래픽으로 매칭 여부를 확인하는 습관이 생겼다.

---

## ③ 관제 서버 → pfSense 단방향 불통 — self 규칙 누락

**증상**
pfSense → 관제서버 ping은 성공, 역방향은 실패.

**원인 격리**
관제 서버에서 `rp_filter`, `iptables`, `ufw`를 전부 확인해 호스트 측 요인을 배제 → 방화벽 규칙 문제로 확정.

**원인**
LAN 규칙을 최소 권한으로 좁히면서, pfSense 자기 자신(self)으로 향하는 ICMP·443 허용이 누락됨.

**조치**
LAN 탭 Block 위에 아래 2개 추가.
```
Pass / TCP  / LAN net → This Firewall (self) / 443
Pass / ICMP / LAN net → This Firewall (self)
```

**배운 점**
정책을 조이는 작업은 자기 자신을 잠글 위험을 항상 동반한다. 최소 권한을 적용할 때 관리 경로가 희생되지 않는지 먼저 점검하게 됐다.

---

## 나머지 10건 요약

장비별로 정리합니다.

- **pfSense** — WAN에 Windows 10 대상 ICMP 허용 규칙 부재 (Default deny). 의도된 차단으로 확인.
- **Kali** — 기본 게이트웨이 라우트 미설정 / curl 빈 응답(302 리다이렉트, 정상).
- **DMZ 웹서버** — nginx 403 (index.php 및 php 블록 누락) / netplan 기본 GW 오설정 / DVWA DB 접속 500 (config.inc.php 키 오타) / Filebeat 설치 경로 결정.
- **관제 서버** — 라우팅 점검(Docker 브리지 의심 → 정상) / Kibana 5601 접속 불가(ES 8.x HTTPS 강제) / 로그 수집 정체(syslog 포트 변경).
- **Windows 10** — 가상 어댑터 "미디어 연결 끊김" / 게이트웨이·DNS 오지정.

> DMZ 웹서버·관제 서버·Kali·Windows 항목 중 애플리케이션·공격 구성 부분은 팀원 담당 영역이며, 여기서는 네트워크 도달성 관점에서 함께 확인한 내용을 기록합니다.
