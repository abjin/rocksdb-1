## 🔍 1. 실험 목적

> **"메모리를 아껴주는 Ribbon 필터는 실전에서도 Bloom을 대체할 수 있을까?"**


Ribbon 필터는 Bloom 필터보다 메모리 사용량을 최대 30% 줄일 수 있지만, 필터 생성 시 CPU 사용량이 더 높습니다. 

이 실험의 목적은 다음과 같습니다:
- **목적 1**: 동일한 false positive rate 설정에서 **Bloom vs Ribbon 필터의 성능(쓰기/읽기/공간 효율 등)**을 비교한다.
- **목적 2**: **Bits-Per-Key(bpk)** 값을 조절하며 성능이 어떻게 달라지는지 분석한다.
- **목적 3**: 워크로드 유형(읽기/쓰기 집중형)에 따라 어떤 필터가 유리한지 평가한다.
    

---

## 🧪 2. 실험 시나리오

### 🎯 실험 A. 기본 성능 비교: Bloom vs Ribbon (bpk=10)

- **가설**: 
    - 동일한 bits-per-key 조건에서 Ribbon은 메모리는 덜 쓰지만 쓰기 시 CPU 오버헤드가 발생할 것이다.
    - 읽기 쓰기 레이턴시는 비슷할 것이다.
    
- **설정**:
    - Filter: BuiltinBloom vs Ribbon
    - bits_per_key = 10
    - Workload: fillrandom, readrandom, readwhilewriting
    - Key-value: key=32B, value=1024B
    - 데이터 수: 5,000,000개 항목
    - write_buffer_size: 8MB
    - target_file_size_base: 8MB
    - 압축: 없음
    - 스레드: 2개
- **주요 확인 지표**:
    - `ops/sec` (특히 fillrandom, readrandom, readwhilewriting 각 단계별 처리량)
    - `latency (P50/P95)` (읽기/쓰기 지연 시간)
    - `memory usage` (필터 자체의 메모리 사용량 및 전체 DB 메모리 사용량)
    - `CPU 사용률` (특히 fillrandom 단계에서의 CPU 사용량)
    - `FPR`: False Positve Rate
    - 컴팩션 통계 및 빈도
    - DB 크기 비교
        

### 🎯 실험 B. Bits-Per-Key 다이얼 튜닝 실험

- **가설**: bits_per_key가 낮아지면 메모리 효율은 올라가지만 false positive 증가로 성능은 하락할 것이다.
    
- **설정**:
    - Filter: BuiltinBloom 또는 Ribbon
    - bits_per_key = 5, 7, 10, 12
    - Workload: readrandom
- **주요 확인 지표**:
    - `memory usage` (bpk 값에 따른 메모리 사용량 변화)
    - `ops/sec` (특히 readrandom에서의 처리량 변화, FP rate 증가로 인한 성능 하락 확인)
    - `latency (P50/P95)` (bpk 값에 따른 지연 시간 변화)
    - (실제 False Positive Rate의 변화 추이)
        

---

## 🖥️ 3. 실험 명령어

### ✅ 실험 A. Bloom vs Ribbon 기본 비교


```bash
# 🔧 공통 옵션
--db=${db_path}
--num=5000000 \
--bloom_bits=10
--value_size=1024 \
--key_size=32 \
--threads=2 \
--block_size=4096 \
--write_buffer_size=8388608 \
--target_file_size_base=8388608 \
--compression_type=none \
--statistics \
--report_bg_io_stats=true\
```

```bash
# Bloom Filter (DB 경로: ./bloom_filter_db)
# 1. 데이터 삽입 (fillrandom)
./db_bench --use_ribbon_filter=false --benchmarks=fillrandom --use_existing_db=0 \
  [...공통 옵션들...]

# 2. 무작위 읽기 (readrandom)
./db_bench --use_ribbon_filter=false --benchmarks=readrandom --use_existing_db=1 \
  [...공통 옵션들...]

# 3. 쓰기 중 읽기 (readwhilewriting)
./db_bench --use_ribbon_filter=false --benchmarks=readwhilewriting --use_existing_db=1 \
  [...공통 옵션들...]


# Ribbon Filter (DB 경로: ./ribbon_filter_db)
# 1. 데이터 삽입 (fillrandom)
./db_bench --use_ribbon_filter=true --benchmarks=fillrandom --use_existing_db=0 \
  [...공통 옵션들...] # 예: --num=5000000 --value_size=1024 ...

# 2. 무작위 읽기 (readrandom)
./db_bench --use_ribbon_filter=true --benchmarks=readrandom --use_existing_db=1 \
  [...공통 옵션들...]

# 3. 쓰기 중 읽기 (readwhilewriting)
./db_bench --use_ribbon_filter=true --benchmarks=readwhilewriting --use_existing_db=1 \
  [...공통 옵션들...]
```

