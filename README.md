# 트래픽 폭증 대응 티켓 예매 서비스

콘서트·스포츠 티켓 예매처럼 순간적으로 트래픽이 폭증하는 서비스를 가정하고, **좌석 동시성 제어 · 대기열 설계 · DB 읽기/쓰기 분리 · 실시간 이상탐지**를 설계·구현하고, K6 부하테스트와 자체 모의해킹(nmap·Nikto)으로 직접 검증한 개인 프로젝트입니다.

> 설계 문서에서 멈추지 않고, VM 8대로 실제 인프라를 구축해 수치로 증명하는 것을 목표로 했습니다.

![아키텍처](repo-assets/architecture.png)

## 핵심 결과

| 검증 항목 | 결과 |
|---|---|
| 좌석 동시성 제어 | 동일 좌석 100건 동시요청 → 성공 정확히 1건 (3가지 조건 모두 동일) |
| 트래픽 폭증 대응 (대기열) | 최대 응답지연 60,000ms → **108.8ms** (약 99.8% 감소) |
| DB 복제 | Primary→Replica 실시간 동기화 확인 (Seconds_Behind_Master: 0) |
| 실시간 이상탐지 | Suricata가 공격 시뮬레이션 트래픽을 실시간 탐지 |
| 자체 모의해킹 | 설계한 포트 매트릭스와 실제 nmap 스캔 결과 100% 일치 |
| 회원 인증 검증 | bcrypt 해시 저장, 로그인 5회 실패 시 423 잠금, 비밀번호 변경 전/후 해시 갱신 확인 |

## 무엇부터 보면 되나요

- **전체를 훑고 싶다면** → [`docs/ticket-portfolio-full.pdf`](docs/ticket-portfolio-full.pdf) (설계 계획 + 실행 결과 통합본, 23p)
- **발표/면접용 요약이 필요하다면** → [`slides/ticket-project-v2.pptx`](slides/ticket-project-v2.pptx)
- **코드를 보고 싶다면** → [`scripts/`](scripts/)
- **트러블슈팅 과정이 궁금하다면** → [`docs/troubleshooting-full.pdf`](docs/troubleshooting-full.pdf)

## 인프라 구성

VMware Fusion 위에 VM 8대, `172.16.60.0/24` 대역.

| 서버 | 역할 |
|---|---|
| bastion | SSH 유일 진입점 |
| queue-gate | 대기열 / Rate Limiting + Suricata IDS |
| web-lb | 로드밸런서 + 정적 페이지 서빙 |
| was-01 / was-02 | 좌석 예약 API (Redis 원자연산 기반 동시성 제어) |
| redis | 좌석 재고 원자연산 |
| db-01 / db-01-replica | MariaDB Primary-Replica 복제 |

관리자는 bastion을 거쳐서만 SSH 접근이 가능하고, 사용자는 queue-gate(80번 포트)만을 통해 서비스를 이용합니다. 두 경로는 방화벽으로 물리적으로 분리되어 있습니다.

## 기술 스택

`VMware Fusion` `Ubuntu` `Nginx` `Node.js` `Express` `Redis` `MariaDB` `Suricata` `K6` `nmap` `Nikto` `ufw`

## 문서 목록 (docs/)

| 문서 | 내용 |
|---|---|
| ticket-portfolio-full.pdf | 설계 계획(Part 1~7) + 실행 결과(Part 8~9) 통합본 |
| session-report.pdf | 구축 세션 요약, 서버 접속경로 다이어그램 |
| as-built.pdf | 설계 대비 실제 구현 변경사항, 서버별 최종 명령어 |
| loadtest-report.pdf | 부하테스트 결과 (대기열 ON/OFF 비교) |
| concurrency-report.pdf | 좌석 동시성 검증 결과 |
| pentest-report.pdf | 자체 모의해킹 결과 (nmap, Nikto) |
| webapp-vuln-report.pdf | 웹 애플리케이션 취약점 분석 (OWASP Top 10 자가점검) |
| auth-verification-report.pdf | 회원가입/로그인/잠금/비밀번호변경 및 관리자 계정 만료 정책 검증 |
| db-replication-log.pdf | DB 복제 구축 기록 및 트러블슈팅 |
| suricata-report.pdf | Suricata 실시간 이상탐지 구축 리포트 |
| troubleshooting-full.pdf | 전체 세션 트러블슈팅 종합 (증상·원인·해결·교훈) |

## 코드 구조 (scripts/)

```
scripts/
├── was-api/       예약 API 서버 (server.js, schema.sql)
├── k6/            부하테스트 스크립트 (동시성/대기열/공격시뮬레이션)
├── nginx/         queue-gate, web-lb 설정
├── suricata/      IDS 룰 및 설치 스크립트
└── firewall/      서버별 ufw 규칙
```

## 다음 단계

- [x] 회원 인증 API 구현 (bcrypt 해시, 로그인 실패 잠금, 비밀번호 변경) — AUTH-2026-0707-01 참고
- [ ] IDOR(접근제어) 테스트 및 결과 문서화
- [ ] 세션 쿠키 보안속성(HttpOnly/Secure/SameSite) 확인
- [ ] TLS 적용, 잔여 보안 헤더 2종 조치
- [ ] Hydra·sqlmap 실증 테스트

## 라이선스 및 주의사항

본 프로젝트는 학습·포트폴리오 목적으로 제작되었으며, 모의해킹 도구는 본인이 소유한 격리된 가상 네트워크에서만 사용했습니다. 문서 내 비밀번호 등 민감값은 실제 값 대신 플레이스홀더로 표기되어 있습니다.

---

작성자: 최시은 · 2026.07
