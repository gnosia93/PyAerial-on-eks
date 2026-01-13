Shared Memory(공유 메모리) 방식은 네트워크 스택을 거치지 않고 메모리 주소를 직접 공유하기 때문에 gRPC보다 훨씬 빠릅니다.
Python에서 이를 구현하려면 multiprocessing.shared_memory를 사용하거나, 더 고성능을 위해 NVIDIA cuMem (CUDA Inter-process Communication)을 활용하지만, 여기서는 가장 일반적이고 구현이 쉬운 POSIX Shared Memory 기반의 샘플을 보여드립니다.

#### 1. [Sionna Pod] 신호 생성 및 메모리 쓰기 ####
생성한 IQ 샘플을 공유 메모리 영역에 쓰고, Event나 Semaphore 역할을 할 플래그를 업데이트합니다.
```
import numpy as np
from multiprocessing import shared_memory
import sionna as sn
import time

# 1. 공유 메모리 생성 (약 10MB 크기: Complex64 125만 샘플 분량)
shm = shared_memory.SharedMemory(name="iq_stream", create=True, size=10 * 1024 * 1024)

# 2. 신호 생성기 설정 (Sionna)
binary_source = sn.utils.BinarySource()
qam_source = sn.utils.QAMSource(num_bits_per_symbol=4)

try:
    print("Sionna: 공유 메모리에 신호를 쓰고 있습니다...")
    while True:
        # 신호 생성
        bits = binary_source([1, 1024])
        x = qam_source(bits).numpy() # (1, 256) 형태의 complex64
        
        # 3. 공유 메모리에 데이터 복사
        # 앞부분 4바이트는 데이터 크기나 시퀀스 번호로 활용 가능
        shared_array = np.ndarray(x.shape, dtype=x.dtype, buffer=shm.buf)
        shared_array[:] = x[:]
        
        time.sleep(0.001) # 실시간 주기 조절 (1ms)
except KeyboardInterrupt:
    shm.close()
    shm.unlink()
```

#### 2. [PyAerial Pod] 메모리 읽기 및 알고리즘 실행 ####
동일한 name으로 메모리에 접근하여 데이터를 즉시 가져옵니다.
```
import numpy as np
from multiprocessing import shared_memory
from nvidia.aerial import pyaerial as pa
import time

# 1. 기존 공유 메모리에 연결
shm = shared_memory.SharedMemory(name="iq_stream")

# 2. PyAerial 초기화
ctx = pa.Context()
# 예: L1 처리를 위한 도구 세팅
# estimator = pa.l1.ChannelEstimator(ctx)

try:
    print("PyAerial: 공유 메모리에서 데이터를 읽어 GPU로 처리합니다...")
    while True:
        # 3. 메모리에서 직접 numpy 배열로 매핑 (복사 없음)
        # 생성단과 동일한 shape와 dtype 지정
        iq_data = np.ndarray((1, 256), dtype=np.complex64, buffer=shm.buf)
        
        if iq_data.any():
            # 4. PyAerial 알고리즘 실행
            # processed = estimator.run(iq_data)
            print(f"처리 중... 첫 번째 샘플: {iq_data[0][0]}")
            
        time.sleep(0.001)
except KeyboardInterrupt:
    shm.close()
```

#### 3. K8s 설정 (Shared Memory 활성화) ####
두 컨테이너가 /dev/shm을 통해 동일한 자원을 보게 하려면 emptyDir 설정이 필수입니다.
```
apiVersion: v1
kind: Pod
metadata:
  name: aerial-shm-pod
spec:
  # 중요: 두 컨테이너가 동일한 IPC 네임스페이스를 공유하도록 설정
  hostIPC: true 
  containers:
  - name: sionna-gen
    image: my-sionna-gen:latest
    volumeMounts:
    - mountPath: /dev/shm
      name: cache-volume
  - name: pyaerial-rx
    image: nvidia-pyaerial:latest
    volumeMounts:
    - mountPath: /dev/shm
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      medium: Memory # RAM을 사용하여 디스크 I/O 병목 제거
```

#### 핵심 장점 ####
* Zero-Copy: 데이터가 네트워크 카드나 커널을 거치지 않고 메모리 주소만 넘겨주므로 지연 시간이 거의 없습니다.
* 간결함: 복잡한 gRPC 프로토콜 정의 없이 넘파이 배열 구조만 맞추면 됩니다.
* 주의: 위 코드는 데이터가 써지는 동안 읽는 경합 현상(Race Condition)이 발생할 수 있습니다. 실제 구현 시에는 Python 주식/거래 시스템처럼 세마포어나 간단한 플래그 변수를 메모리 첫 칸에 두어 쓰기 완료 여부를 체크하는 것이 좋습니다.
