# 함수형 프로그래밍 (Functional Programming)

## 목차
1. [함수형 프로그래밍 개요](#1-함수형-프로그래밍-개요)
2. [불변성 (Immutability)](#2-불변성-immutability)
3. [순수 함수와 부수 효과](#3-순수-함수와-부수-효과)
4. [고차 함수와 클로저](#4-고차-함수와-클로저)
5. [map, filter, reduce](#5-map-filter-reduce)
6. [모나드 개념](#6-모나드-개념)
7. [함수형 프로그래밍 실전](#7-함수형-프로그래밍-실전)
---

## 1. 함수형 프로그래밍 개요

### 1.1 패러다임 비교

```
┌────────────────────────────────────────────────────────────────┐
│                    프로그래밍 패러다임                           │
├────────────────────────┬───────────────────────────────────────┤
│       명령형            │              선언형                    │
│   (Imperative)         │          (Declarative)                │
├────────────────────────┼─────────────────┬─────────────────────┤
│                        │     함수형       │       논리형         │
│  - 절차적 프로그래밍    │  (Functional)   │    (Logic)          │
│  - 객체지향 프로그래밍   │                 │    예: Prolog       │
│                        │  예: Haskell,    │                     │
│  예: C, Java           │  Scala, Clojure │                     │
└────────────────────────┴─────────────────┴─────────────────────┘
```

### 1.2 명령형 vs 함수형

```python
# 명령형: "어떻게(How)" - 상태 변경의 연속
numbers = [1, 2, 3, 4, 5]
result = []
for num in numbers:
    if num % 2 == 0:
        result.append(num * 2)
# result = [4, 8]

# 함수형: "무엇을(What)" - 데이터 변환의 선언
result = list(
    map(lambda x: x * 2,
        filter(lambda x: x % 2 == 0, numbers))
)
# result = [4, 8]

# 더 Pythonic한 함수형 스타일
result = [x * 2 for x in numbers if x % 2 == 0]
```

```java
// Java - 명령형 vs 함수형 (Stream API)
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// 명령형
List<Integer> result1 = new ArrayList<>();
for (Integer num : numbers) {
    if (num % 2 == 0) {
        result1.add(num * 2);
    }
}

// 함수형
List<Integer> result2 = numbers.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * 2)
    .collect(Collectors.toList());
```

### 1.3 함수형 프로그래밍의 핵심 원칙

| 원칙 | 설명 |
|------|------|
| 불변성 (Immutability) | 데이터는 생성 후 변경되지 않음 |
| 순수 함수 (Pure Functions) | 동일 입력 → 동일 출력, 부수 효과 없음 |
| 일급 함수 (First-class Functions) | 함수를 값처럼 전달, 반환 가능 |
| 고차 함수 (Higher-order Functions) | 함수를 인자로 받거나 반환하는 함수 |
| 참조 투명성 (Referential Transparency) | 표현식을 결과값으로 대체 가능 |

---

## 2. 불변성 (Immutability)

### 2.1 가변 vs 불변

```python
# 가변 객체 - 문제 발생 가능
class MutablePoint:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def move(self, dx, dy):
        self.x += dx  # 상태 변경!
        self.y += dy

p1 = MutablePoint(0, 0)
p2 = p1  # 같은 객체 참조
p1.move(5, 5)
print(p2.x, p2.y)  # 5, 5 - 의도치 않은 변경!

# 불변 객체 - 안전
from dataclasses import dataclass

@dataclass(frozen=True)  # 불변 객체
class ImmutablePoint:
    x: int
    y: int

    def move(self, dx, dy):
        return ImmutablePoint(self.x + dx, self.y + dy)  # 새 객체 반환

p1 = ImmutablePoint(0, 0)
p2 = p1.move(5, 5)  # 새 객체 생성
print(p1.x, p1.y)  # 0, 0 - 원본 유지
print(p2.x, p2.y)  # 5, 5 - 새 객체
```

```java
// Java - 불변 객체 패턴
public final class Money {
    private final int amount;
    private final String currency;

    public Money(int amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }

    // 상태 변경 대신 새 객체 반환
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount + other.amount, this.currency);
    }

    public Money multiply(int factor) {
        return new Money(this.amount * factor, this.currency);
    }

    // Getter만 제공, Setter 없음
    public int getAmount() { return amount; }
    public String getCurrency() { return currency; }
}

// Java 16+ Record (불변 객체를 위한 간결한 문법)
public record MoneyRecord(int amount, String currency) {
    public MoneyRecord add(MoneyRecord other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new MoneyRecord(this.amount + other.amount, this.currency);
    }
}
```

### 2.2 컬렉션의 불변성

```java
// Java - 불변 컬렉션
import java.util.List;
import java.util.Map;
import java.util.Set;

// 불변 컬렉션 생성 (Java 9+)
List<String> immutableList = List.of("a", "b", "c");
Set<Integer> immutableSet = Set.of(1, 2, 3);
Map<String, Integer> immutableMap = Map.of("one", 1, "two", 2);

// 수정 시도 시 UnsupportedOperationException
// immutableList.add("d");  // 예외 발생!

// 기존 컬렉션을 불변으로 래핑
List<String> original = new ArrayList<>(List.of("a", "b"));
List<String> unmodifiable = Collections.unmodifiableList(original);

// 방어적 복사
public class SafeContainer {
    private final List<String> items;

    public SafeContainer(List<String> items) {
        this.items = List.copyOf(items);  // 방어적 복사
    }

    public List<String> getItems() {
        return items;  // 이미 불변이므로 안전
    }
}
```

```python
# Python - 불변 컬렉션
from typing import Tuple, FrozenSet

# tuple (불변 리스트)
immutable_list: Tuple[int, ...] = (1, 2, 3)
# immutable_list[0] = 10  # TypeError!

# frozenset (불변 set)
immutable_set: FrozenSet[int] = frozenset({1, 2, 3})
# immutable_set.add(4)  # AttributeError!

# 불변 딕셔너리는 없지만 MappingProxyType 사용
from types import MappingProxyType

original = {"a": 1, "b": 2}
immutable_dict = MappingProxyType(original)
# immutable_dict["c"] = 3  # TypeError!
```

### 2.3 Persistent Data Structures

```
일반적인 수정 (Destructive Update):
┌───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │  원본
└───┴───┴───┴───┴───┘
         ↓ arr[2] = 10
┌───┬───┬────┬───┬───┐
│ 1 │ 2 │ 10 │ 4 │ 5 │  수정됨 (원본 파괴)
└───┴───┴────┴───┴───┘


Persistent (구조 공유):
         ┌───┐
         │ 2 │
         └─┬─┘
      ┌────┴────┐
    ┌─┴─┐     ┌─┴─┐
    │ 1 │     │ 3 │      원본 트리
    └───┘     └───┘

         ┌───┐
         │ 2 │──────────┐
         └─┬─┘          │  구조 공유
      ┌────┘            │
    ┌─┴─┐             ┌─┴─┐
    │ 1 │             │10 │  새 노드만 생성
    └───┘             └───┘
```

```python
# Python - pyrsistent 라이브러리 (Persistent Data Structures)
from pyrsistent import pvector, pmap, pset

# Persistent Vector
v1 = pvector([1, 2, 3])
v2 = v1.append(4)      # O(log n) - 새 벡터 반환
v3 = v1.set(0, 10)     # O(log n) - 새 벡터 반환

print(v1)  # pvector([1, 2, 3])    - 원본 유지
print(v2)  # pvector([1, 2, 3, 4])
print(v3)  # pvector([10, 2, 3])

# Persistent Map
m1 = pmap({"a": 1, "b": 2})
m2 = m1.set("c", 3)
m3 = m1.remove("a")

print(m1)  # pmap({'a': 1, 'b': 2})  - 원본 유지
print(m2)  # pmap({'a': 1, 'b': 2, 'c': 3})
```

---

## 3. 순수 함수와 부수 효과

### 3.1 순수 함수 (Pure Function)

```
순수 함수의 조건:
1. 동일 입력 → 동일 출력 (결정적, Deterministic)
2. 부수 효과 없음 (No Side Effects)
```

```python
# 순수 함수 예시
def add(a: int, b: int) -> int:
    return a + b  # 항상 같은 입력에 같은 출력

def calculate_tax(income: float, rate: float) -> float:
    return income * rate

def sort_list(items: list) -> list:
    return sorted(items)  # 원본 수정 없이 새 리스트 반환

# 비순수 함수 예시
total = 0
def impure_add(value: int) -> int:
    global total
    total += value  # 외부 상태 변경 (부수 효과)
    return total

counter = 0
def get_id() -> int:
    global counter
    counter += 1  # 매번 다른 결과 (비결정적)
    return counter

import random
def random_add(a: int) -> int:
    return a + random.randint(1, 10)  # 비결정적

import datetime
def get_greeting(name: str) -> str:
    hour = datetime.datetime.now().hour  # 외부 상태 의존
    if hour < 12:
        return f"Good morning, {name}"
    return f"Good afternoon, {name}"
```

### 3.2 부수 효과 (Side Effects)

```
부수 효과의 종류:
─────────────────
• 전역 변수 수정
• 입력 매개변수 수정
• 파일 읽기/쓰기
• 데이터베이스 연산
• 네트워크 요청
• 콘솔 출력
• 예외 발생
• 현재 시간/랜덤 값 사용
```

```java
// 부수 효과 격리 패턴
public class OrderService {
    private final OrderRepository repository;  // 부수 효과 담당
    private final OrderCalculator calculator;  // 순수 함수 담당

    public Order processOrder(OrderRequest request) {
        // 1. 입력 검증 (순수)
        ValidationResult validation = OrderValidator.validate(request);
        if (!validation.isValid()) {
            throw new ValidationException(validation.getErrors());
        }

        // 2. 비즈니스 로직 (순수)
        OrderDetails details = calculator.calculate(request);

        // 3. 부수 효과 (격리된 영역)
        Order order = repository.save(new Order(details));
        eventPublisher.publish(new OrderCreatedEvent(order));

        return order;
    }
}

// 순수 함수로 분리된 계산 로직
public class OrderCalculator {
    public OrderDetails calculate(OrderRequest request) {
        BigDecimal subtotal = calculateSubtotal(request.getItems());
        BigDecimal tax = calculateTax(subtotal, request.getTaxRate());
        BigDecimal discount = calculateDiscount(subtotal, request.getCoupon());
        BigDecimal total = subtotal.add(tax).subtract(discount);

        return new OrderDetails(subtotal, tax, discount, total);
    }

    // 모든 메서드가 순수 함수
    private BigDecimal calculateSubtotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    private BigDecimal calculateTax(BigDecimal amount, BigDecimal rate) {
        return amount.multiply(rate);
    }

    private BigDecimal calculateDiscount(BigDecimal amount, Coupon coupon) {
        if (coupon == null) return BigDecimal.ZERO;
        return coupon.apply(amount);
    }
}
```

### 3.3 참조 투명성 (Referential Transparency)

```python
# 참조 투명성: 표현식을 그 결과값으로 대체 가능

# 참조 투명한 함수
def double(x):
    return x * 2

result = double(5) + double(5)
# double(5)를 10으로 대체 가능
result = 10 + 10  # 동일한 결과

# 참조 투명하지 않은 함수
counter = 0
def next_id():
    global counter
    counter += 1
    return counter

result = next_id() + next_id()
# next_id()를 1로 대체하면?
# result = 1 + 1 = 2 (실제: 1 + 2 = 3)
```

---

## 4. 고차 함수와 클로저

### 4.1 일급 함수 (First-class Functions)

```python
# 함수를 변수에 할당
greet = lambda name: f"Hello, {name}"
print(greet("World"))  # Hello, World

# 함수를 리스트에 저장
operations = [
    lambda x: x + 1,
    lambda x: x * 2,
    lambda x: x ** 2
]

value = 5
for op in operations:
    value = op(value)
print(value)  # ((5 + 1) * 2) ** 2 = 144

# 함수를 인자로 전달
def apply_operation(x, operation):
    return operation(x)

result = apply_operation(10, lambda x: x * 2)
print(result)  # 20

# 함수를 반환
def make_multiplier(n):
    return lambda x: x * n

double = make_multiplier(2)
triple = make_multiplier(3)
print(double(5))  # 10
print(triple(5))  # 15
```

### 4.2 고차 함수 (Higher-order Functions)

```java
import java.util.function.*;

public class HigherOrderFunctions {
    public static void main(String[] args) {
        // Function: T -> R
        Function<Integer, Integer> double_ = x -> x * 2;
        Function<Integer, Integer> addOne = x -> x + 1;

        // 함수 합성
        Function<Integer, Integer> doubleThenAddOne = double_.andThen(addOne);
        System.out.println(doubleThenAddOne.apply(5));  // 11

        Function<Integer, Integer> addOneThenDouble = double_.compose(addOne);
        System.out.println(addOneThenDouble.apply(5));  // 12

        // Predicate: T -> boolean
        Predicate<Integer> isEven = x -> x % 2 == 0;
        Predicate<Integer> isPositive = x -> x > 0;
        Predicate<Integer> isEvenAndPositive = isEven.and(isPositive);

        // Consumer: T -> void (부수 효과)
        Consumer<String> print = System.out::println;
        Consumer<String> log = s -> System.out.println("[LOG] " + s);
        Consumer<String> printAndLog = print.andThen(log);

        // Supplier: () -> T
        Supplier<Double> randomSupplier = Math::random;

        // BiFunction: (T, U) -> R
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
    }

    // 함수를 반환하는 고차 함수
    public static Function<Integer, Integer> createMultiplier(int n) {
        return x -> x * n;
    }

    // 함수를 인자로 받는 고차 함수
    public static <T, R> List<R> map(List<T> list, Function<T, R> mapper) {
        List<R> result = new ArrayList<>();
        for (T item : list) {
            result.add(mapper.apply(item));
        }
        return result;
    }
}
```

### 4.3 클로저 (Closure)

```python
# 클로저: 함수가 정의된 환경(스코프)을 기억

def make_counter():
    count = 0  # 자유 변수 (free variable)

    def counter():
        nonlocal count  # 외부 변수 참조
        count += 1
        return count

    return counter

counter1 = make_counter()
counter2 = make_counter()

print(counter1())  # 1
print(counter1())  # 2
print(counter2())  # 1 (독립적인 환경)
print(counter1())  # 3

# 클로저 활용: 설정 값 캡처
def create_logger(prefix):
    def log(message):
        print(f"[{prefix}] {message}")
    return log

error_log = create_logger("ERROR")
info_log = create_logger("INFO")

error_log("Something went wrong")  # [ERROR] Something went wrong
info_log("Process started")        # [INFO] Process started
```

```javascript
// JavaScript - 클로저와 모듈 패턴
const createBankAccount = (initialBalance) => {
    let balance = initialBalance;  // 클로저로 캡슐화

    return {
        deposit: (amount) => {
            balance += amount;
            return balance;
        },
        withdraw: (amount) => {
            if (amount > balance) {
                throw new Error("Insufficient funds");
            }
            balance -= amount;
            return balance;
        },
        getBalance: () => balance
    };
};

const account = createBankAccount(1000);
console.log(account.getBalance());  // 1000
account.deposit(500);
console.log(account.getBalance());  // 1500
// console.log(balance);  // ReferenceError - 외부에서 접근 불가
```

### 4.4 커링 (Currying)

```python
# 커링: 다중 인자 함수를 단일 인자 함수들의 체인으로 변환

# 일반 함수
def add(a, b, c):
    return a + b + c

# 커링된 함수
def add_curried(a):
    def add_b(b):
        def add_c(c):
            return a + b + c
        return add_c
    return add_b

# 사용
result = add_curried(1)(2)(3)  # 6

# 부분 적용 (Partial Application)
add_one = add_curried(1)
add_one_and_two = add_one(2)
result = add_one_and_two(3)  # 6

# functools.partial 활용
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)

print(square(4))  # 16
print(cube(4))    # 64
```

```java
// Java - 커링
import java.util.function.Function;

public class Currying {
    // 커링된 함수
    public static Function<Integer, Function<Integer, Function<Integer, Integer>>>
    curriedAdd = a -> b -> c -> a + b + c;

    public static void main(String[] args) {
        // 완전 적용
        int result = curriedAdd.apply(1).apply(2).apply(3);  // 6

        // 부분 적용
        Function<Integer, Function<Integer, Integer>> add1 = curriedAdd.apply(1);
        Function<Integer, Integer> add1And2 = add1.apply(2);
        int partialResult = add1And2.apply(3);  // 6
    }
}
```

---

## 5. map, filter, reduce

### 5.1 Map

```
map: 각 요소에 함수를 적용하여 새 컬렉션 생성

[1, 2, 3, 4, 5]
      │
      │ map(x => x * 2)
      ▼
[2, 4, 6, 8, 10]
```

```python
# Python map
numbers = [1, 2, 3, 4, 5]

# 기본 map
doubled = list(map(lambda x: x * 2, numbers))
# [2, 4, 6, 8, 10]

# 리스트 컴프리헨션 (더 Pythonic)
doubled = [x * 2 for x in numbers]

# 여러 iterable에 map
list1 = [1, 2, 3]
list2 = [10, 20, 30]
sums = list(map(lambda a, b: a + b, list1, list2))
# [11, 22, 33]
```

```java
// Java Stream map
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

List<Integer> doubled = numbers.stream()
    .map(x -> x * 2)
    .collect(Collectors.toList());
// [2, 4, 6, 8, 10]

// 객체 변환
List<User> users = getUsers();
List<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.toList());

// flatMap: 중첩 구조 평탄화
List<List<Integer>> nested = List.of(
    List.of(1, 2),
    List.of(3, 4),
    List.of(5, 6)
);

List<Integer> flattened = nested.stream()
    .flatMap(List::stream)
    .collect(Collectors.toList());
// [1, 2, 3, 4, 5, 6]
```

### 5.2 Filter

```
filter: 조건을 만족하는 요소만 선택

[1, 2, 3, 4, 5, 6, 7, 8]
         │
         │ filter(x => x % 2 == 0)
         ▼
    [2, 4, 6, 8]
```

```python
# Python filter
numbers = [1, 2, 3, 4, 5, 6, 7, 8]

# 기본 filter
evens = list(filter(lambda x: x % 2 == 0, numbers))
# [2, 4, 6, 8]

# 리스트 컴프리헨션
evens = [x for x in numbers if x % 2 == 0]

# 복합 조건
result = [x for x in numbers if x % 2 == 0 and x > 4]
# [6, 8]
```

```java
// Java Stream filter
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8);

List<Integer> evens = numbers.stream()
    .filter(x -> x % 2 == 0)
    .collect(Collectors.toList());
// [2, 4, 6, 8]

// 복합 조건
List<User> activeAdults = users.stream()
    .filter(u -> u.isActive())
    .filter(u -> u.getAge() >= 18)
    .collect(Collectors.toList());

// takeWhile, dropWhile (Java 9+)
List<Integer> taken = Stream.of(1, 2, 3, 4, 5, 1, 2)
    .takeWhile(x -> x < 4)
    .collect(Collectors.toList());
// [1, 2, 3]
```

### 5.3 Reduce (Fold)

```
reduce: 컬렉션을 단일 값으로 축소

[1, 2, 3, 4, 5]
      │
      │ reduce(0, (acc, x) => acc + x)
      │
      │  0 + 1 = 1
      │  1 + 2 = 3
      │  3 + 3 = 6
      │  6 + 4 = 10
      │ 10 + 5 = 15
      ▼
     15
```

```python
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# 합계
total = reduce(lambda acc, x: acc + x, numbers, 0)
# 15

# 곱
product = reduce(lambda acc, x: acc * x, numbers, 1)
# 120

# 최대값
maximum = reduce(lambda a, b: a if a > b else b, numbers)
# 5

# 복잡한 축소: 빈도수 계산
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
word_count = reduce(
    lambda acc, word: {**acc, word: acc.get(word, 0) + 1},
    words,
    {}
)
# {'apple': 3, 'banana': 2, 'cherry': 1}

# 파이프라인 구성
from functools import reduce

def compose(*functions):
    return reduce(lambda f, g: lambda x: f(g(x)), functions)

add_one = lambda x: x + 1
double = lambda x: x * 2
square = lambda x: x ** 2

pipeline = compose(square, double, add_one)  # 오른쪽부터 적용
print(pipeline(3))  # ((3 + 1) * 2) ** 2 = 64
```

```java
// Java Stream reduce
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

// 합계
int sum = numbers.stream()
    .reduce(0, (acc, x) -> acc + x);
// 또는
int sum2 = numbers.stream()
    .reduce(0, Integer::sum);

// Optional 반환 (초기값 없는 경우)
Optional<Integer> max = numbers.stream()
    .reduce(Integer::max);

// 복잡한 축소
Map<String, Integer> wordCount = words.stream()
    .reduce(
        new HashMap<>(),
        (map, word) -> {
            map.merge(word, 1, Integer::sum);
            return map;
        },
        (map1, map2) -> {
            map2.forEach((k, v) -> map1.merge(k, v, Integer::sum));
            return map1;
        }
    );

// collect 사용 (더 효율적)
Map<String, Long> wordCount2 = words.stream()
    .collect(Collectors.groupingBy(
        Function.identity(),
        Collectors.counting()
    ));
```

### 5.4 map, filter, reduce 조합

```python
# 실전 예제: 주문 처리
orders = [
    {"id": 1, "status": "completed", "amount": 100, "customer": "Alice"},
    {"id": 2, "status": "pending", "amount": 200, "customer": "Bob"},
    {"id": 3, "status": "completed", "amount": 150, "customer": "Alice"},
    {"id": 4, "status": "completed", "amount": 300, "customer": "Charlie"},
    {"id": 5, "status": "pending", "amount": 250, "customer": "Bob"},
]

# 완료된 주문의 총 금액
total_completed = reduce(
    lambda acc, order: acc + order["amount"],
    filter(lambda o: o["status"] == "completed", orders),
    0
)
# 550

# 파이프라인 스타일
from toolz import pipe, curry

@curry
def filter_by(key, value, items):
    return filter(lambda x: x[key] == value, items)

@curry
def map_to(key, items):
    return map(lambda x: x[key], items)

total = pipe(
    orders,
    filter_by("status", "completed"),
    map_to("amount"),
    sum
)
# 550
```

```java
// Java - 실전 예제
@Data
class Order {
    private Long id;
    private String status;
    private BigDecimal amount;
    private String customer;
}

public class OrderProcessor {
    public BigDecimal getTotalCompletedAmount(List<Order> orders) {
        return orders.stream()
            .filter(o -> "completed".equals(o.getStatus()))
            .map(Order::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    public Map<String, BigDecimal> getAmountByCustomer(List<Order> orders) {
        return orders.stream()
            .filter(o -> "completed".equals(o.getStatus()))
            .collect(Collectors.groupingBy(
                Order::getCustomer,
                Collectors.reducing(
                    BigDecimal.ZERO,
                    Order::getAmount,
                    BigDecimal::add
                )
            ));
    }

    public Optional<Order> getHighestOrder(List<Order> orders) {
        return orders.stream()
            .filter(o -> "completed".equals(o.getStatus()))
            .max(Comparator.comparing(Order::getAmount));
    }
}
```

---

## 6. 모나드 개념

### 6.1 모나드란?

```
모나드는 "값을 감싸는 컨텍스트"를 다루기 위한 디자인 패턴

모나드의 3가지 요소:
1. 타입 생성자 (Type Constructor): 값을 감싸는 컨테이너
2. unit (return): 값을 모나드로 감싸기
3. bind (flatMap): 모나드 내부 값에 함수 적용

모나드 법칙:
1. Left Identity:  unit(a).flatMap(f) == f(a)
2. Right Identity: m.flatMap(unit) == m
3. Associativity:  m.flatMap(f).flatMap(g) == m.flatMap(x -> f(x).flatMap(g))
```

### 6.2 Optional (Maybe) 모나드

```java
// Java Optional - null 안전성
public class OptionalExample {
    public static void main(String[] args) {
        // 생성 (unit)
        Optional<String> some = Optional.of("Hello");
        Optional<String> none = Optional.empty();
        Optional<String> nullable = Optional.ofNullable(getValue());

        // map: Optional<T> -> Optional<U>
        Optional<Integer> length = some.map(String::length);

        // flatMap: Optional 중첩 방지
        Optional<String> result = getUserById(1L)
            .flatMap(user -> getAddressByUser(user))
            .flatMap(address -> getCityByAddress(address));

        // 체이닝
        String city = getUserById(1L)
            .flatMap(User::getAddress)      // Optional<Address>
            .flatMap(Address::getCity)      // Optional<String>
            .map(String::toUpperCase)
            .orElse("Unknown");

        // orElse vs orElseGet
        String value1 = some.orElse("default");           // 항상 평가
        String value2 = some.orElseGet(() -> "default");  // lazy 평가

        // orElseThrow
        String value3 = some.orElseThrow(
            () -> new NoSuchElementException("Value not found")
        );
    }

    // null 체크 없이 안전한 체이닝
    public static String getUserCity(Long userId) {
        return Optional.ofNullable(userRepository.findById(userId))
            .flatMap(user -> Optional.ofNullable(user.getAddress()))
            .flatMap(address -> Optional.ofNullable(address.getCity()))
            .orElse("Unknown");
    }
}
```

### 6.3 Result (Either) 모나드

```java
// 성공/실패를 명시적으로 표현
public sealed interface Result<T> {
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String error) implements Result<T> {}

    static <T> Result<T> success(T value) {
        return new Success<>(value);
    }

    static <T> Result<T> failure(String error) {
        return new Failure<>(error);
    }

    default <U> Result<U> map(Function<T, U> mapper) {
        return switch (this) {
            case Success<T> s -> Result.success(mapper.apply(s.value()));
            case Failure<T> f -> Result.failure(f.error());
        };
    }

    default <U> Result<U> flatMap(Function<T, Result<U>> mapper) {
        return switch (this) {
            case Success<T> s -> mapper.apply(s.value());
            case Failure<T> f -> Result.failure(f.error());
        };
    }

    default T getOrElse(T defaultValue) {
        return switch (this) {
            case Success<T> s -> s.value();
            case Failure<T> f -> defaultValue;
        };
    }
}

// 사용 예
public class UserService {
    public Result<User> createUser(UserRequest request) {
        return validateRequest(request)
            .flatMap(this::checkDuplicate)
            .flatMap(this::saveUser)
            .map(this::toDto);
    }

    private Result<UserRequest> validateRequest(UserRequest request) {
        if (request.getEmail() == null) {
            return Result.failure("Email is required");
        }
        return Result.success(request);
    }
}
```

### 6.4 CompletableFuture (Promise) 모나드

```java
// 비동기 연산 체이닝
public class AsyncExample {
    public CompletableFuture<OrderResult> processOrderAsync(Long orderId) {
        return CompletableFuture.supplyAsync(() -> findOrder(orderId))
            .thenApply(this::validateOrder)        // map
            .thenCompose(this::checkInventory)     // flatMap
            .thenCompose(this::processPayment)
            .thenApply(this::createShipment)
            .exceptionally(ex -> {
                log.error("Order processing failed", ex);
                return OrderResult.failed(ex.getMessage());
            });
    }

    // 여러 비동기 작업 조합
    public CompletableFuture<UserProfile> getUserProfile(Long userId) {
        CompletableFuture<User> userFuture = fetchUser(userId);
        CompletableFuture<List<Order>> ordersFuture = fetchOrders(userId);
        CompletableFuture<Preferences> prefsFuture = fetchPreferences(userId);

        return CompletableFuture.allOf(userFuture, ordersFuture, prefsFuture)
            .thenApply(v -> new UserProfile(
                userFuture.join(),
                ordersFuture.join(),
                prefsFuture.join()
            ));
    }
}
```

### 6.5 Stream 모나드

```java
// Stream도 모나드 법칙을 따름
public class StreamMonad {
    public static void main(String[] args) {
        // unit: Stream.of()
        Stream<Integer> stream = Stream.of(1, 2, 3);

        // map: 변환
        Stream<Integer> doubled = stream.map(x -> x * 2);

        // flatMap: 중첩 스트림 평탄화
        List<List<Integer>> nested = List.of(
            List.of(1, 2), List.of(3, 4), List.of(5, 6)
        );

        List<Integer> flat = nested.stream()
            .flatMap(List::stream)
            .collect(Collectors.toList());
        // [1, 2, 3, 4, 5, 6]

        // 실전: 1:N 관계 처리
        List<Order> allOrders = customers.stream()
            .flatMap(customer -> customer.getOrders().stream())
            .collect(Collectors.toList());
    }
}
```

---

## 7. 함수형 프로그래밍 실전

### 7.1 함수형 에러 처리

```python
# Python - Result 패턴
from dataclasses import dataclass
from typing import TypeVar, Generic, Callable, Union

T = TypeVar('T')
E = TypeVar('E')

@dataclass
class Ok(Generic[T]):
    value: T

@dataclass
class Err(Generic[E]):
    error: E

Result = Union[Ok[T], Err[E]]

def safe_divide(a: float, b: float) -> Result[float, str]:
    if b == 0:
        return Err("Division by zero")
    return Ok(a / b)

def safe_sqrt(x: float) -> Result[float, str]:
    if x < 0:
        return Err("Cannot sqrt negative number")
    return Ok(x ** 0.5)

# 체이닝
def bind(result: Result[T, E], f: Callable[[T], Result]) -> Result:
    if isinstance(result, Err):
        return result
    return f(result.value)

result = bind(
    bind(safe_divide(16, 4), safe_sqrt),
    lambda x: Ok(x * 2)
)
# Ok(value=4.0)
```

### 7.2 함수형 데이터 파이프라인

```python
from functools import reduce
from typing import List, Dict, Any

# 파이프라인 연산자
def pipe(data, *functions):
    return reduce(lambda acc, f: f(acc), functions, data)

# 데이터 처리 함수들
def parse_json(raw: str) -> List[Dict]:
    import json
    return json.loads(raw)

def filter_active(users: List[Dict]) -> List[Dict]:
    return [u for u in users if u.get('active', False)]

def extract_emails(users: List[Dict]) -> List[str]:
    return [u['email'] for u in users if 'email' in u]

def to_lowercase(items: List[str]) -> List[str]:
    return [item.lower() for item in items]

def remove_duplicates(items: List[str]) -> List[str]:
    return list(set(items))

# 파이프라인 실행
raw_data = '[{"email": "A@test.com", "active": true}, ...]'

emails = pipe(
    raw_data,
    parse_json,
    filter_active,
    extract_emails,
    to_lowercase,
    remove_duplicates
)
```

```java
// Java - 실전 데이터 처리
public class DataPipeline {
    public List<CustomerReport> generateReports(List<Order> orders) {
        return orders.stream()
            // 필터링
            .filter(o -> o.getStatus() == OrderStatus.COMPLETED)
            .filter(o -> o.getCreatedAt().isAfter(LocalDate.now().minusDays(30)))

            // 그룹화
            .collect(Collectors.groupingBy(Order::getCustomerId))

            // Map 엔트리 스트림으로 변환
            .entrySet().stream()

            // 리포트 생성
            .map(entry -> {
                Long customerId = entry.getKey();
                List<Order> customerOrders = entry.getValue();

                BigDecimal totalAmount = customerOrders.stream()
                    .map(Order::getAmount)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

                int orderCount = customerOrders.size();

                BigDecimal avgAmount = totalAmount.divide(
                    BigDecimal.valueOf(orderCount),
                    RoundingMode.HALF_UP
                );

                return new CustomerReport(
                    customerId,
                    orderCount,
                    totalAmount,
                    avgAmount
                );
            })

            // 정렬 (총 금액 내림차순)
            .sorted(Comparator.comparing(
                CustomerReport::getTotalAmount,
                Comparator.reverseOrder()
            ))

            .collect(Collectors.toList());
    }
}
```

### 7.3 함수형 상태 관리 (Redux 스타일)

```javascript
// JavaScript - 함수형 상태 관리
const initialState = {
    count: 0,
    items: [],
    loading: false
};

// 순수 함수인 리듀서
const reducer = (state = initialState, action) => {
    switch (action.type) {
        case 'INCREMENT':
            return { ...state, count: state.count + 1 };

        case 'DECREMENT':
            return { ...state, count: state.count - 1 };

        case 'ADD_ITEM':
            return {
                ...state,
                items: [...state.items, action.payload]
            };

        case 'REMOVE_ITEM':
            return {
                ...state,
                items: state.items.filter(item => item.id !== action.payload)
            };

        case 'SET_LOADING':
            return { ...state, loading: action.payload };

        default:
            return state;
    }
};

// 불변성 유지 - immer 라이브러리 활용
import produce from 'immer';

const reducerWithImmer = (state = initialState, action) => {
    return produce(state, draft => {
        switch (action.type) {
            case 'INCREMENT':
                draft.count += 1;
                break;
            case 'ADD_ITEM':
                draft.items.push(action.payload);
                break;
            case 'REMOVE_ITEM':
                const index = draft.items.findIndex(i => i.id === action.payload);
                if (index !== -1) draft.items.splice(index, 1);
                break;
        }
    });
};
```

---

## 참고 자료

- "Functional Programming in Java" - Venkat Subramaniam
- "Effective Java" (Chapter 7: Lambdas and Streams) - Joshua Bloch
- "Grokking Functional Programming" - Michal Plachta
- "Professor Frisby's Mostly Adequate Guide to Functional Programming"
- "Learn You a Haskell for Great Good!" - Miran Lipovaca
- Python Functional Programming HOWTO
