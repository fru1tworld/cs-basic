# 객체지향 프로그래밍 (OOP)

## 목차
1. [OOP 기본 개념](#1-oop-기본-개념)
2. [SOLID 원칙](#2-solid-원칙)
3. [디자인 패턴](#3-디자인-패턴)
4. [상속 vs 합성](#4-상속-vs-합성)
5. [의존성 주입 (DI)](#5-의존성-주입-di)
6. [추상화와 캡슐화](#6-추상화와-캡슐화)
---

## 1. OOP 기본 개념

### 1.1 4대 특성

```
┌─────────────────────────────────────────────────────────────┐
│                     OOP 4대 특성                             │
├─────────────┬─────────────┬─────────────┬─────────────────┤
│   캡슐화     │    상속     │   다형성    │     추상화       │
│ Encapsulation│ Inheritance │ Polymorphism│   Abstraction   │
├─────────────┼─────────────┼─────────────┼─────────────────┤
│ 데이터와     │ 기존 클래스 │ 같은 인터   │ 복잡한 시스템   │
│ 메서드 묶음  │ 재사용/확장 │ 페이스로    │ 을 단순한       │
│ 정보 은닉    │             │ 다양한 구현 │ 모델로 표현     │
└─────────────┴─────────────┴─────────────┴─────────────────┘
```

```java
// 4대 특성 예제
// 추상화: Animal이라는 추상적 개념 정의
abstract class Animal {
    protected String name;  // 캡슐화: protected 접근 제어

    public abstract void makeSound();  // 추상 메서드
}

// 상속: Animal 클래스 확장
class Dog extends Animal {
    public Dog(String name) {
        this.name = name;
    }

    @Override
    public void makeSound() {  // 다형성: 오버라이딩
        System.out.println(name + ": 멍멍!");
    }
}

class Cat extends Animal {
    public Cat(String name) {
        this.name = name;
    }

    @Override
    public void makeSound() {  // 다형성: 다른 구현
        System.out.println(name + ": 야옹!");
    }
}

public class Main {
    public static void main(String[] args) {
        // 다형성: 부모 타입으로 자식 객체 참조
        Animal[] animals = {new Dog("바둑이"), new Cat("나비")};

        for (Animal animal : animals) {
            animal.makeSound();  // 런타임에 실제 타입의 메서드 호출
        }
    }
}
```

---

## 2. SOLID 원칙

### 2.1 Single Responsibility Principle (SRP) - 단일 책임 원칙

> "클래스는 단 하나의 변경 이유만 가져야 한다"

```java
// 위반 사례: User 클래스가 여러 책임을 가짐
class UserBad {
    private String name;
    private String email;

    public void saveToDatabase() { /* DB 저장 로직 */ }
    public void sendEmail() { /* 이메일 발송 로직 */ }
    public String formatJson() { /* JSON 변환 로직 */ }
}

// 개선: 책임 분리
class User {
    private String name;
    private String email;

    // Getter, Setter만 포함
    public String getName() { return name; }
    public String getEmail() { return email; }
}

class UserRepository {
    public void save(User user) { /* DB 저장 로직 */ }
    public User findById(Long id) { /* 조회 로직 */ }
}

class EmailService {
    public void sendEmail(User user, String message) { /* 이메일 발송 로직 */ }
}

class UserSerializer {
    public String toJson(User user) { /* JSON 변환 로직 */ }
    public User fromJson(String json) { /* 역직렬화 */ }
}
```

### 2.2 Open-Closed Principle (OCP) - 개방-폐쇄 원칙

> "확장에는 열려 있고, 수정에는 닫혀 있어야 한다"

```java
// 위반 사례: 새 할인 타입 추가 시 메서드 수정 필요
class DiscountCalculatorBad {
    public double calculate(String discountType, double price) {
        if (discountType.equals("FIXED")) {
            return price - 1000;
        } else if (discountType.equals("PERCENT")) {
            return price * 0.9;
        }
        // 새 할인 타입 추가 시 여기를 수정해야 함
        return price;
    }
}

// 개선: 인터페이스로 확장 가능하게 설계
interface DiscountPolicy {
    double discount(double price);
}

class FixedDiscountPolicy implements DiscountPolicy {
    private final double amount;

    public FixedDiscountPolicy(double amount) {
        this.amount = amount;
    }

    @Override
    public double discount(double price) {
        return price - amount;
    }
}

class PercentDiscountPolicy implements DiscountPolicy {
    private final double percent;

    public PercentDiscountPolicy(double percent) {
        this.percent = percent;
    }

    @Override
    public double discount(double price) {
        return price * (1 - percent / 100);
    }
}

// 새 할인 정책 추가 시 기존 코드 수정 없이 클래스만 추가
class BulkDiscountPolicy implements DiscountPolicy {
    @Override
    public double discount(double price) {
        return price * 0.7;  // 30% 할인
    }
}

class Order {
    private DiscountPolicy discountPolicy;

    public Order(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }

    public double calculatePrice(double price) {
        return discountPolicy.discount(price);
    }
}
```

### 2.3 Liskov Substitution Principle (LSP) - 리스코프 치환 원칙

> "자식 클래스는 부모 클래스를 대체할 수 있어야 한다"

```java
// 위반 사례: 직사각형-정사각형 문제
class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width) { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    public int getArea() { return width * height; }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // 정사각형이므로 높이도 변경
    }

    @Override
    public void setHeight(int height) {
        this.width = height;  // 정사각형이므로 너비도 변경
        this.height = height;
    }
}

// 클라이언트 코드에서 문제 발생
void test(Rectangle rect) {
    rect.setWidth(5);
    rect.setHeight(4);
    assert rect.getArea() == 20;  // Square면 실패! (16)
}

// 개선: 공통 인터페이스로 분리
interface Shape {
    int getArea();
}

class RectangleGood implements Shape {
    private final int width;
    private final int height;

    public RectangleGood(int width, int height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public int getArea() { return width * height; }
}

class SquareGood implements Shape {
    private final int side;

    public SquareGood(int side) {
        this.side = side;
    }

    @Override
    public int getArea() { return side * side; }
}
```

### 2.4 Interface Segregation Principle (ISP) - 인터페이스 분리 원칙

> "클라이언트는 자신이 사용하지 않는 메서드에 의존하면 안 된다"

```java
// 위반 사례: 거대한 인터페이스
interface Worker {
    void work();
    void eat();
    void sleep();
}

class Robot implements Worker {
    @Override
    public void work() { /* 작업 */ }

    @Override
    public void eat() {
        throw new UnsupportedOperationException();  // 로봇은 안 먹음
    }

    @Override
    public void sleep() {
        throw new UnsupportedOperationException();  // 로봇은 안 잠
    }
}

// 개선: 인터페이스 분리
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

interface Sleepable {
    void sleep();
}

class Human implements Workable, Eatable, Sleepable {
    @Override public void work() { /* 작업 */ }
    @Override public void eat() { /* 식사 */ }
    @Override public void sleep() { /* 수면 */ }
}

class RobotGood implements Workable {
    @Override public void work() { /* 작업 */ }
    // 필요 없는 메서드 구현 불필요
}
```

### 2.5 Dependency Inversion Principle (DIP) - 의존 역전 원칙

> "고수준 모듈은 저수준 모듈에 의존해서는 안 되며, 둘 다 추상화에 의존해야 한다"

```java
// 위반 사례: 고수준 모듈이 저수준 모듈에 직접 의존
class MySQLDatabase {
    public void save(String data) { /* MySQL 저장 */ }
}

class UserServiceBad {
    private MySQLDatabase database = new MySQLDatabase();  // 직접 의존

    public void createUser(String userData) {
        database.save(userData);  // MySQL에 강하게 결합
    }
}

// 개선: 추상화에 의존
interface Database {
    void save(String data);
}

class MySQLDatabaseGood implements Database {
    @Override
    public void save(String data) { /* MySQL 저장 */ }
}

class MongoDBDatabase implements Database {
    @Override
    public void save(String data) { /* MongoDB 저장 */ }
}

class UserServiceGood {
    private final Database database;  // 추상화에 의존

    // 의존성 주입
    public UserServiceGood(Database database) {
        this.database = database;
    }

    public void createUser(String userData) {
        database.save(userData);  // 어떤 DB든 가능
    }
}
```

---

## 3. 디자인 패턴

### 3.1 생성 패턴 (Creational Patterns)

#### Singleton (싱글톤)

```java
// 1. Eager Initialization
public class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    private EagerSingleton() {}

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}

// 2. Double-Checked Locking (DCL)
public class DCLSingleton {
    private static volatile DCLSingleton instance;

    private DCLSingleton() {}

    public static DCLSingleton getInstance() {
        if (instance == null) {
            synchronized (DCLSingleton.class) {
                if (instance == null) {
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}

// 3. Bill Pugh Singleton (권장)
public class BillPughSingleton {
    private BillPughSingleton() {}

    private static class SingletonHelper {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}

// 4. Enum Singleton (Joshua Bloch 권장)
public enum EnumSingleton {
    INSTANCE;

    public void doSomething() {
        // 비즈니스 로직
    }
}
```

#### Factory Method (팩토리 메서드)

```java
// Product 인터페이스
interface Notification {
    void send(String message);
}

class EmailNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

class SMSNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}

class PushNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Push: " + message);
    }
}

// Factory
class NotificationFactory {
    public Notification createNotification(String type) {
        return switch (type.toUpperCase()) {
            case "EMAIL" -> new EmailNotification();
            case "SMS" -> new SMSNotification();
            case "PUSH" -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// 사용
NotificationFactory factory = new NotificationFactory();
Notification notification = factory.createNotification("EMAIL");
notification.send("Hello!");
```

#### Builder (빌더)

```java
public class User {
    private final String name;       // 필수
    private final String email;      // 필수
    private final int age;           // 선택
    private final String phone;      // 선택
    private final String address;    // 선택

    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
        this.phone = builder.phone;
        this.address = builder.address;
    }

    public static class Builder {
        // 필수 매개변수
        private final String name;
        private final String email;

        // 선택 매개변수 - 기본값
        private int age = 0;
        private String phone = "";
        private String address = "";

        public Builder(String name, String email) {
            this.name = name;
            this.email = email;
        }

        public Builder age(int age) {
            this.age = age;
            return this;
        }

        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }

        public Builder address(String address) {
            this.address = address;
            return this;
        }

        public User build() {
            // 유효성 검사
            if (name == null || name.isEmpty()) {
                throw new IllegalStateException("Name is required");
            }
            return new User(this);
        }
    }

    // Getter 메서드들...
}

// 사용
User user = new User.Builder("John", "john@email.com")
    .age(25)
    .phone("010-1234-5678")
    .build();
```

### 3.2 구조 패턴 (Structural Patterns)

#### Adapter (어댑터)

```java
// 기존 인터페이스
interface MediaPlayer {
    void play(String fileName);
}

// 호환되지 않는 인터페이스
interface AdvancedMediaPlayer {
    void playVlc(String fileName);
    void playMp4(String fileName);
}

class VlcPlayer implements AdvancedMediaPlayer {
    @Override public void playVlc(String fileName) {
        System.out.println("Playing VLC: " + fileName);
    }
    @Override public void playMp4(String fileName) {}
}

class Mp4Player implements AdvancedMediaPlayer {
    @Override public void playVlc(String fileName) {}
    @Override public void playMp4(String fileName) {
        System.out.println("Playing MP4: " + fileName);
    }
}

// 어댑터
class MediaAdapter implements MediaPlayer {
    private AdvancedMediaPlayer advancedPlayer;

    public MediaAdapter(String audioType) {
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedPlayer = new VlcPlayer();
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedPlayer = new Mp4Player();
        }
    }

    @Override
    public void play(String fileName) {
        String extension = fileName.substring(fileName.lastIndexOf('.') + 1);
        if (extension.equalsIgnoreCase("vlc")) {
            advancedPlayer.playVlc(fileName);
        } else if (extension.equalsIgnoreCase("mp4")) {
            advancedPlayer.playMp4(fileName);
        }
    }
}
```

#### Decorator (데코레이터)

```java
// 컴포넌트 인터페이스
interface Coffee {
    double getCost();
    String getDescription();
}

// 기본 구현
class SimpleCoffee implements Coffee {
    @Override public double getCost() { return 1.0; }
    @Override public String getDescription() { return "Simple Coffee"; }
}

// 데코레이터 추상 클래스
abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;

    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }

    @Override
    public double getCost() { return decoratedCoffee.getCost(); }

    @Override
    public String getDescription() { return decoratedCoffee.getDescription(); }
}

// 구체적인 데코레이터들
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }

    @Override
    public double getCost() { return super.getCost() + 0.5; }

    @Override
    public String getDescription() { return super.getDescription() + ", Milk"; }
}

class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) { super(coffee); }

    @Override
    public double getCost() { return super.getCost() + 0.2; }

    @Override
    public String getDescription() { return super.getDescription() + ", Sugar"; }
}

// 사용
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
// 출력: Simple Coffee, Milk, Sugar $1.7
```

#### Proxy (프록시)

```java
interface Image {
    void display();
}

// 실제 객체 (무거운 리소스)
class RealImage implements Image {
    private String fileName;

    public RealImage(String fileName) {
        this.fileName = fileName;
        loadFromDisk();  // 비용이 큰 작업
    }

    private void loadFromDisk() {
        System.out.println("Loading " + fileName);
    }

    @Override
    public void display() {
        System.out.println("Displaying " + fileName);
    }
}

// 프록시 (지연 로딩)
class ProxyImage implements Image {
    private RealImage realImage;
    private String fileName;

    public ProxyImage(String fileName) {
        this.fileName = fileName;
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(fileName);  // 최초 사용 시에만 로딩
        }
        realImage.display();
    }
}

// 사용
Image image = new ProxyImage("photo.jpg");
// 이미지가 아직 로딩되지 않음
image.display();  // 이 시점에 로딩
image.display();  // 이미 로딩됨, 바로 표시
```

### 3.3 행위 패턴 (Behavioral Patterns)

#### Strategy (전략)

```java
// 전략 인터페이스
interface PaymentStrategy {
    void pay(int amount);
}

// 구체적인 전략들
class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;

    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }

    @Override
    public void pay(int amount) {
        System.out.println(amount + "원을 신용카드로 결제");
    }
}

class KakaoPayPayment implements PaymentStrategy {
    private String phoneNumber;

    public KakaoPayPayment(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    @Override
    public void pay(int amount) {
        System.out.println(amount + "원을 카카오페이로 결제");
    }
}

// 컨텍스트
class ShoppingCart {
    private PaymentStrategy paymentStrategy;

    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    public void checkout(int amount) {
        paymentStrategy.pay(amount);
    }
}

// 사용
ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new CreditCardPayment("1234-5678"));
cart.checkout(50000);

cart.setPaymentStrategy(new KakaoPayPayment("010-1234-5678"));
cart.checkout(30000);
```

#### Observer (옵저버)

```java
import java.util.*;

// 옵저버 인터페이스
interface Observer {
    void update(String message);
}

// 주제 (Subject)
class NewsPublisher {
    private List<Observer> observers = new ArrayList<>();
    private String latestNews;

    public void subscribe(Observer observer) {
        observers.add(observer);
    }

    public void unsubscribe(Observer observer) {
        observers.remove(observer);
    }

    public void publish(String news) {
        this.latestNews = news;
        notifyObservers();
    }

    private void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(latestNews);
        }
    }
}

// 구체적인 옵저버
class EmailSubscriber implements Observer {
    private String email;

    public EmailSubscriber(String email) {
        this.email = email;
    }

    @Override
    public void update(String message) {
        System.out.println("Email to " + email + ": " + message);
    }
}

class SMSSubscriber implements Observer {
    private String phone;

    public SMSSubscriber(String phone) {
        this.phone = phone;
    }

    @Override
    public void update(String message) {
        System.out.println("SMS to " + phone + ": " + message);
    }
}

// 사용
NewsPublisher publisher = new NewsPublisher();
publisher.subscribe(new EmailSubscriber("user@email.com"));
publisher.subscribe(new SMSSubscriber("010-1234-5678"));
publisher.publish("Breaking News!");
```

#### Template Method (템플릿 메서드)

```java
abstract class OrderProcessor {
    // 템플릿 메서드 - 알고리즘의 골격
    public final void processOrder() {
        validateOrder();
        calculateTotal();
        applyDiscount();  // Hook 메서드
        processPayment();
        sendConfirmation();
    }

    protected abstract void validateOrder();
    protected abstract void processPayment();

    // 기본 구현 (선택적 오버라이드)
    protected void calculateTotal() {
        System.out.println("총액 계산");
    }

    // Hook 메서드 - 서브클래스가 선택적으로 오버라이드
    protected void applyDiscount() {
        // 기본: 할인 없음
    }

    private void sendConfirmation() {
        System.out.println("확인 이메일 발송");
    }
}

class OnlineOrderProcessor extends OrderProcessor {
    @Override
    protected void validateOrder() {
        System.out.println("온라인 주문 유효성 검사");
    }

    @Override
    protected void processPayment() {
        System.out.println("온라인 결제 처리");
    }

    @Override
    protected void applyDiscount() {
        System.out.println("온라인 전용 5% 할인 적용");
    }
}

class InStoreOrderProcessor extends OrderProcessor {
    @Override
    protected void validateOrder() {
        System.out.println("매장 주문 유효성 검사");
    }

    @Override
    protected void processPayment() {
        System.out.println("POS 결제 처리");
    }
}
```

### 3.4 디자인 패턴 분류표

| 분류 | 패턴 | 목적 |
|------|------|------|
| **생성** | Singleton | 인스턴스 하나만 생성 |
| | Factory Method | 객체 생성을 서브클래스에 위임 |
| | Abstract Factory | 관련 객체 군 생성 |
| | Builder | 복잡한 객체 단계별 생성 |
| | Prototype | 기존 객체 복제 |
| **구조** | Adapter | 호환되지 않는 인터페이스 연결 |
| | Bridge | 추상화와 구현 분리 |
| | Composite | 트리 구조로 객체 구성 |
| | Decorator | 동적으로 기능 추가 |
| | Facade | 복잡한 서브시스템에 단순한 인터페이스 제공 |
| | Flyweight | 다수의 객체 공유로 메모리 절약 |
| | Proxy | 다른 객체에 대한 접근 제어 |
| **행위** | Chain of Responsibility | 요청을 처리할 객체 연결 |
| | Command | 요청을 객체로 캡슐화 |
| | Iterator | 컬렉션 순회 |
| | Mediator | 객체 간 통신 중재 |
| | Memento | 객체 상태 저장/복원 |
| | Observer | 상태 변경 알림 |
| | State | 상태에 따른 행동 변경 |
| | Strategy | 알고리즘 군 캡슐화 |
| | Template Method | 알고리즘 골격 정의 |
| | Visitor | 구조와 기능 분리 |

---

## 4. 상속 vs 합성

### 4.1 상속의 문제점

```java
// 문제 1: 캡슐화 깨짐 (Fragile Base Class Problem)
class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);  // 내부적으로 add() 호출!
    }

    public int getAddCount() {
        return addCount;
    }
}

// 사용
InstrumentedHashSet<String> set = new InstrumentedHashSet<>();
set.addAll(List.of("a", "b", "c"));
System.out.println(set.getAddCount());  // 예상: 3, 실제: 6 (addAll이 add 호출)
```

```java
// 문제 2: 불필요한 메서드 노출
class Stack<E> extends Vector<E> {
    public E push(E item) {
        addElement(item);
        return item;
    }

    public E pop() {
        E obj = peek();
        removeElementAt(size() - 1);
        return obj;
    }

    public E peek() {
        return elementAt(size() - 1);
    }
}

// 문제: Vector의 모든 메서드(add, remove, get 등)가 노출됨
// 스택의 LIFO 특성을 깨뜨릴 수 있음
```

### 4.2 합성 (Composition) 으로 해결

```java
// 합성을 사용한 개선
class InstrumentedSet<E> implements Set<E> {
    private final Set<E> set;  // 합성
    private int addCount = 0;

    public InstrumentedSet(Set<E> set) {
        this.set = set;
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return set.add(e);
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return set.addAll(c);
    }

    public int getAddCount() {
        return addCount;
    }

    // Set 인터페이스의 다른 메서드들은 위임
    @Override public int size() { return set.size(); }
    @Override public boolean isEmpty() { return set.isEmpty(); }
    @Override public boolean contains(Object o) { return set.contains(o); }
    @Override public Iterator<E> iterator() { return set.iterator(); }
    // ... 기타 메서드들
}

// 사용 - 어떤 Set 구현체든 사용 가능
Set<String> set1 = new InstrumentedSet<>(new HashSet<>());
Set<String> set2 = new InstrumentedSet<>(new TreeSet<>());
```

### 4.3 상속 vs 합성 비교

```
상속 (is-a 관계)
─────────────────
    Animal
       △
       │
     ┌─┴─┐
    Dog  Cat

합성 (has-a 관계)
─────────────────
    ┌───────┐     ┌────────┐
    │  Car  │────▶│ Engine │
    └───────┘     └────────┘
         │        ┌───────┐
         └───────▶│ Wheel │
                  └───────┘
```

| 특성 | 상속 | 합성 |
|------|------|------|
| 관계 | is-a (자식 is a 부모) | has-a (전체 has a 부분) |
| 결합도 | 강함 (컴파일 타임) | 약함 (런타임에 변경 가능) |
| 코드 재사용 | 자동으로 부모 기능 상속 | 명시적으로 위임 필요 |
| 유연성 | 낮음 (정적) | 높음 (동적) |
| 캡슐화 | 깨질 수 있음 | 유지됨 |

> **원칙**: "상속보다 합성을 선호하라" (Favor composition over inheritance)
> - 상속은 명확한 is-a 관계일 때만 사용
> - 확장이 아닌 코드 재사용이 목적이면 합성 사용

---

## 5. 의존성 주입 (DI)

### 5.1 의존성 주입 방식

```java
// 1. 생성자 주입 (권장)
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    // final 필드 + 생성자 = 불변성 보장
    @Autowired  // Spring 4.3+에서는 단일 생성자면 생략 가능
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    public void createUser(User user) {
        userRepository.save(user);
        emailService.sendWelcomeEmail(user);
    }
}

// 2. 세터 주입
@Service
public class UserServiceSetter {
    private UserRepository userRepository;

    @Autowired
    public void setUserRepository(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

// 3. 필드 주입 (권장하지 않음)
@Service
public class UserServiceField {
    @Autowired
    private UserRepository userRepository;  // 테스트 어려움
}
```

### 5.2 DI의 장점

```java
// DI 없는 코드 - 테스트 어려움
class OrderServiceBad {
    private final PaymentGateway gateway = new StripePaymentGateway();

    public void processOrder(Order order) {
        gateway.charge(order.getAmount());  // 실제 결제 발생!
    }
}

// DI 적용 코드 - 테스트 용이
class OrderServiceGood {
    private final PaymentGateway gateway;

    public OrderServiceGood(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    public void processOrder(Order order) {
        gateway.charge(order.getAmount());
    }
}

// 테스트
class OrderServiceTest {
    @Test
    void testProcessOrder() {
        // Mock 객체 주입
        PaymentGateway mockGateway = mock(PaymentGateway.class);
        OrderServiceGood service = new OrderServiceGood(mockGateway);

        Order order = new Order(10000);
        service.processOrder(order);

        // 실제 결제 없이 검증
        verify(mockGateway).charge(10000);
    }
}
```

### 5.3 DI 컨테이너 (Spring)

```java
// 설정 클래스
@Configuration
public class AppConfig {

    @Bean
    public UserRepository userRepository() {
        return new JpaUserRepository();
    }

    @Bean
    public EmailService emailService() {
        return new SmtpEmailService();
    }

    @Bean
    public UserService userService(UserRepository userRepository,
                                    EmailService emailService) {
        return new UserService(userRepository, emailService);
    }
}

// 또는 컴포넌트 스캔
@Repository
public class JpaUserRepository implements UserRepository { }

@Service
public class SmtpEmailService implements EmailService { }

// 조건부 Bean 등록
@Configuration
public class DatabaseConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new H2DataSource();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        return new MySQLDataSource();
    }
}
```

---

## 6. 추상화와 캡슐화

### 6.1 추상화 (Abstraction)

```java
// 추상화: 공통된 특성을 뽑아내어 일반화

// 낮은 수준의 추상화
class MySQLConnection {
    public void connect() { /* MySQL 연결 */ }
    public ResultSet query(String sql) { /* 쿼리 실행 */ }
}

// 높은 수준의 추상화
interface DatabaseConnection {
    void connect();
    ResultSet query(String sql);
}

class MySQLConnectionImpl implements DatabaseConnection { }
class PostgreSQLConnectionImpl implements DatabaseConnection { }
class MongoDBConnectionImpl implements DatabaseConnection { }

// 더 높은 수준의 추상화
interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    T save(T entity);
    void delete(T entity);
}

// 구현 세부사항 숨김
class UserRepository implements Repository<User, Long> {
    private DatabaseConnection connection;

    @Override
    public User findById(Long id) {
        // SQL 쿼리 작성, 결과 매핑 등 내부 구현
        return null;
    }
    // ...
}
```

### 6.2 캡슐화 (Encapsulation)

```java
// 나쁜 예: 내부 상태 직접 노출
class BankAccountBad {
    public double balance;  // public 필드

    // 외부에서 직접 수정 가능
    // account.balance = -1000;  // 유효하지 않은 상태!
}

// 좋은 예: 캡슐화 적용
class BankAccount {
    private double balance;  // private 필드

    public BankAccount(double initialBalance) {
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.balance = initialBalance;
    }

    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (amount > balance) {
            throw new IllegalStateException("Insufficient funds");
        }
        this.balance -= amount;
    }
}
```

### 6.3 정보 은닉 수준

```java
public class Example {
    public int publicField;        // 어디서든 접근 가능
    protected int protectedField;  // 같은 패키지 + 자식 클래스
    int defaultField;              // 같은 패키지 (package-private)
    private int privateField;      // 해당 클래스만

    // 불변 객체 (Immutable Object)
    private final List<String> items;

    public Example(List<String> items) {
        // 방어적 복사 (Defensive Copy)
        this.items = new ArrayList<>(items);
    }

    public List<String> getItems() {
        // 불변 뷰 반환
        return Collections.unmodifiableList(items);
    }
}
```

---

## 참고 자료

- "Design Patterns: Elements of Reusable Object-Oriented Software" (GoF)
- "Effective Java" - Joshua Bloch
- "Clean Code" - Robert C. Martin
- "Head First Design Patterns" - Eric Freeman
- Spring Framework Documentation
