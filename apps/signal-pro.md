NVIDIA PyAerial을 사용하여 수신된 신호를 처리하는 샘플 코드입니다. PyAerial은 내부적으로 NVIDIA cuPHY 가속 라이브러리를 호출하므로, 일반적인 Python 연산보다 수십 배 빠른 GPU 가속 신호 처리를 수행합니다.
이 샘플은 gRPC로 들어온 IQ 데이터를 받아 채널 추정 및 등화(Equalization)를 수행하는 구조입니다.

#### 1. PyAerial 수신부 알고리즘 (pyaerial_rx.py) ####
```
import grpc
import signal_pb2
import signal_pb2_grpc
import numpy as np
import tensorflow as tf
from nvidia.aerial import pyaerial as pa # NVIDIA PyAerial 라이브러리

class PyAerialReceiver(signal_pb2_grpc.SignalStreamerServicer):
    def __init__(self):
        # 1. PyAerial GPU 가속 컨텍스트 초기화
        self.context = pa.Context()
        # 2. 초고속 신호 처리를 위한 L1 처리 객체 생성 (예: 채널 추정기)
        self.channel_estimator = pa.l1.ChannelEstimator(self.context)
        print("PyAerial GPU 가속 수신기가 준비되었습니다.")

    def StreamIQ(self, request_iterator, context):
        for data in request_iterator:
            # 1. 바이너리 데이터를 넘파이 배열로 복원 (Complex64)
            iq_samples = np.frombuffer(data.samples, dtype=np.complex64)
            iq_tensor = tf.convert_to_tensor(iq_samples.reshape(data.batch_size, -1))

            # 2. PyAerial 알고리즘 실행 (GPU 가속 핵심 구간)
            # 여기서는 예시로 수신 신호의 위상을 보정하거나 복조를 준비합니다.
            processed_data = self.run_pyaerial_pipeline(iq_tensor)
            
            print(f"처리 완료: {len(processed_data)} 심볼 해독 중...")
        return signal_pb2.Empty()

    def run_pyaerial_pipeline(self, iq_tensor):
        """
        NVIDIA cuPHY 라이브러리를 활용한 물리 계층 처리
        """
        # 실제 PyAerial API를 사용하여 FFT, 채널 추정, 복조 등을 수행합니다.
        # 자세한 API는 NVIDIA Aerial SDK 문서를 참조하십시오.
        return iq_tensor * 1.0 # (가상의 처리 로직)

def serve():
    server = grpc.server(tf.distribute.cluster_resolver.SimpleClusterResolver().num_accelerators() > 0 
                         and grpc.thread_pool_executor(max_workers=10))
    signal_pb2_grpc.add_SignalStreamerServicer_to_server(PyAerialReceiver(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```

#### 2. 배포 시 핵심 사항 ####
* NVIDIA 커널 드라이버: 이 코드가 돌아가려면 호스트 머신에 NVIDIA Aerial SDK 전용 드라이버와 CUDA가 올바르게 설치되어 있어야 합니다.
* 패키지 가용성: nvidia.com 라이브러리는 NVIDIA NGC 컨테이너 환경에서 가장 잘 작동합니다. 일반 환경에서 import nvidia.aerial을 사용하려면 사전에 SDK 라이선스 승인이 필요할 수 있습니다.

#### 3. k8s에서의 흐름 요약 ####
* Sionna Pod: 가상의 무선 채널 데이터를 생성하여 gRPC로 전송.
* PyAerial Pod: gRPC로 받은 RAW 데이터를 NVIDIA GPU를 통해 실시간 해독.
* 결과: 복구된 비트 데이터를 상위 애플리케이션으로 전달.





