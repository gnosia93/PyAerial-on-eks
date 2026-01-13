### _Instance Consideration_ ### 

  * G 타입 (Shared Memory / Socket) - 1/2, 1/4, 1/8, 1, 4, 8 GPU per Instance
    * 데이터 복사시 CPU가 개입하여 인터럽트를 처리하고 데이터를 메모리에 배치.
    * OS 스케줄링에 따라 지연 시간이 튀는 Jitter가 발생할 수 있음.
    * PCIe 토폴로지 단절: P2P(및 RDMA)가 작동하려면 데이터가 CPU를 거치지 않고 PCIe 스위치를 통해 직접 흘러야 하지만 G 타입은 하이퍼바이저(가상화 계층)가 GPU와 네트워크 카드 사이의 직접적인 통신 경로를 노출하지 않음.
  
  * P 타입 (외부 - GPUDirect RDMA / 내부 - NVLink) - 8 GPU per Instance
    * NIC 과 GPU 메모리가 직접 통신.
    * NVIDIA Aerial SDK가 요구하는 1ms 미만의 엄격한 TTI(Transmission Time Interval) 지원.