각 필터 타입 별로 다음 벤치마크를 순차적으로 실행:
- fillrandom (데이터 삽입)
- readrandom (무작위 읽기)
- readwhilewriting (읽기 중 쓰기)

추가 모니터링:
- CPU, 메모리, 디스크 I/O 지속 모니터링
- 컴팩션 통계 수집
- 최종 DB 크기 비교


### ✅ 실험 B. bpk 다이얼 테스트

```bash
# 🔧 공통 옵션
--db=${db_path}
--num=3000000 \
--bloom_bits=10
--value_size=1024 \
--key_size=32 \
--threads=2 \
--block_size=4096 \
--write_buffer_size=8388608 \
--target_file_size_base=8388608 \
--compression_type=none \
--statistics \
--report_bg_io_stats=true\
```

```bash
# 다이얼 테스트
for bpk in 0 5 10 15 20 25; do
    # Bloom Filter
    ./db_bench --bloom_bits=$bpk --use_ribbon_filter=false \ 
    --use_existing_db=0 \
    --benchmarks=fillrandom \
    [...공통 옵션들...]

    ./db_bench --bloom_bits=$bpk --use_ribbon_filter=false \
    --use_existing_db=1 \
    --benchmarks=readrandom \
    [...공통 옵션들...]

    # Ribbon Filter
    ./db_bench --bloom_bits=$bpk --use_ribbon_filter=true \ 
    --use_existing_db=0 \
    --benchmarks=fillrandom \
    [...공통 옵션들...]

    ./db_bench --bloom_bits=$bpk --use_ribbon_filter=true \
    --use_existing_db=1 \
    --benchmarks=readrandom \
    [...공통 옵션들...]
done
```

---

## 🧠 결과 해석 프레임

- Ribbon의 **공간 절약 vs CPU 오버헤드**는 어떤 워크로드에서 더 타당한가?
- **읽기 중심** vs **쓰기 중심** 워크로드에서 어떤 필터가 적합한가?
- bits-per-key를 조절하면서 FPR이 실제 성능에 어떤 영향을 주는가?
    

---

## 📊 4. 실험 결과 요약

### 📈 실험 A 결과 요약


1. **성능 지표**:

    #### 1. **fillrandom (랜덤 쓰기)**
    
    - **처리량(ops/sec)**: 
      - Bloom: **4,401 ops/sec**
      - Ribbon: **4,373 ops/sec**
      - → Bloom이 약 **0.6% 더 빠름**, 그러나 차이는 무시할 수준
    
    - **P50 쓰기 지연시간**: 
      - Bloom: **26.75 μs**
      - Ribbon: **24.18 μs**
      - → 실질적으로 동일한 지연시간
    
    #### 2. **readrandom (랜덤 읽기)**
    
    - **처리량(ops/sec)**: 
      - Bloom: **30,430 ops/sec**
      - Ribbon: **30,321 ops/sec**
      - → 차이는 **0.3%** 이하로 거의 동일
    
    - **P50/P99 읽기 지연시간**: 
      - Bloom: **17.17 μs / 510.40 μs**
      - Ribbon: **17.34 μs / 512.34 μs**
      - → 실질적으로 동일한 지연시간
    
    #### 3. **readwhilewriting (쓰기 중 읽기)**
    
    - **처리량(ops/sec)**: 
      - Bloom: **23,455 ops/sec**
      - Ribbon: **23,378 ops/sec**
      - → 차이는 **0.3%** 이하로 거의 동일

    ![워크로드별 처리량 비교](images/throughput_comparison.png)
    
    - **P50/P95/P99 읽기 지연시간**: 
      - Bloom: **6.33 μs / 29.04 μs / 45.27 μs**
      - Ribbon: **6.26 μs / 28.87 μs / 44.48 μs**
      - → 거의 동일한 읽기 지연시간 보임

    ![워크로드별 지연 시간 비교](images/latency_comparison.png)

