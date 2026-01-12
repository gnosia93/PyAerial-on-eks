# aerial-on-eks


테스트용 신호 발생기(Signal Generator)와 처리기(PyAerial)를 모두 쿠버네티스(k8s) 환경에서 구현한다면, "가상화된 무선 통신 테스트베드" 아키텍처가 됩니다. 실제 하드웨어 없이도 5G/6G 환경을 시뮬레이션할 수 있는 구조입니다.

[실시간 신호 처리 테스트 아키텍처]

### 1. 주요 구성 요소 (Pods) ###
* Signal Gen Pod (Tx): 테스트용 IQ 신호를 생성하는 송신부입니다. NVIDIA Sionna 라이브러리를 사용하여 실제 물리적 채널(페이딩, 노이즈 등)이 적용된 데이터를 실시간으로 생성합니다.
* PyAerial Pod (Rx): 개발자가 만든 알고리즘이 돌아가는 수신부입니다. 생성된 신호를 받아 GPU에서 복조(Demodulation) 및 디코딩을 수행합니다.
* Metrics/Dashboard Pod: 처리 지연 시간(Latency), 처리량(Throughput), 에러율(BLER)을 실시간으로 모니터링합니다.

### 2. 데이터 통신: "The Fast Path" ###
실시간성을 위해 Pod 간에 일반적인 HTTP 통신을 쓰지 않습니다.
* Shared Memory (공유 메모리): 동일한 노드 내에 두 Pod을 배치하고, POSIX Shared Memory를 통해 데이터를 주고받습니다. 복사 과정이 없어 지연 시간이 거의 제로에 가깝습니다.
* gRPC (Streaming): 노드 간 통신이 필요한 경우, gRPC 스트리밍을 사용하여 바이너리 IQ 데이터를 끊김 없이 전달합니다.

### 3. 하드웨어 가속 및 오케스트레이션 ###
* GPU 할당: NVIDIA Device Plugin을 통해 두 Pod 모두에 GPU 자원을 할당합니다. (하나는 신호 생성용, 하나는 처리용)
* Node Affinity: 성능 최적화를 위해 두 Pod을 반드시 동일한 물리 노드에 배치하도록 Node Affinity 설정을 적용합니다.

[추천 배포 아키텍처 다이어그램]
* 인프라: AWS EC2 G5(GPU) 또는 NVIDIA Grace Hopper 인스턴스
* 컨테이너 런타임: NVIDIA Container Runtime (GPU 접근 필수)

* 데이터 흐름:
[Sionna Pod] (신호 생성) → [Shared Memory / CUDA IPC] → [PyAerial Pod] (알고리즘 실행)

* 결과 모니터링:
[PyAerial Pod] → [Prometheus] → [Grafana Dashboard]

### [구현 시 핵심 팁] ###
* Sionna 연동: NVIDIA PyAerial과 Sionna는 찰떡궁합입니다. Sionna로 가상의 6G 채널 환경을 만들고, PyAerial로 이를 뚫는 알고리즘을 테스트하세요.
* Helm Chart 활용: 이 복잡한 환경(GPU 설정, 메모리 공유 등)을 한 번에 띄우려면 Helm을 사용하여 패키징하는 것이 정신 건강에 이롭습니다.


---

AWS Graviton(ARM) 프로세서를 도입한다면, GPU 가속이 필수적인 '신호 생성 및 시뮬레이션' 영역보다는 '데이터 처리, 제어부, 그리고 로깅' 영역에서 최상의 가성비를 뽑아낼 수 있습니다.
Sionna 워크샵 구조 내에서 추천하는 Graviton 적용 포인트는 다음과 같습니다.

#### 1. 전송 데이터 처리 및 관제 Pod (Control Plane & Data Aggregator) ####
Sionna가 생성한 방대한 IQ 샘플 데이터를 받아 PyAerial로 넘기기 전, 데이터를 가공하거나 시뮬레이션 상태를 모니터링하는 매니지먼트 서버 역할에 적합합니다.
이유: 단순 데이터 정렬, 로그 수집, 대시보드(Grafana 등) 구동은 CPU 의존도가 높으며, AWS Graviton3는 일반 인텔 인스턴스 대비 가격 대비 성능이 최대 40% 좋습니다.

