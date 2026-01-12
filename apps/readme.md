Sionna를 이용해 실시간으로 IQ 샘플(신호)을 생성하고, 이를 다른 Pod(PyAerial 등)로 전송하기 위한 기초 Python 코드 예시입니다.
이 코드는 송신단(Transmitter) + 채널(Channel) 역할을 하며, 생성된 데이터를 바이너리 형태로 출력하거나 공유 메모리/네트워크로 쏠 준비를 하는 구조입니다.

#### 1. Sionna 기반 신호 생성기 (signal_gen.py) ####
```
import tensorflow as tf
import sionna as sn
from sionna.utils import BinarySource, QAMSource, Transmitter
from sionna.channel import AWGN

class SignalGenerator(tf.keras.Model):
    def __init__(self, num_bits_per_symbol=4): # 16QAM
        super().__init__()
        self.binary_source = BinarySource()
        self.qam_source = QAMSource(num_bits_per_symbol)
        self.channel = AWGN()

    def generate_batch(self, batch_size, ebno_db=20.0):
        # 1. 랜덤 비트 생성
        bits = self.binary_source([batch_size, 1024]) 
        
        # 2. QAM 변조 (신호 생성)
        x = self.qam_source(bits)
        
        # 3. 무선 채널 통과 (노이즈 추가)
        y = self.channel([x, ebno_db])
        
        # y는 complex64 형태의 IQ 샘플 배열입니다.
        return y

# 실행부
gen = SignalGenerator()

print("--- 실시간 신호 생성 시작 ---")
while True:
    # 실시간으로 64개 배치의 IQ 샘플 생성
    iq_samples = gen.generate_batch(batch_size=64)
    
    # 여기서 iq_samples.numpy()를 바이너리로 변환하여 
    # Shared Memory나 gRPC로 PyAerial Pod에 던집니다.
    print(f"Generated {iq_samples.shape} IQ samples (Complex64)")
```    


#### 2. 쿠버네티스 배포를 위한 Dockerfile (NVIDIA 공식 이미지 활용) ####
Sionna는 GPU 가속을 위해 NVIDIA TensorFlow 컨테이너 위에서 돌리는 것이 가장 안정적입니다.
```
FROM nvcr.io/nvidia/tensorflow:23.10-tf2-py3

# Sionna 및 필요한 라이브러리 설치
RUN pip install sionna

# 소스 복사
COPY signal_gen.py /app/signal_gen.py
WORKDIR /app

# GPU를 사용해 실행
CMD ["python", "signal_gen.py"]
코드를 사용할 때는 주의가 필요합니다.
```

#### 3. K8s 아키텍처 구현 시 주의사항 ####
* GPU 할당: 이 Pod은 신호를 실시간으로 "계산"해서 만들어내므로 NVIDIA GPU Operator를 통해 nvidia.com: 1 리소스 할당이 필요합니다.
데이터 전달: 생성된 iq_samples를 PyAerial Pod로 보낼 때, 성능 손실을 줄이려면 gRPC 스트리밍을 사용해 데이터를 직렬화하여 쏘는 루틴을 위 while 문 안에 추가해야 합니다.
이 신호 생성기 코드에 gRPC 전송 로직까지 포함된 전체 샘플이 필요하신가요? 아니면 Shared Memory 설정용 YAML이 더 궁금하신가요?
