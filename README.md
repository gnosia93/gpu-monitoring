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

----

아니요, CloudWatch로도 됩니다. GPU 모니터링은 CloudWatch가 네이티브로 지원하고 경로도 여러 개예요. "불가능"이 아니라 "무엇을 더 보고 싶으냐, 규모가 얼마냐"의 문제입니다.

CloudWatch로 GPU 보는 법
EC2 — CloudWatch 에이전트 설정에 nvidia_gpu 섹션만 추가하면 가동률·메모리·온도·전력이 CloudWatch로 올라갑니다. 기본 대시보드까지 딸린 "EC2 NVIDIA GPU" 전용 솔루션도 공식 제공됩니다. (내부적으로 nvidia-smi 기반) (CW 에이전트 GPU 문서)
EKS/ECS — Container Insights(enhanced observability)가 이제 NVIDIA DCGM 지표를 컨테이너·태스크·인스턴스 레벨로 자동 수집합니다. 에이전트 수동 설치 없이 add-on만 켜면 됩니다. AWS가 "예전엔 DCGM exporter·Prometheus를 직접 깔아야 했는데 이제 Container Insights가 대신 해준다"고 못 박을 정도로 턴키입니다. (AWS MT 블로그)
즉 EKS/ECS는 요즘 CloudWatch가 꽤 편해졌습니다.

그런데도 워크샵을 Prometheus/Grafana로 가는 이유
지표 깊이 (특히 EC2 단독): CW 에이전트 네이티브는 nvidia-smi 수준(가동률·메모리·온도·전력)입니다. DCGM+Prometheus는 Tensor Core 활성도, SM 활성도, NVLink/PCIe, XID/ECC 오류, 프로파일링까지 봅니다. "내 Tensor Core가 실제로 일하고 있나?"는 CW 네이티브로는 안 보입니다. (단, EKS/ECS는 CloudWatch도 이제 DCGM을 쓰므로 이 격차가 좁아집니다. 깊이 논리는 EC2 단독에서 가장 셉니다.)
대규모 비용: CloudWatch 커스텀 지표는 '지표 수 × GPU 수'로 과금돼 플릿이 커지면 비싸고 예측이 어렵습니다. AWS 자신의 EKS 비용 가이드도 CloudWatch/Prometheus 지표를 "주요 비용 동인"으로 꼽고, 커스텀 지표 축소를 1순위 절감책으로 둡니다 (EKS 비용 가이드). Prometheus는 돌리는 EC2 비용만 들어 카디널리티에 둔감합니다.
시각화·유연성: Grafana 대시보드와 커뮤니티 DCGM 대시보드, PromQL이 CloudWatch 대시보드보다 훨씬 자유롭습니다.
이식성·탈(脫)락인: 온프레미스 GPU와 타 클라우드까지 같은 스택으로 봅니다. 이게 바로 "상용(Datadog) 말고 오픈소스 self-managed"를 원하는 우리 타깃 고객이 CloudWatch 대신 이걸 고르는 핵심 이유입니다.
그래서 제안
워크샵에 "CloudWatch vs Prometheus/Grafana, 언제 뭘 고르나" 짧은 비교 섹션을 넣는 걸 추천합니다. 솔직하게 "작고 간단하면 CloudWatch, 깊고 크고 이식성이 필요하면 OSS"로 정리하고 본 실습은 OSS로 가면, 오히려 신뢰가 올라갑니다. 전략 문서의 "self-managed = EC2 소비" 논리와도 그대로 이어지고요.

이 비교 섹션을 워크샵 1번 모듈(개요)에 녹여서 초안 잡아드릴까요? 아니면 플랫폼(EC2/EKS) 정해서 Lab 1부터 바로 들어갈까요?




