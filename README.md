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

-----


좋습니다. 플랫폼은 앞서 추천한 EC2 먼저 → EKS 확장 2-트랙, 2.5~3시간, 입문자 대상으로 가정하고 초안을 잡았습니다. (다르면 조정할게요.) 지금은 폴더가 안 열려 있어 채팅으로 드리고, 개요 모듈은 완성본, 각 Lab은 "무엇을·어떻게"가 보이는 계획 수준으로 썼습니다.

GPU 관측성 워크샵 — Prometheus & Grafana로 NVIDIA GPU 모니터링 (self-managed)
오픈소스만으로 EC2 GPU의 가동률·메모리·Tensor Core·온도·오류를 실시간으로 보고 알림까지 거는 핸즈온.

대상: AWS에서 GPU(학습/추론) 워크로드를 돌리며, 상용 도구(Datadog 등) 대신 오픈소스로 직접 관측성을 꾸리려는 엔지니어 선수지식: 기본 Linux·EC2, Docker는 몰라도 됨(개념부터 설명) 소요: 약 2.5~3시간 비용 주의: GPU 인스턴스는 과금이 큽니다. 실습은 저렴한 g5.xlarge 또는 g6.xlarge(GPU 1장)로 충분하고, 끝나면 반드시 종료합니다.

끝나면 할 수 있는 것

GPU EC2에 dcgm-exporter를 띄워 GPU 지표 노출
self-managed Prometheus + Grafana로 수집·시각화
핵심 지표(가동률·메모리·Tensor Core·온도·오류) 해석
실제 부하로 지표가 움직이는 걸 관찰
유휴·과열·오류 알림 구성
(확장) EKS에서 GPU Operator + kube-prometheus-stack으로 플릿 단위 관측
Module 0 — 개요 & 아키텍처
왜 GPU를 따로 모니터링하나
GPU는 EC2에서 가장 비싼 자원입니다. 그런데 안 보면 세 가지를 놓칩니다. (1) 비싼 GPU가 10%만 쓰여도 모른 채 돈을 태우고, (2) 메모리 부족(OOM)이나 열 스로틀로 학습이 느려지거나 죽는데 원인을 못 찾고, (3) XID·ECC 같은 하드웨어 오류를 놓칩니다. 관측성은 "이 비싼 자원을 제대로 쓰고 있는가"를 보는 눈입니다.

데이터 흐름

[GPU EC2 인스턴스]
  NVIDIA 드라이버
  dcgm-exporter  ──(:9400/metrics)──┐
                                     ▼
                              [Prometheus] ──(쿼리)──▶ [Grafana 대시보드]
                                     │
                               [Alertmanager] ──▶ Slack / Email
DCGM: NVIDIA가 만든 GPU 텔레메트리 수집기(드라이버 수준에서 지표를 뽑음)
dcgm-exporter: DCGM 지표를 Prometheus가 읽을 수 있는 형식으로 :9400에 노출
Prometheus: 그 엔드포인트를 주기적으로 긁어와(scrape) 시계열로 저장
Grafana: Prometheus에 질의해 그래프/대시보드로 보여줌
Alertmanager: 조건(예: 유휴·과열)에 걸리면 알림 발송
꼭 볼 핵심 지표 (DCGM 필드)
지표	의미	왜 중요한가
DCGM_FI_DEV_GPU_UTIL	GPU 가동률(%)	유휴(낭비)·과포화 판단
DCGM_FI_DEV_FB_USED/FREE	GPU 메모리 사용/여유	OOM 예방, 배치 크기 튜닝
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE	Tensor Core 활용률	ML 효율의 핵심 신호
DCGM_FI_PROF_GR_ENGINE_ACTIVE	연산 엔진 활성 비율	"진짜로 일하는 중인지"
DCGM_FI_DEV_GPU_TEMP	온도	스로틀링·하드웨어 보호
DCGM_FI_DEV_POWER_USAGE	전력(W)	전력 상한·효율
DCGM_FI_DEV_XID_ERRORS	XID 오류	하드웨어/드라이버 장애 조기 신호
DCGM_FI_DEV_ECC_*	ECC 메모리 오류	메모리 결함 감지
CloudWatch vs Prometheus/Grafana — 언제 뭘 쓰나
가능 여부로는 CloudWatch도 GPU를 봅니다(EC2는 CW 에이전트 nvidia_gpu, EKS/ECS는 Container Insights가 DCGM 자동 수집). 그래서 선택 기준은 "되냐 안 되냐"가 아니라 아래입니다.

