# Programming Language 공통 질문

> 카테고리: 프로그래밍 언어
> [<- 면접 질문 목록으로 돌아가기](../interview.md)

---

## 언어별 질문

- [Java 질문 보기](./java.md)
- [JavaScript / TypeScript 질문 보기](./javascript.md)
- [Python 질문 보기](./python.md)
- [Go 질문 보기](./go.md)

---

## 공통 질문

### LANG-001

객체지향 프로그래밍(OOP)의 4가지 특징을 설명해주세요.

### LANG-002

SOLID 원칙에 대해 설명해주세요.

### LANG-003

함수형 프로그래밍의 특징과 장점은 무엇인가요?

### LANG-004

순수 함수(Pure Function)란 무엇인가요?

### LANG-005

불변성(Immutability)의 중요성과 구현 방법은 무엇인가요?

### LANG-006

동시성(Concurrency)과 병렬성(Parallelism)의 차이점은 무엇인가요?

### LANG-007

Race Condition과 Deadlock에 대해 설명해주세요.

### LANG-008

동기(Synchronous)와 비동기(Asynchronous)의 차이점은 무엇인가요?

### LANG-009

블로킹(Blocking)과 논블로킹(Non-blocking)의 차이점은 무엇인가요?

### LANG-010

컴파일 언어와 인터프리터 언어의 차이점은 무엇인가요?

### LANG-011

JIT(Just-In-Time) 컴파일러의 동작 원리를 설명해주세요.

### LANG-012

정적 타이핑과 동적 타이핑의 장단점은 무엇인가요?

### LANG-013

강타입과 약타입 언어의 차이점은 무엇인가요?

### LANG-014

Call by Value와 Call by Reference의 차이점을 설명해주세요.

### LANG-015

메모리 누수(Memory Leak)가 발생하는 원인과 방지 방법은 무엇인가요?

---

## 타입 이론 (Type Theory)

### LANG-016

곱 타입(Product Type)과 합 타입(Sum Type)이란 무엇이며, 각각의 사용 예시를 설명해주세요.

### LANG-017

Null 참조가 "10억 달러 실수(Billion Dollar Mistake)"라고 불리는 이유와, 현대 언어들이 이를 해결하기 위해 채택한 방식(Optional, Nullable 타입 등)을 설명해주세요.

### LANG-018

타입 추론(Type Inference)이란 무엇이며, 컴파일러는 어떻게 타입을 추론하나요?

### LANG-019

제네릭(Generic)과 공변성(Covariance), 반공변성(Contravariance)에 대해 설명해주세요.

### LANG-020

구조적 타이핑(Structural Typing)과 명목적 타이핑(Nominal Typing)의 차이점은 무엇인가요?

### LANG-021

의존 타입(Dependent Type)이란 무엇이며, 어떤 언어에서 지원하나요?

### LANG-022

타입 시스템의 건전성(Soundness)이란 무엇이며, 왜 중요한가요?

---

## 에러 처리 (Error Handling)

### LANG-023

Checked Exception과 Unchecked Exception의 차이점과 각각의 장단점은 무엇인가요?

### LANG-024

예외(Exception) 기반 에러 처리와 Result/Either 타입 기반 에러 처리의 차이점과 트레이드오프를 설명해주세요.

### LANG-025

Go 언어의 에러 처리 방식(error 반환)과 Java의 예외 처리 방식을 비교해주세요.

### LANG-026

패닉(Panic)과 예외(Exception)의 차이점은 무엇이며, 언제 각각을 사용해야 하나요?

### LANG-027

에러 처리에서 Fail-Fast 전략과 Fail-Safe 전략의 차이점을 설명해주세요.

---

## 다형성 (Polymorphism)

### LANG-028

다형성의 종류(Ad-hoc, Parametric, Subtype Polymorphism)를 설명하고 각각의 예시를 들어주세요.

### LANG-029

오버로딩(Overloading)과 오버라이딩(Overriding)의 차이점과 각각이 어떤 다형성에 해당하는지 설명해주세요.

### LANG-030

Duck Typing과 Structural Subtyping의 관계를 설명해주세요.

### LANG-031

상속(Inheritance)보다 컴포지션(Composition)을 선호해야 하는 이유를 설명해주세요.

### LANG-032

인터페이스와 추상 클래스의 차이점, 그리고 Go나 Rust처럼 상속 없이 다형성을 구현하는 방법을 설명해주세요.

### LANG-033

패턴 매칭(Pattern Matching)과 완전 패턴 매칭(Exhaustive Pattern Matching)이 제공하는 안전성 이점은 무엇인가요?

---

## 메타프로그래밍 (Metaprogramming)

### LANG-034

메타프로그래밍이란 무엇이며, 어떤 상황에서 유용한가요?

### LANG-035

리플렉션(Reflection)의 장단점과 성능 영향에 대해 설명해주세요.

### LANG-036

매크로(Macro)란 무엇이며, 함수와 어떤 차이가 있나요? (C 전처리기 매크로, Rust 매크로, Lisp 매크로 비교)

### LANG-037

코드 생성(Code Generation)과 런타임 리플렉션의 트레이드오프를 설명해주세요.

### LANG-038

어노테이션/데코레이터(Annotation/Decorator)는 어떻게 동작하며, 메타프로그래밍에서 어떤 역할을 하나요?

### LANG-039

AST(Abstract Syntax Tree)란 무엇이며, 컴파일러와 메타프로그래밍에서 어떻게 활용되나요?

---

## 고수준 언어와 저수준 언어

### LANG-040

고수준 언어와 저수준 언어의 차이점과 트레이드오프를 설명해주세요.

### LANG-041

가상 머신(VM) 기반 언어(Java, C#)와 네이티브 컴파일 언어(C, Rust)의 장단점을 비교해주세요.

### LANG-042

가비지 컬렉션(GC)과 수동 메모리 관리, Rust의 소유권 시스템을 비교해주세요.

### LANG-043

Zero-Cost Abstraction이란 무엇이며, 어떤 언어에서 이를 지원하나요?

### LANG-044

FFI(Foreign Function Interface)란 무엇이며, 서로 다른 언어 간 상호 운용 시 고려해야 할 점은 무엇인가요?

---

## 동시성 프로그래밍 (Concurrent Programming)

### LANG-045

공유 메모리 기반 동시성(Shared Memory)과 메시지 전달 기반 동시성(Message Passing)의 차이점을 설명해주세요.

### LANG-046

Actor 모델이란 무엇이며, 기존 스레드 기반 동시성과 어떻게 다른가요?

### LANG-047

CSP(Communicating Sequential Processes)란 무엇이며, Go의 채널과 어떤 관계가 있나요?

### LANG-048

Futures/Promises와 async/await의 동작 원리를 설명해주세요.

### LANG-049

코루틴(Coroutine)과 스레드(Thread)의 차이점, 그리고 협력적 멀티태스킹에 대해 설명해주세요.

### LANG-050

동시성 프로그래밍에서 락(Lock) 없이 안전하게 데이터를 공유하는 방법(Lock-free, Immutability, STM 등)을 설명해주세요.

### LANG-051

Green Thread와 OS Thread의 차이점, 그리고 M:N 스케줄링에 대해 설명해주세요.

### LANG-052

타입 시스템을 통해 동시성 안전성을 컴파일 타임에 보장하는 방법에 대해 설명해주세요. (예: Rust의 Send/Sync, 불변성 기반 접근 등)
