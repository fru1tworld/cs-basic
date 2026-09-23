# TLB와 큰 페이지

## TLB란?

가상 주소를 물리 주소로 바꾸려고 매번 페이지 테이블을 따라가면 메모리 접근이 추가된다. 여기서 다루는 4단계 Page Table Walk는 4번의 메모리 접근과 약 100-400 cycles가 든다.

TLB는 페이지 테이블 엔트리를 캐싱해 이 과정을 줄이는 하드웨어다. 필요한 변환이 TLB에 있으면 Page Table Walk 대신 1-2 cycles의 TLB Lookup으로 물리 주소를 얻는다.

## TLB 구조

- TLB Entry:
- Valid, ASID, Virtual Page Number, PFN, Permission Bits
- TLB 타입:
- Fully Associative: 어디든 저장 가능 (비쌈, 작은 TLB)
- Set-Associative: 일부 위치에 저장 (일반적)
- Direct-Mapped: 정해진 위치에만 저장 (저렴, 충돌 많음)

## TLB 미스 처리

```c
// 소프트웨어 관리 TLB (MIPS 스타일) 개념적 구현
#include <stdint.h>

#define TLB_SIZE 64

struct tlb_entry {
    uint64_t vpn;       // Virtual Page Number
    uint64_t pfn;       // Physical Frame Number
    uint8_t  valid;
    uint8_t  asid;      // Address Space ID
    uint8_t  flags;     // Read, Write, Execute, Global
};

struct tlb_entry tlb[TLB_SIZE];

// TLB Lookup
int tlb_lookup(uint64_t vpn, uint8_t asid, uint64_t *pfn) {
    for (int i = 0; i < TLB_SIZE; i++) {
        if (tlb[i].valid &&
            (tlb[i].vpn == vpn) &&
            (tlb[i].flags & TLB_GLOBAL || tlb[i].asid == asid)) {
            *pfn = tlb[i].pfn;
            return 1;  // TLB Hit
        }
    }
    return 0;  // TLB Miss
}

// TLB Miss Handler (페이지 테이블 워킹)
void tlb_miss_handler(uint64_t faulting_addr, uint8_t asid) {
    uint64_t vpn = faulting_addr >> 12;
    uint64_t pfn;

    // 페이지 테이블에서 PTE 조회
    if (!page_table_walk(vpn, &pfn)) {
        // 페이지 폴트 - 페이지가 메모리에 없음
        handle_page_fault(faulting_addr);
        return;
    }

    // TLB 엔트리 추가 (랜덤 또는 LRU 교체)
    int victim = random() % TLB_SIZE;
    tlb[victim].vpn = vpn;
    tlb[victim].pfn = pfn;
    tlb[victim].valid = 1;
    tlb[victim].asid = asid;
}
```

## ASID (Address Space Identifier)

주소 공간을 구분하지 않는 TLB는 문맥 교환 시 기존 엔트리를 무효화해야 한다. 그러면 새 프로세스가 실행될 때 TLB Miss가 몰릴 수 있다.

ASID는 엔트리의 주소 공간을 구분해 이런 플러시를 피한다. 문맥 교환 시 ASID 레지스터를 바꾸면 다른 ASID의 엔트리는 일치하지 않는 것으로 처리하고, 같은 프로세스로 돌아왔을 때 남아 있는 엔트리를 재활용할 수 있다.

## TLB 성능 측정

```python
import time
import random
import mmap
import os

def measure_tlb_performance():
    """TLB 성능 측정 (간접적)"""
    page_size = 4096

    # 다양한 페이지 수로 테스트
    test_sizes = [16, 32, 64, 128, 256, 512, 1024, 2048]
    results = []

    for num_pages in test_sizes:
        # 메모리 할당
        size = num_pages * page_size
        buf = mmap.mmap(-1, size, mmap.MAP_PRIVATE | mmap.MAP_ANONYMOUS,
                        mmap.PROT_READ | mmap.PROT_WRITE)

        # 각 페이지 터치 (TLB 로드)
        for i in range(num_pages):
            buf[i * page_size] = 0

        # 랜덤 접근 시간 측정
        iterations = 1000000
        indices = [random.randint(0, num_pages - 1) * page_size
                   for _ in range(iterations)]

        start = time.perf_counter()
        for idx in indices:
            _ = buf[idx]
        end = time.perf_counter()

        avg_time_ns = (end - start) / iterations * 1e9
        results.append((num_pages, avg_time_ns))

        buf.close()

    print("=== TLB Performance Test ===")
    print("Pages\tAvg Access Time (ns)")
    print("-" * 30)
    for pages, time_ns in results:
        marker = " <-- Possible TLB thrashing" if pages > 64 and time_ns > results[0][1] * 1.5 else ""
        print(f"{pages}\t{time_ns:.2f}{marker}")

if __name__ == "__main__":
    # 실제 실행 시 Linux에서만 작동
    try:
        measure_tlb_performance()
    except Exception as e:
        print(f"Test failed: {e}")
        print("Note: This test works best on Linux systems")
```

## Huge Pages

- TLB 효율성을 높이기 위해 큰 페이지를 사용함.

```bash
# Linux에서 Huge Pages 설정

# 현재 설정 확인
cat /proc/meminfo | grep -i huge

# Huge Pages 할당 (root 권한)
echo 1024 > /proc/sys/vm/nr_hugepages

# Transparent Huge Pages (THP) 상태
cat /sys/kernel/mm/transparent_hugepage/enabled
```

```c
// Huge Page 할당 (C)
#include <sys/mman.h>
#include <stdio.h>

#define HUGE_PAGE_SIZE (2 * 1024 * 1024)  // 2MB

int main() {
    // 2MB Huge Page 할당
    void *ptr = mmap(NULL, HUGE_PAGE_SIZE,
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                     -1, 0);

    if (ptr == MAP_FAILED) {
        perror("mmap with MAP_HUGETLB failed");
        return 1;
    }

    printf("Huge page allocated at: %p\n", ptr);

    // 사용...

    munmap(ptr, HUGE_PAGE_SIZE);
    return 0;
}
```

## 참고 자료

- [OSTEP - Virtualization (Memory)](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [Intel 64 and IA-32 Architectures Software Developer's Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Linux Kernel Memory Management](https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html)

- [What Every Programmer Should Know About Memory - Ulrich Drepper](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- [Understanding the Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/)

- `/proc/[pid]/maps` - 프로세스 메모리 맵
- `/proc/meminfo` - 시스템 메모리 정보
- `vmstat`, `free`, `top` - 메모리 모니터링
- `perf` - TLB 미스 등 성능 분석

- 문서 작성일: 2024년 12월
- 참고: 이 문서는 OSTEP, Linux 커널 문서, Intel 아키텍처 매뉴얼을 기반으로 작성되었음.

## 관련 학습

- [페이징과 세그먼테이션](02-페이징과-세그먼테이션.md)
- [페이지 교체와 프레임 할당](04-페이지-교체와-프레임-할당.md)
- [가상 메모리와 주소 변환](01-가상-메모리와-주소-변환.md)
- [단편화와 메모리 할당](05-단편화와-메모리-할당.md)
