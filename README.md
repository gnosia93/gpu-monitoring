# gpu-monitoring




표준 아키텍처: NVIDIA DCGM Exporter → Prometheus → Grafana. dcgm-exporter가 GPU 텔레메트리(가동률, 메모리, 온도, 전력, Tensor Core 사용률, ECC/XID 오류 등)를 Prometheus 포맷(9400 포트)으로 노출하고, Prometheus가 수집, Grafana가 대시보드로 보여줍니다. self-managed(오픈소스 직접 운영) 기준으로 가면 됩니다.

제안 모듈 구성 (핸즈온)
* 개요 & 아키텍처 — 왜 GPU 관측성인가, 데이터 흐름(DCGM→Prom→Grafana), 꼭 봐야 할 핵심 지표 설명
* Lab 1: GPU 인스턴스 준비 — GPU EC2(g5/g6/p4d/p5) 기동, NVIDIA 드라이버(DLAMI 사용 또는 설치) 확인
* Lab 2: dcgm-exporter 배포 — 컨테이너로 띄우고 curl :9400/metrics로 지표 확인
* Lab 3: Prometheus + Grafana 올리기 — Docker Compose로 self-managed 스택 구성, 스크랩 설정
* Lab 4: 대시보드 — 커뮤니티 대시보드(ID 12239 등) 임포트 + 각 패널/지표 의미 해설
* Lab 5: 부하 걸어 보기 — gpu-burn 또는 실제 추론(vLLM)·학습(PyTorch)으로 가동률·메모리·Tensor Core가 실제로 움직이는 걸 관찰 (이게 있어야 대시보드가 "살아있게" 보입니다)
* Lab 6: 알림 — Alertmanager로 유휴 GPU(비용 낭비), 과열, ECC/XID 오류 알림
* 정리(Cleanup) — 리소스 삭제로 과금 방지

결정이 필요한 한 가지 (플랫폼)
GPU 모니터링은 이 지점에서 내용이 크게 갈립니다.

A. EC2 단독 — 가장 쉬움. dcgm-exporter를 Docker로 바로 실행. 학습/추론 VM 한두 대 모니터링용. 입문자 추천.
B. EKS — 운영/스케일용. NVIDIA GPU Operator(dcgm-exporter를 DaemonSet으로 자동 배포) + kube-prometheus-stack. Pod/네임스페이스 단위 GPU 귀속, 오토스케일링 신호까지 가능. 전략 문서의 "EKS 스케일링"과 직접 연결됩니다.
추천: A(EC2)로 "누구나 10분 안에 그래프가 뜨는" 성공 경험을 주고, B(EKS)를 프로덕션 확장 모듈로 얹는 2-트랙. 설치→설정→운영 흐름과도 맞습니다.

시작 전에 두 가지만 확인할게요
플랫폼: A만 / B만 / A→B 둘 다 중 뭘로 갈까요? (추천은 둘 다, EC2 먼저)
포맷: 다른 워크샵들처럼 AWS Workshop Studio 카탈로그용으로 만들 건가요, 아니면 우선 **일반 핸즈온 가이드(README+스크립트)**로 시작할까요? 전자면 Workshop Studio 디렉터리 구조(contentspec.yaml 등)에 맞춰 뼈대를 잡겠습니다.
(기본 가정: 2~3시간, 입문자 대상 핸즈온. 다르면 알려주세요.)
