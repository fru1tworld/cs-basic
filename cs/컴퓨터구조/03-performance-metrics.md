# 성능 지표 (Performance Metrics)

## 개요

컴퓨터 성능을 측정하고 비교하기 위해 다양한 지표가 사용된다. "무엇이 더 빠른가?"라는 질문에 답하려면 명확한 성능 지표의 정의와 이해가 필요하다. 성능 지표를 올바르게 이해해야 시스템 최적화와 하드웨어 선택에서 올바른 결정을 내릴 수 있다.

## 핵심 개념

### 성능의 정의

성능(Performance)은 상황에 따라 다르게 정의된다:

```
┌────────────────────────────────────────────────────────────┐
│                      성능의 두 가지 관점                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  응답 시간 (Response Time / Latency)                       │
│  ├── 작업 하나를 완료하는 데 걸리는 시간                     │
│  ├── 사용자 관점 (개인 컴퓨터, 인터랙티브 시스템)            │
│  └── 작을수록 좋음                                         │
│                                                            │
│  처리량 (Throughput / Bandwidth)                           │
│  ├── 단위 시간당 완료한 작업의 수                           │
│  ├── 관리자 관점 (서버, 데이터센터)                         │
│  └── 클수록 좋음                                           │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**성능의 상대적 비교**:
```
성능_X = 1 / 실행시간_X

X가 Y보다 n배 빠르다:
성능_X / 성능_Y = 실행시간_Y / 실행시간_X = n
```

### CPU 시간 (CPU Time)

프로그램 실행에 CPU가 실제로 사용한 시간:

```
┌────────────────────────────────────────────────────────────┐
│                    실행 시간의 구성                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  경과 시간 (Elapsed Time / Wall Clock Time)                │
│  = CPU Time + I/O 대기 + 다른 프로세스 실행 시간 + ...      │
│                                                            │
│  CPU 시간 (CPU Time)                                       │
│  = User CPU Time + System CPU Time                        │
│                                                            │
│  User CPU Time: 프로그램 코드 실행                          │
│  System CPU Time: OS 커널 코드 실행                         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 클록 사이클과 클록 속도

```
클록 주기 (Clock Period):
하나의 클록 사이클에 걸리는 시간 (초)

클록 속도 (Clock Rate / Frequency):
초당 클록 사이클 수 (Hz)

Clock Rate = 1 / Clock Period

예: 3 GHz CPU
- Clock Rate = 3 × 10⁹ Hz (초당 30억 사이클)
- Clock Period = 1 / (3 × 10⁹) ≈ 0.33 ns
```

```
      ┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐
클록  │  │  │  │  │  │  │  │  │  │
   ───┘  └──┘  └──┘  └──┘  └──┘  └──

      │←─────────────────────────→│
              1초 = n 사이클

      │←→│
      Clock Period (주기)
```

### CPU 성능 공식

**기본 공식**:
```
CPU Time = Clock Cycles × Clock Period
         = Clock Cycles / Clock Rate
```

**명령어 기반 공식**:
```
CPU Time = Instruction Count × CPI × Clock Period
         = (Instruction Count × CPI) / Clock Rate

여기서:
- Instruction Count: 실행되는 명령어 수
- CPI (Cycles Per Instruction): 명령어당 평균 클록 사이클 수
```

```
┌────────────────────────────────────────────────────────────┐
│                CPU Time 분해                               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  CPU Time = Instruction Count × CPI × Clock Period        │
│                    │            │          │               │
│                    │            │          │               │
│                    ▼            ▼          ▼               │
│              컴파일러/ISA   아키텍처    하드웨어 기술        │
│              알고리즘       구현                           │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### CPI (Cycles Per Instruction)

명령어 하나를 실행하는 데 필요한 평균 클록 사이클 수:

```
             n
CPI = (1/IC) × Σ (CPIᵢ × ICᵢ)
            i=1

여기서:
- IC: 전체 명령어 수
- CPIᵢ: i번째 명령어 종류의 CPI
- ICᵢ: i번째 명령어 종류의 실행 횟수
```

**명령어 종류별 CPI 예시**:
```
┌──────────────────┬─────────┬────────────┐
│ 명령어 종류       │ CPI     │ 빈도       │
├──────────────────┼─────────┼────────────┤
│ ALU 연산         │ 1       │ 40%        │
│ Load/Store       │ 3       │ 30%        │
│ Branch           │ 2       │ 20%        │
│ Multiply         │ 4       │ 10%        │
└──────────────────┴─────────┴────────────┘

평균 CPI = 0.4×1 + 0.3×3 + 0.2×2 + 0.1×4
         = 0.4 + 0.9 + 0.4 + 0.4
         = 2.1
```

### MIPS와 FLOPS

#### MIPS (Million Instructions Per Second)
```
MIPS = Instruction Count / (Execution Time × 10⁶)
     = Clock Rate / (CPI × 10⁶)
```

**MIPS의 문제점**:
- ISA에 따라 명령어 수가 다름 (CISC vs RISC)
- 같은 작업에 다른 명령어 수 필요
- 명령어 종류 무시 (NOP도 1 instruction)

#### FLOPS (Floating-Point Operations Per Second)
```
과학/공학 계산에서 주로 사용:
- MFLOPS: 10⁶ FLOPS
- GFLOPS: 10⁹ FLOPS
- TFLOPS: 10¹² FLOPS
- PFLOPS: 10¹⁵ FLOPS
- EFLOPS: 10¹⁸ FLOPS
```

### Amdahl's Law (암달의 법칙)

시스템 일부를 개선했을 때 전체 성능 향상의 한계를 설명:

```
                            1
