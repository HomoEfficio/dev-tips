# Vector Data Quantization

- 수백 만 ~ 수억 건의 벡터 데이터를 그대로 저장해서 탐색하면 비효율적이므로, 데이터의 크기 압축 필요 -> 양자화(quantization) 기법을 통해 데이터 크기를 줄인다.

## Product Quantization

- 고차원의 벡터 데이터를 M개의 서브 벡터를 사용하는 PQ를 사용해서 8bit(1byte) * M 로 압축
- 예를 들어, 4byte인 float 값을 가지는 512차원 데이터 100만개를, M = 4 인 PQ로 압축하면,
  - 4byte * 512차원 * 1,000,000 을 1byte(`[0, 255]`인 정수) * M(==4) * 1,000,000 로 압축하므로 1/512로 압축
- 쿼리 요청 시마다 거리표 생성, 기존 데이터(100만건)와의 근사 거리 계산이 수행되므로 데이터 갯수에 비례하는 반복 연산을 줄이기 위해 IVF, HNSW 등과 함께 사용

### 양자화 방식

<img width="720" height="1019" alt="슬라이드1" src="https://github.com/user-attachments/assets/71082098-e993-473a-a59e-06e6d4051b4a" />

### 벡터 데이터 검색

<img width="720" height="1019" alt="슬라이드2" src="https://github.com/user-attachments/assets/8aca8867-28f1-4fc2-85d2-5a264612e9d0" />