기준	CloudWatch	Prometheus/Grafana (이 워크샵)
설치·운영	관리형, 운영 부담 적음	직접 설치·운영
지표 깊이	EC2는 nvidia-smi 수준(기본)	DCGM 전체(Tensor Core·XID·ECC 등)
대규모 비용	지표 수×GPU로 증가, 예측 어려움	돌리는 EC2 비용만, 카디널리티에 둔감
시각화	기본 대시보드	Grafana·커뮤니티 DCGM 대시보드로 자유
이식성	AWS 전용	온프레미스·타 클라우드 동일 스택
정리: 작고 간단하면 CloudWatch, 깊이·대규모 비용·이식성이 필요하면 OSS. 이 워크샵의 타깃(상용 대신 오픈소스 self-managed를 원하는 고객)에게는 OSS가 정답이고, 스택이 EC2 위에서 상시 도는 것도 이 선택의 결과입니다.

Lab 1 — GPU EC2 준비 (약 20분)
목표: 드라이버가 깔린 GPU 인스턴스를 띄우고 nvidia-smi로 GPU 인식 확인
핵심: g5.xlarge 또는 g6.xlarge 기동, Deep Learning AMI(드라이버 사전 설치) 사용 권장, 보안그룹은 내 IP로만 제한
확인: nvidia-smi 에 GPU 1장이 보이면 성공
Lab 2 — dcgm-exporter 배포 (약 20분)
목표: GPU 지표를 :9400에 노출
핵심: Docker로 nvcr.io/nvidia/k8s/dcgm-exporter 실행(--gpus all, -p 9400:9400)
확인: curl localhost:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL 로 지표가 나오면 성공
Lab 3 — Prometheus + Grafana 올리기 (약 30분)
목표: self-managed 수집·시각화 스택 구성
핵심: Docker Compose로 Prometheus(:9090) + Grafana(:3000) 기동, Prometheus scrape_configs에 dcgm-exporter 타깃 등록
보안: 3000·9090·9400 포트를 공인망에 열지 말고, SSH 터널 또는 내 IP 한정 보안그룹으로 접근
확인: Prometheus Targets 화면에서 dcgm-exporter가 UP
Lab 4 — 대시보드 (약 25분)
목표: GPU 대시보드를 띄우고 패널을 읽는다
핵심: Grafana에 Prometheus 데이터소스 연결 → 커뮤니티 대시보드 ID 12239(NVIDIA DCGM) 임포트 → 각 패널(가동률·메모리·온도·Tensor Core) 의미 해설
확인: 실시간 그래프가 그려짐
Lab 5 — 부하 걸어 관찰 (약 25분)
목표: 지표가 "살아있게" 움직이는 걸 본다 (이 랩이 체감의 핵심)
핵심: gpu-burn 컨테이너로 간단 부하, 또는 실제 추론 서버(vLLM)로 모델 서빙 → 가동률·Tensor Core·전력·온도 상승 관찰
확인: 부하 전/후 그래프 비교
Lab 6 — 알림 (약 25분)
목표: 돈·장애와 직결된 알림 걸기
핵심: Alertmanager 연결 후 규칙 작성 — ① 유휴 GPU(장시간 저가동=비용 낭비), ② 고온, ③ XID/ECC 오류
확인: 임계치를 낮춰 알림이 실제로 발화
(확장 트랙) Lab 7 — EKS에서 플릿 단위 관측 (약 40분)
목표: 프로덕션 규모로 올리기
핵심: EKS GPU 노드그룹 + NVIDIA GPU Operator(dcgm-exporter를 DaemonSet으로 자동 배포) + kube-prometheus-stack(Prometheus Operator·Grafana·Alertmanager 번들), ServiceMonitor로 수집, Pod/네임스페이스 단위 GPU 귀속까지
확인: 여러 GPU 노드가 한 대시보드에 모임
정리 (Cleanup)
GPU 인스턴스·EKS 노드그룹을 반드시 종료/삭제해 과금 방지 (체크리스트 제공)
이게 전체 뼈대 초안입니다. 다음으로 둘 중 하나를 하면 됩니다.

Lab 1~3을 실제 명령어·Compose 파일·Prometheus 설정까지 완성본으로 작성 (가장 먼저 "그래프가 뜨는" 지점까지)
혹은 이걸 AWS Workshop Studio 디렉터리 구조(contentspec.yaml + 모듈별 markdown)로 변환할 수 있게 폴더를 열어두고 파일로 생성
어느 쪽부터 갈까요? 그리고 실습 명령을 작성할 때 현재 버전(dcgm-exporter, GPU Operator 등)을 그 시점에 한 번 더 확인해서 핀 고정하겠습



이 비교 섹션을 워크샵 1번 모듈(개요)에 녹여서 초안 잡아드릴까요? 아니면 플랫폼(EC2/EKS) 정해서 Lab 1부터 바로 들어갈까요?

