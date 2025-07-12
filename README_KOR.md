# quickpool

![build status](https://github.com/tnagler/quickpool/actions/workflows/main.yml/badge.svg?branch=main)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/ed2deb06d4454ab3b488536426ec3066)](https://www.codacy.com/gh/tnagler/quickpool/dashboard?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=tnagler/quickpool&amp;utm_campaign=Badge_Grade)
[![codecov](https://codecov.io/gh/tnagler/quickpool/branch/main/graph/badge.svg?token=ERPXZC8378)](https://codecov.io/gh/tnagler/quickpool)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Documentation](https://img.shields.io/website/http/tnagler.github.io/quickpool.svg)](https://tnagler.github.io/quickpool/)
[![DOI](https://zenodo.org/badge/427536398.svg)](https://zenodo.org/badge/latestdoi/427536398)

> C++11에서의 빠르고 쉬운 병렬 컴퓨팅
## Why quickpool?

### Developer friendly

자유로운 [single header file](https://github.com/tnagler/quickpool/blob/parallel-for/quickpool.hpp)만으로 구성되어있으며,
C++11만 있으면 됩니다.
`quickpool.hpp`를 프로젝트 안에 넣기만 하면 됩니다.

### User friendly API

* [`push(f, args...)`](https://tnagler.github.io/quickpool/namespacequickpool.html#affc41895dab281715c271aca3649e830) 반환값 없이 `f(args...)`를 실행하는 작업을 예약
* [`async(f, args...)`](https://tnagler.github.io/quickpool/namespacequickpool.html#a10575809d24ead3716e312585f90a94a) `f(args...)`를 실행하는 작업을 예약하고, [`std::future`](https://en.cppreference.com/w/cpp/thread/future)를 반환
* [`wait()`](https://tnagler.github.io/quickpool/namespacequickpool.html#a086671a25cc4f207112bc82a00688301) 예약된 모든 작업이 끝날 때까지 대기
* [`parallel_for(b, e, f)`](https://tnagler.github.io/quickpool/namespacequickpool.html#aa72b140a64eabe34cd9302bab837c24c) `b`이상 `e`미만의 모든 i에 대해 `f(i)`를 실행
* [`parallel_for_each(x, f)`](https://tnagler.github.io/quickpool/namespacequickpool.html#aeb91fe18664b8d06523aba081174abe3) `x`의 모든 원소(반복자)에 대해 `f(*it)`를 병렬로 실행

반복문은 중첩하여 사용할 수 있으며, 아래의 예시를 참고하면 됩니다.
모든 함수는 한 번만 생성되는 global thread pool로 분배되며, 이 pool은 시스템의 코어 수만큼 thread를 가집니다.
선택적으로, 앞서 나온 함수들로 로컬 `ThreadPool`을 직접 제작할 수 있습니다. 
자세한 내용은 [API 문서](https://tnagler.github.io/quickpool/)을 참고하세요.

### Cutting edge algorithms(최신 알고리즘)

모든 스케줄링은 [work stealing](https://en.wikipedia.org/wiki/Work_stealing) 기법을 사용하며, [캐시 정렬 원자 연산(cache-aligned atomic)](https://github.com/tnagler/aligned_atomic)으로 동기화됩니다.
* [`work stealing`]: 각 프로세서가 자신의 작업 큐를 우선 처리하다가, 작업이 없으면 다른 큐에서 작업을 가져오는 방식의 스케줄링 기법
* [`cache-aligned atomic`]: 멀티스레드 환경에서 여러 스레드가 각각 다른 변수를 접근할 때 발생할 수 있는 `false sharing` 문제를 방지하기 위해, 원자 변수를 캐시 라인 크기에 정렬하고 패딩하는 최적화 기법dlqslek.
* [`false sharing`]: 서로 다른 스레드가 같은 캐시 라인에 위치한 변수에 접근해 불필요한 캐시 무효화와 성능 저하를 유발하는 현상

스레드 풀은 각 작업자에게 작업 큐를 할당합니다. 작업자들은 먼저 자신의 큐에 있는 작업을 처리한 뒤, 다른 작업자의 큐에서 작업을 훔쳐옵니다.
이러한 알고리즘은 표준적인 상황(오직 하나의 스레드만이 작업을 풀에 추가하는 경우)일 때 [lock-free](https://en.wikipedia.org/wiki/Non-blocking_algorithm) 방식으로 동작합니다.
* [`lock-free`]: 여러 스레드가 동시에 접근해도 락(잠금) 없이 안전하게 동작하는 알고리즘

병렬 반복문에서는 각 작업자들에게 반복 범위의 일부를 할당합니다.
작업자가 자신의 범위를 모두 처리한다면, 다른 작업자의 범위 중 절반을 훔쳐와 처리합니다. 이 방식은 작업 부하를 균형 있게 분산시키며, 경합(contention)이 발생하는 작업 훔치기(steal)은 로그(logarithmic) 수준으로 최소화됩니다.
이 알고리즘은 [double-wide compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap#Extensions) 연산을 사용하는데, 이는 대부분의 최신 프로세서 아키텍처에서 lock-free로 동작합니다.
* [`double-wide compare-and-swap`]: 두 개의 값을 한 번에 비교해서, 둘 다 예상한 값일 때만 동시에 새로운 값으로 바꿔주는 원자적 연산

quickpool에서 `double-wide compare-and-swap` 알고리즘을 사용하는 이유는, 병렬 반복문에서 각 작업자가 작업 범위를 나누고 훔쳐갈 때
두 개의 값(예: 작업 범위의 시작과 끝)을 동시에 원자적으로 갱신해야
여러 스레드가 동시에 접근해도 데이터 충돌이나 일관성 문제가 발생하지 않기 때문입니다. 따라서 lock-free인 double-wide compare-and-swap을 사용하는 것입니다.

## Examples


### Thread pool

```cpp
push([] { /* some work */ });
wait(); // 현재 작업이 끝날 때까지 기다림

auto f = async([] { return 1 + 1; }); // 결과를 받을 future 객체 생성
// 다른 작업 수행...
auto result = f.get(); // 작업이 끝날 때까지 기다렸다가 결과를 반환
```

[`push()`](https://tnagler.github.io/quickpool/namespacequickpool.html#affc41895dab281715c271aca3649e830)와 [`async()`](https://tnagler.github.io/quickpool/namespacequickpool.html#a10575809d24ead3716e312585f90a94a) 함수 모두, 함수에 전달할 추가 인자들과 함께 호출할 수 있습니다. 아래의 예시를 참고하세요.

```cpp
auto work = [] (const std::string& title, int i) { 
  std::cout << title << ": " << i << std::endl; 
};
push(work, "first title", 5);
async(work, "other title", 99);
wait();
```

### Parallel loops

기존의 순차 반복문은 쉽게 병렬화할 수 있습니다:
```cpp
std::vector<double> x(10, 1);

// sequential version
for (int i = 0; i < x.size(); ++i) 
  x[i] *= 2;

// parallel version
parallel_for(0, x.size(), [&] (int i) { x[i] *= 2; };


// sequential version
for (auto& xx : x) 
  xx *= 2;

// parallel version
parallel_for_each(x, [] (double& xx) { xx *= 2; };
```
반복문 함수들은 모든 작업이 끝날 때까지 자동으로 대기하지만,
이는 메인 스레드에서 호출될 때에만 적용됩니다.

### Nested parallel loops

내부 반복문에서 기다릴 필요가 없다면, 병렬 for 반복문을 중첩해서 사용하는 것도 가능합니다.
```cpp
std::vector<double> x(10, 1);

// sequential version
for (int i = 0; i < x.size(); ++i) {
  for (int j = 4; j < 9; j++) {
    x[i] *= j;
  }
}

// parallel version
parallel_for(0, x.size(), [&] (int i) { 
  // *중요*: i를 복사로 캡처해야 안전합니다.
  // 내부 반복문이 끝날 때까지 외부 반복문이 기다리지 않습니다.
  parallel_for(4, 9, [&, i] (int j) {  x[i] *= j; }); // 내부 반복문은 끝나기를 기다리지 않음
}; // 외부 반복문은 모든 i 작업이 끝날 때까지 기다림
```

내부 반복문이 모두 끝나야만 외부 반복문의 다음 작업이 진행되는 순차 for문과 달리, 
병렬 반복문(parallel_for)에서는 외부와 내부 반복문이 서로 독립적으로 동시에 처리됩니다.
즉, 외부 반복문의 각 작업이 병렬로 실행되는 동시에, 각 작업마다 내부 반복문도 별도로 병렬 실행됩니다.
이러한 구조는 반복 간 데이터 의존성이 없을 때만 안전하게 사용할 수 있으며, 내부 반복문이 끝나기를 반드시 기다릴 필요가 없는 상황에 적합합니다.

### Local thread pool

[`ThreadPool`](https://tnagler.github.io/quickpool/classquickpool_1_1ThreadPool.html)을 사용하면 원하는 개수의 스레드로 구성된 로컬(개별) 스레드 풀을 직접 만들 수 있습니다.
이 풀은 전역 스레드 풀과 별도로 동작하며, 풀 객체가 소멸할 때 내부의 모든 스레드가 자동으로 정리(join)됩니다.

```cpp
ThreadPool pool(2); // 2개의 스레드로 구성된 풀 생성

pool.push([] {});
pool.async([] {});
pool.wait();

pool.parallel_for(2, 5, [&] (int i) {});
auto x = std::vector<double>{10};
pool.parallel_for_each(x, [] (double& xx) {});
```