2. **리소스 사용량**:

    #### 1. **fillrandom (랜덤 쓰기)**

    ![Fillrandom CPU 및 메모리 사용량](images/fillrandom.png)

    - **CPU 사용량:**
        
        - Bloom: **최소 3.08% ~ 최대 6.27%**
            
        - Ribbon: **최소 3.15% ~ 최대 7.71%**
            
        - → Ribbon이 최대 CPU 사용률이 **1.44%p 더 높음**, 이는 필터 생성 과정에서 **더 복잡한 연산**을 수행하기 때문으로 보임.
            
    - **메모리 사용량:**
        
        - Bloom: **986MB ~ 1079MB**
            
        - Ribbon: **965MB ~ 1040MB**
            
        - → Ribbon이 평균적으로 **약 30~40MB 적은 메모리**를 사용. 메모리 효율성이 뛰어남.
            


    #### 2. **readrandom (랜덤 읽기)**

    ![Readrandom CPU 및 메모리 사용량](images/readrandom.png)

    - **CPU 사용량:**
        
        - Bloom: **5.56% ~ 8.24%**
            
        - Ribbon: **5.57% ~ 8.34%**
            
        - → 거의 동일 (**0.1%p 차이**), 필터 조회 시 성능 차이는 **통계적으로 유의미하지 않은 수준**.
            
    - **메모리 사용량:**
        
        - Bloom: **1011MB ~ 1037MB**
            
        - Ribbon: **980MB ~ 1025MB**
            
        - → Ribbon이 평균적으로 **약 20MB 더 적게 사용**, 읽기 성능은 유사하되 메모리 측면에서 이점 존재.
            


    #### 3. **readwhilewriting (쓰기 중 읽기)**

    ![Readwhilewriting CPU 및 메모리 사용량](images/readwhilewriting.png)

    - **CPU 사용량:**
        
        - Bloom: **8.71% ~ 10.86%**
            
        - Ribbon: **8.50% ~ 10.80%**
            
        - → 거의 동일 (**0.1%p 차이**), 필터 조회 시 성능 차이는 **통계적으로 유의미하지 않은 수준**.
            
    - **메모리 사용량:**
        
        - Bloom: **1048MB ~ 1076MB**
            
        - Ribbon: **1035MB ~ 1067MB**
            
        - → Ribbon이 **10MB 정도 적게 메모리** 사용. 고부하 상황에서도 메모리 측면에서 유리.

    ![워크로드별 CPU 사용량 비교](images/cpu_usage_comparison.png)
    ![워크로드별 메모리 사용량 비교](images/memory_usage_comparison.png)

3. **컴팩션 통계**:
    
    - **컴팩션 CPU 시간**:
        - Bloom: **884,532,930 μs**
        - Ribbon: **892,376,951 μs**
        - → 평균 컴팩션 시간은 Ribbon이 약간 더 김
    
    ![컴팩션 통계](images/compaction_time_comparison.png)
    


5. **False Positive Rate (FPR)**:

    ![FPR 비교 (bpk=10)](images/fpr_comparison.png)

    - Bloom 필터와 Ribbon 필터 모두 **약 0.94%** 수준의 유사한 FPR을 기록함 (bpk=10 기준). 이는 설정된 bits_per_key 값에 부합하는 결과임.

### 📈 실험 B 결과 요약

쓰기 성능 (fillrandom, ops/초)
| bits_per_key | Bloom 필터 | Ribbon 필터 |
|--------------|------------|-------------|
| 0 | 11,065 | 10,516 |
| 5 | 10,951 | 10,721 |
| 10 | 10,641 | 10,561 |
| 15 | 10,229 | 10,340 |
| 20 | 10,017 | 10,142 |
| 25 | 10,244 | 10,237 |