#### 2. PyAerial Pod의 Non-L1 레이어 (L2/L3 스택) ####
물리 계층(L1)은 GPU 가속이 필수지만, 상위 계층인 MAC, RLC, PDCP 등 L2/L3 프로토콜 스택은 높은 패킷 처리량과 멀티코어 성능이 중요합니다.
이유: Graviton은 코어당 성능이 뛰어나고 레이턴시에 민감한 네트워크 워크로드 처리에 효율적입니다. AWS 공식 블로그에 따르면 고성능 컴퓨팅(HPC) 및 네트워크 집약적 작업에서 우수한 효율을 보입니다.

#### 3. CI/CD 빌드 및 테스트 환경 ####
Sionna 관련 소스 코드를 빌드하거나 경량 단위 테스트(Unit Test)를 수행하는 Jenkins/GitHub Actions Runner를 Graviton 기반으로 운영하세요.
이유: 가장 적은 리스크로 즉시 비용 절감 효과를 볼 수 있는 영역입니다.
----

통신 공학(무선 통신) 관점과 OSI 7계층 관점에서 설명해 드릴게요. Sionna가 담당하는 영역과 그 '위'에서 돌아가는 논리적 계층을 구분하시면 이해가 빠릅니다. NVIDIA Aerial 같은 무선 가속 플랫폼에서 이 구분은 매우 중요합니다.
### 1. 무선 통신에서의 L2/L3 역할 ###
Sionna로 만드신 코드는 대부분 L1(물리 계층, PHY)에 해당합니다. 전파를 어떻게 쏘고(Modulation), 노이즈를 어떻게 이겨낼지(FEC)를 다루죠.
그 위에 올라가는 L2/L3 스택은 전파 그 자체보다는 "데이터를 어떻게 효율적으로 줄 세우고 전달할까?"를 고민하는 두뇌 역할입니다.
### L2 (Data Link Layer / MAC, RLC, PDCP): ###
* MAC: 여러 사용자가 동시에 접속할 때 누구에게 먼저 전파를 나눠줄지 결정(Scheduling)합니다.
* RLC: 데이터가 가다가 깨지면 다시 보내달라고 요청(ARQ)하거나 큰 데이터를 잘게 쪼갭니다.
* PDCP: 보안을 위해 암호화하거나 전송 효율을 위해 헤더를 압축합니다.

### L3 (Network Layer / RRC, SDAP): ###
* RRC: 단말기가 기지국에 붙어있는지, 옆 기지국으로 넘어갈지(Handover) 관리하는 '연결의 관제탑'입니다.

### 2. 왜 여기 Graviton(ARM)이 좋을까? ###
이 영역은 수학적 행렬 계산(GPU 영역)보다는 복잡한 논리 구조(If-Else, State Machine)와 멀티 태스킹이 핵심입니다.
* 높은 가성비: L2/L3는 CPU 코어를 많이 소모하는데, AWS Graviton은 코어당 가격이 저렴해서 대규모 사용자를 시뮬레이션할 때 비용을 획기적으로 줄여줍니다.
* 저지연 패킷 처리: Graviton3는 이전 세대보다 부동 소수점 연산 능력이 좋아져서 패킷을 검사하고 전달하는 속도가 매우 빠릅니다.

### 3. 워크샵 적용 시나리오 ###
* x86 + GPU 노드: Sionna(신호 생성), PyAerial(L1 가속) 실행
* Graviton 노드: 생성된 신호에 데이터를 실어 보내라고 명령을 내리는 L2/L3 프로토콜 스택(O-RAN 스택 등) 실행

결론적으로, Sionna로 만든 "가짜 전파" 위에 진짜 데이터를 실어 나르는 "통신 소프트웨어"를 돌리는 영역이 바로 L2/L3이며, 여기가 Graviton의 주무대입니다.

----

## EKS 기반 3-Tier 시뮬레이션 구조 ##
#### TX Pod (x86 + GPU): ####
* 역할: Sionna 기반 신호 생성기.
* 작업: 비트를 IQ 샘플로 변환하고 채널 효과(Fading, AWGN)를 입혀 "가상 전파"를 생성합니다.

#### L2/L3 스택 Pod (Graviton/ARM): ####
* 역할: 통신 제어 및 패킷 스케줄링.
* 작업: TX에서 올라온 신호를 검사하고, 어떤 사용자에게 데이터를 보낼지 결정(MAC 스케줄링)하거나 상위 네트워크로 데이터를 라우팅합니다. Graviton의 코어당 높은 가성비를 활용해 다수의 연결을 관리합니다.