-----------------

GPU 확보가 어려운 상황, 워크샵 개발할 때 정말 흔한 벽이에요. 결론부터: 됩니다. GPU 없이 CPU 인스턴스에서 DCGM 지표를 "가짜로" 만들어 Prometheus로 보내는 방법이 여러 개 있습니다. 현재 쓸 수 있는 실제 도구들을 확인해서 정확히 알려드릴게요.


3 tool calls
Web search: fake dcgm-exporter mock GPU metrics Prometheus without GPU test
10 results

Web search: DCGM fake GPU injection mode nv-hostengine dcgmi test inject metrics
10 results

Web search: avalanche prometheus synthetic metrics generator fake exporter
10 results


Thought complete
This is a solid set of real tools to work with. I'm ranking the best fit for building a GPU monitoring workshop on a CPU-only instance without real hardware, starting with purpose-built fake DCGM exporters that let dashboards and alerts work unchanged. Fake-dcgm-exporter is the simplest drop-in since it serves DCGM-named metrics so Prometheus and existing dashboards light up immediately. The dcgm-fake-gpu-exporter is more compelling for a workshop though, since it offers realistic behavior profiles (idle, training, thermal throttle), scales to simulate large fleets, and uses DCGM's actual injection framework so metric names match a real deployment exactly—great for both the fleet module and alert testing. There's also an Intel-based Go exporter worth considering as a third option. For Kubernetes-level simulation, run-ai/fake-gpu-operator stands out—it can make a CPU-only EKS cluster appear to have GPU nodes complete with a fake dcgm-exporter, letting the whole EKS GPU-observability lab run without real hardware, though I should flag that it emits no temperature or power data so some dashboard panels will stay blank unless recording rules are added. Beyond that, there are generic Prometheus load-testing tools like avalanche that could help with scale testing even though they're not GPU-specific. CGM's own injection framework — synthetic field injection via dcgmi test --inject, NVML Injection mode that swaps in fake implementations, and an error injection framework for simulating GPU faults like XID/ECC events. The fake exporters mentioned are actually built on top of this, so while you could wire nv-hostengine and dcgm-exporter against injected NVML directly on a CPU box, it's fiddly — better to lean on the pre-built fakes and reserve DCGM's native error-injection for the proper XID/ECC alert lab. I also want to flag Grafana dashboard 25618, "NVIDIA DCGM GPU Overview (tested GPU-free)" — it's explicitly designed and validated without a real GPU, working against both real dcgm-exporter and simulated sources like fake-gpu-operator. That's a great fit for the workshop, though temp/power panels need recording rules when paired with fake-gpu-operator. For the recommendation itself, I'm leaning toward saiakhil2012/dcgm-fake-gpu-exporter as the primary choice for the EC2/Docker track given its realistic profiles and scale, with crazytaxii/fake-dcgm-exporter as a simpler fallback — both pointed at Prometheus the same way the real exporter would be. For the EKS track, run-ai/fake-gpu-operator lets a CPU-only cluster present fake GPU nodes and metrics, enabling a full GPU Operator + kube-prometheus-stack + ServiceMonitor lab without real hardware, with the same temp/power caveat. The key selling point across both tracks is that since these tools emit identical DCGM metric names, everything built on fake data — dashboards, PromQL, alert rules — transfers unchanged once real GPUs are available; you just swap exporters. I should be upfront that synthetic data means some panels like temperature/power may stay flat or blank depending on the tool, and realism is bounded by the profile quality. For the "load → watch it move" lab, fake data means driving the pattern via a profile switch rather than a real gpu-burn workload, which works for demos but is worth flagging. For actual development, the fastest setup is Docker Compose on a cheap CPU instance (or even locally) running fake-dcgm-exporter, Prometheus, Grafana, and the standard dashboards — enough to build the whole EC2 track without touching a GPU until final validation. This also opens up a nice workshop feature: offering a "no-GPU / simulation mode" path for attendees without GPU quota, lowering the barrier to entry. I'll structure my answer as a direct yes with the real tools grouped by fit, per-track recommendations for EC2/EKS, the key insight about matching exact metric names to avoid rework, caveats about synthetic data and blank temp/power fields, and a concrete dev setup suggestion — possibly offering to add a "시뮬레이션 모드" to the Lab setup or produce a Docker Compose file. I'll link the GitHub repos inline, keeping citations light and properly attributed rather than over-quoting. I'm drafting the response in Korean, concise but complete, using a short table or grouped bullets since the user prefers beginner-friendly explanations of what each tool does and why to pick it. For primary recommendations, I'm leaning toward the Docker-based fake GPU exporters for simple and realistic EC2 setups, a config-driven alternative, the fake-gpu-operator for EKS, a GPU-free-tested Grafana dashboard, and a DCGM error-injection framework for alert labs. I'll make sure to emphasize the "zero rework" point prominently since that's the key professional insight, and close by offering to rewrite Labs 1-4 in a "simulation mode" so the workshop can run end-to-end without real GPUs.
있습니다. 그것도 아주 잘 맞는 도구들이 여러 개 있어요. 핵심은 **"DCGM과 똑같은 지표 이름(DCGM_FI_DEV_*)을 가짜로 뿜어주는 익스포터"**를 CPU 인스턴스에 띄우는 겁니다. 이러면 Prometheus는 진짜 dcgm-exporter를 긁는 것과 똑같이 동작하고, 대시보드·알림도 그대로 켜집니다.