![BPK별 쓰기 성능 비교](images/b-write_performance.png)

읽기 성능 (readrandom, ops/초)
| bits_per_key | Bloom 필터 | Ribbon 필터 |
|--------------|------------|-------------|
| 0 | 60,543 | 68,272 |
| 5 | 60,071 | 61,369 |
| 10 | 67,836 | 69,809 |
| 15 | 71,241 | 66,653 |
| 20 | 60,125 | 57,310 |
| 25 | 69,364 | 39,241 |

![BPK별 읽기 성능 비교](images/b-read_performance.png)

DB 크기 (MB)
| bits_per_key | Bloom 필터 | Ribbon 필터 |
|--------------|------------|-------------|
| 0 | 2,867.91 | 2,870.87 |
| 5 | 2,861.26 | 2,868.48 |
| 10 | 2,871.73 | 2,870.64 |
| 15 | 2,872.67 | 2,871.98 |
| 20 | 2,871.41 | 2,873.30 |
| 25 | 2,875.19 | 2,871.88 |

![BPK별 DB 크기 비교](images/b-db_size.png)

성능 최대값 요약
| 측정항목 | Bloom 필터 | Ribbon 필터 |
|-------------------|-------------|-------------|
| 최고 쓰기 성능 | 11,065 (bpk=0) | 10,721 (bpk=5) |
| 최고 읽기 성능 | 71,241 (bpk=15) | 69,809 (bpk=10) |
| 최적 bits_per_key | 15 | 10 |


오탐률 (False Positive Rate)
| bits_per_key | Bloom 필터 FPR (%) | Ribbon 필터 FPR (%) |
| :----------- | :----------------- | :------------------ |
| 5 | 9.130 | 9.165 |
| 10 | 0.942 | 0.947 |
| 15 | 0.119 | 0.122 |
| 20 | 0.0196 | 0.0179 |
| 25 | 0.0040 | 0.0032 |

![BPK별 FPR 비교](images/b-fpr.png)



1. 쓰기 성능은 bits-per-key 값이 낮을수록 유리하다
bits-per-key(bpk)가 낮아지면 필터 생성 시 필요한 메모리가 줄고, 그만큼 생성 속도도 빨라진다
→ 따라서 쓰기 중심의 워크로드에서는 bpk를 낮게 설정하는 것이 유리하지만, 어느 정도의 FPR은 감수해야 한다.

2. 읽기 성능은 bpk가 중간값일 때 가장 좋다
bpk가 너무 낮으면 FPR이 높아지고, 이는 불필요한 디스크 접근을 유발해 읽기 성능을 저하시킨다. 반대로 bpk가 너무 높으면 필터 정확도는 좋아지지만 필터 크기가 커져 캐시 부담이 생기거나 필터 접근 자체가 느려질 수 있다.
실험 결과 Bloom 필터는 bpk=15에서, Ribbon 필터는 bpk=10에서 가장 높은 읽기 성능을 보였다.
→ 필터 종류에 따라 최적의 bpk는 다르다.

3. FPR과 성능은 트레이드오프 관계
FPR은 bpk가 증가할수록 로그 스케일로 감소한다.
bpk=5: FPR 약 9%
bpk=10: 약 1%
bpk=15: 0.1% 이하
bpk=25: 0.003% 수준
읽기 성능은 FPR이 너무 높으면 급격히 떨어지며, 반대로 FPR이 너무 낮다고 해서 무조건 성능이 좋아지는 것은 아니다. 오히려 필터 조회 속도나 캐시 효율이 떨어져 성능이 저하될 수 있다.

4. Ribbon 필터가 유리한 상황
Ribbon은 쓰기 성능보다는 메모리 효율에서 강점을 가진다.
쓰기 작업이 많지 않고, 시스템 메모리가 제한적인 경우에는 Ribbon이 적합하며 bpk는 10~12 정도가 적절하다.
반대로 쓰기 부하가 크고 CPU 자원이 부족하다면 Bloom 필터가 더 나은 선택이 될 수 있다.
특히 대규모 데이터셋을 다루는 환경에서 메모리 절약이 중요한 경우, Ribbon 필터가 유리하다.