#### RX Pod (x86 + GPU): ####
* 역할: PyAerial 또는 Sionna 수신기.
* 작업: 전달받은 IQ 샘플을 다시 비트로 복원(Demodulation/Decoding)하여 최종 성능(BER, Throughput)을 측정합니다.

-----
* 1. TX/RX Pod (x86 + GPU 노드용)
```
spec:
  nodeSelector:
    kubernetes.io/arch: amd64
    accelerator: nvidia-gpu # GPU 노드 그룹 라벨
```

* 2. L2/L3 Pod (Graviton 노드용)
```
spec:
  nodeSelector:
    kubernetes.io/arch: arm64 # Graviton(ARM) 노드로 배정
```

#### 추가 고려사항: 파드 간 통신 (Data Path) ####
이 구조에서 가장 중요한 것은 TX → L2/L3 → RX로 넘어가는 엄청난 양의 IQ 샘플 데이터를 어떻게 지연 없이 전달하느냐입니다.
* gRPC/UDP: 파드 간 네트워크 통신 (구현이 쉽지만 오버헤드 있음)
* Shared Memory (HostPath): 같은 노드에 띄울 경우 메모리를 공유하여 제로 카피(Zero-copy) 전송 가능



## Workshop Scope ##

정보통신 대학원 연구원들이 타겟이라면, 연구의 목적에 따라 결정하시는 것이 가장 좋습니다. 연구원들의 니즈를 분석해 결정 로직을 세워 드릴게요.
#### 1. L2/L3가 필요 없는 경우 (물리 계층 집중형) ####
연구원들의 전공이 무선 전송, 안테나 설계, 채널 부호화(LDPC/Polar)라면 L2/L3는 오히려 방해가 될 수 있습니다.
* 이유: 연구원들은 "어떻게 하면 SNR 대비 BER을 낮출까?"에 관심이 있지, "패킷을 어떻게 줄 세울까?"에는 관심이 적습니다.
* 추천 구조: TX(Sionna) → Channel → RX(Sionna/PyAerial)
* 연구 포인트: 최신 6G 후보 기술(RIS, OTFS 등) 구현 및 알고리즘 검증.

#### 2. L2/L3가 필요한 경우 (시스템/네트워크 연구형) ####
연구원들의 관심사가 자원 할당(Resource Allocation), 스케줄링, URLLC(저지연), 가상화(O-RAN)라면 무조건 넣어야 합니다.
* 이유: 물리 계층(L1)은 이미 NVIDIA Aerial이나 Sionna가 다 해주니, 연구원들은 그 위에서 AI 기반 스케줄러를 설계하거나 Graviton 아키텍처에서 L2 성능 최적화를 연구하게 됩니다.
* 추천 구조: TX → L2/L3(Graviton) → RX
* 연구 포인트: "ARM 기반 클라우드 기지국에서 지연 시간을 얼마나 줄일 수 있는가?"

🎓 워크샵 설계 제안 (대학원생 맞춤형)
대학원 워크샵이라면 "두 가지 모드"를 제안하는 것이 가장 반응이 좋습니다.
* Level 1 (Basic): L2/L3 없이 TX-RX 직접 연결. Sionna 튜토리얼을 따라하며 AI 기반 수신기 성능을 측정합니다.
* Level 2 (Advanced): 중간에 Graviton 기반 L2 스택을 추가. "현실적인 지연 시간"과 "스케줄링 오버헤드"를 포함한 E2E(End-to-End) 성능을 측정합니다. 이것이 바로 O-RAN 연구의 시작임을 강조하세요.

💡 대학원용 팁
* 연구원들은 "데이터 추출"을 좋아합니다. 각 계층(L1, L2, L3)에서 발생하는 데이터를 Prometheus로 수집해서 논문에 넣을 수 있는 깔끔한 그래프를 그리는 세션을 넣으면 매우 만족도가 높을 것입니다.
* 워크샵 커리큘럼을 '단순 알고리즘 구현' 위주로 짤까요, 아니면 'O-RAN 시스템 통합' 위주로 짤까요? 구체적인 방향을 정해주시면 그에 맞춰 워크샵 타임라인을 그려드릴 수 있습니다