Speedup_overall = ─────────────────────────────
                  (1 - F) + F/Speedup_enhanced

여기서:
- F: 개선되는 부분의 비율 (0 ≤ F ≤ 1)
- Speedup_enhanced: 해당 부분의 성능 향상 배율
```

**예시**:
```
프로그램의 80%가 병렬화 가능, 나머지 20%는 순차적
병렬 부분을 10배 빠르게 개선하면:

Speedup = 1 / (0.2 + 0.8/10)
        = 1 / (0.2 + 0.08)
        = 1 / 0.28
        ≈ 3.57배

→ 80%를 10배 빠르게 해도 전체는 3.57배만 향상!
```

```
┌────────────────────────────────────────────────────────────┐
│                    Amdahl's Law 시각화                     │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  원본:    |========순차========|===병렬===|                │
│                 20%                80%                     │
│                                                            │
│  개선 후:  |========순차========|=|       (병렬 10배)       │
│                 20%            8%                          │
│                                                            │
│  → 순차 부분이 병목이 됨 (최대 5배까지만 가능)               │
│                                                            │
│  lim(Speedup) as Speedup_enhanced → ∞ = 1/(1-F) = 5       │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Gustafson's Law (구스타프슨의 법칙)

암달의 법칙의 대안적 관점 - 문제 크기를 증가시키는 경우:

```
Speedup = (1 - F) + F × N

여기서:
- F: 병렬화 가능한 비율
- N: 프로세서 수
```

**암달 vs 구스타프슨**:
```
암달:     고정된 문제 크기, 프로세서 추가
          → 순차 부분이 병목

구스타프슨: 고정된 시간, 문제 크기 확장
           → 병렬 부분이 커져서 효과 증가
```

### Iron Law of Processor Performance

프로세서 성능을 결정하는 세 가지 요소:

```
              Instructions   Cycles       Seconds
CPU Time  =  ──────────── × ───────── × ─────────
               Program     Instruction    Cycle

           = Instruction Count × CPI × Clock Period


┌──────────────────┬────────────────────────────────────────┐
│ 요소              │ 영향을 주는 것                          │
├──────────────────┼────────────────────────────────────────┤
│ Instruction Count│ 알고리즘, 컴파일러, ISA                 │
├──────────────────┼────────────────────────────────────────┤
│ CPI              │ 마이크로아키텍처, ISA, 메모리 시스템      │
├──────────────────┼────────────────────────────────────────┤
│ Clock Period     │ 마이크로아키텍처, 반도체 기술            │
└──────────────────┴────────────────────────────────────────┘
```

**트레이드오프**:
- 클록 속도↑ → CPI↑ (파이프라인 깊어짐)
- CISC: Instruction Count↓, CPI↑
- RISC: Instruction Count↑, CPI↓

### 벤치마크 (Benchmarks)

실제 성능 비교를 위한 표준 프로그램:

```
┌────────────────────────────────────────────────────────────┐
│                    주요 벤치마크                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  SPEC CPU:                                                 │
│  ├── SPECint: 정수 연산 (컴파일러, 압축, 게임 AI 등)         │
│  └── SPECfp: 부동소수점 연산 (과학 계산, 시뮬레이션)         │
│                                                            │
│  SPEC JBB: Java 비즈니스 벤치마크                           │
│  TPC-C: 데이터베이스 트랜잭션 처리                          │
│  LINPACK: 선형 대수 연산 (슈퍼컴퓨터 순위)                   │
│  MLPerf: 머신러닝 워크로드                                  │
│  CoreMark: 임베디드 시스템용                                │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## 실무 적용

### 벤치마킹 시 주의사항

```java
// Java에서 마이크로벤치마킹 시 주의점

// 나쁜 예: JIT 워밍업 무시
public void badBenchmark() {
    long start = System.nanoTime();
    doWork();
    long end = System.nanoTime();
    System.out.println("Time: " + (end - start));
}

// 좋은 예: 워밍업 후 측정, JMH 사용
@Benchmark
@Warmup(iterations = 5)
@Measurement(iterations = 10)
public void goodBenchmark() {
    doWork();
}
```

### 성능 분석 시 고려사항

```python
# 알고리즘 비교 시 성능 지표 활용

# 단순히 실행 시간만 보면 안 됨
# 입력 크기, 데이터 분포, 하드웨어 특성 고려

import time
import statistics

def benchmark(func, *args, iterations=100):
    times = []
    for _ in range(iterations):
        start = time.perf_counter()
        func(*args)
        end = time.perf_counter()
        times.append(end - start)

    return {
        'mean': statistics.mean(times),
        'median': statistics.median(times),
        'stdev': statistics.stdev(times),
        'min': min(times),
        'max': max(times)
    }
```

### 성능 개선 우선순위

```
암달의 법칙을 적용한 최적화 우선순위:

1. 프로파일링으로 병목 지점 파악
   → 실행 시간의 큰 비중을 차지하는 부분 찾기

2. 병목 부분 최적화
   → 작은 비중의 최적화는 효과 미미

3. 반복 측정
   → 병목이 이동했는지 확인
```

## 참고 자료

- Hennessy & Patterson, "Computer Architecture: A Quantitative Approach" Chapter 1
- Patterson & Hennessy, "Computer Organization and Design" Chapter 1
- SPEC: https://www.spec.org/
- Gene Amdahl, "Validity of the Single Processor Approach to Achieving Large Scale Computing Capabilities" (1967)
- John Gustafson, "Reevaluating Amdahl's Law" (1988)