바로 쓸 수 있는 도구 (실제 존재 확인함)
EC2(Docker) — 가장 잘 맞음

saiakhil2012/dcgm-fake-gpu-exporter — 워크샵용 1순위 추천. DCGM injection 프레임워크 기반이라 지표 이름이 실제와 동일하고, **7가지 동작 프로파일(유휴/학습/추론/스로틀 등)**과 1000+ GPU 스케일 시뮬레이션, Docker 지원. 대시보드도 살아있게 움직이고 플릿 규모도 흉내 냅니다.
crazytaxii/fake-dcgm-exporter — "GPU 없는 개발 환경용 가짜 DCGM exporter." 가장 단순한 드롭인. 일단 그래프부터 띄우고 싶을 때.
intel/fakedev-exporter — Intel이 만든 Go 익스포터, GPU 시뮬레이션 주목적에 설정으로 값 패턴 제어. 설정 기반을 선호하면.
EKS 트랙 — 쿠버네티스 레벨에서 통째로 가짜 4. run-ai/fake-gpu-operator — CPU 노드만 있는 EKS 클러스터를 GPU 노드가 있는 것처럼 보이게 합니다(가짜 dcgm-exporter 포함). GPU Operator + kube-prometheus-stack + ServiceMonitor + Pod 단위 귀속 랩을 GPU 없이 그대로 만들 수 있어요.

대시보드 / 알림 보조 5. Grafana 대시보드 25618 "NVIDIA DCGM GPU Overview (tested GPU-free)" — 아예 GPU 없이 만들고 테스트한 대시보드. 진짜 dcgm-exporter에도 그대로 동작. (단, fake-gpu-operator는 온도·전력을 안 뿜어서 그 두 패널은 recording rule을 넣어야 채워집니다 — 캐비엇.) 6. DCGM Error Injection 프레임워크 (NVIDIA 문서) — XID/ECC 같은 하드웨어 오류를 주입해 Lab 6(알림)에서 "오류 알림이 실제로 터지는" 걸 보여줄 때 정석.

(참고로 prometheus-community/avalanche는 지표 이름이 GPU용이 아니라 범용 부하 생성기라, "대규모 비용/카디널리티" 설명용으로는 좋지만 GPU 대시보드엔 안 맞습니다.)

이게 왜 결정적이냐 (zero rework)
위 도구들은 실제 DCGM과 동일한 지표 이름을 노출합니다. 그래서 가짜 데이터로 만든 대시보드·PromQL·알림 규칙이 나중에 진짜 GPU로 바꿔도 그대로 작동합니다. 개발 땐 fake exporter, 검증 땐 진짜 dcgm-exporter로 컨테이너만 교체하면 끝. 워크샵 제작 전 과정을 GPU 없이 끝내고, 마지막에 GPU 한 대로 한 번만 최종 확인하면 됩니다.

솔직한 한계
합성 데이터라 패턴이 "진짜 학습 부하"만큼 자연스럽진 않습니다. Lab 5(부하 관찰)는 gpu-burn 대신 프로파일 전환(유휴→학습)으로 연출하게 됩니다.
도구에 따라 온도·전력 등 일부 지표는 비거나 평평할 수 있습니다(위 캐비엇).
제안
이거 하나로 워크샵이 더 좋아집니다. **"GPU 없이도 되는 시뮬레이션 모드"**를 워크샵에 정식 경로로 넣으면, GPU 쿼터가 없는 수강생도 끝까지 따라올 수 있어 진입 장벽이 확 낮아집니다.

원하시면 Lab 1~4를 "시뮬레이션 모드"(fake-dcgm-exporter + Prometheus + Grafana + 대시보드 25618) 기준으로 Docker Compose·설정까지 완성본으로 다시 잡아드릴게요. 지금 바로 CPU 인스턴스(또는 로컬)에서 돌려볼 수 있는 형태로요. 그렇게 갈까요?

Est. Credits Used: 3.85




