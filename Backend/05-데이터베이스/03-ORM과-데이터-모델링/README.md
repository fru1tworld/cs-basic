# ORM과 데이터 모델링

## 목차
1. [ORM 개요](#orm-개요)
2. [ORM 패턴](#orm-패턴)
3. [N+1 문제와 해결](#n1-문제와-해결)
4. [ERD 설계](#erd-설계)
5. [실무 모델링 패턴](#실무-모델링-패턴)
---

## ORM 개요

### ORM이란?

ORM(Object-Relational Mapping)은 객체 지향 프로그래밍 언어의 객체와 관계형 데이터베이스의 테이블 간의 매핑을 자동화하는 기술입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORM 동작 방식                                 │
│                                                                  │
│  ┌─────────────────┐         ┌─────────────────┐                │
│  │   Application   │         │    Database     │                │
│  │                 │         │                 │                │
│  │  ┌───────────┐  │         │  ┌───────────┐  │                │
│  │  │  User     │  │   ORM   │  │  users    │  │                │
│  │  │  Object   │◄─┼─────────┼──│  table    │  │                │
│  │  │           │  │         │  │           │  │                │
│  │  │ id: 1     │  │         │  │ id = 1    │  │                │
│  │  │ name: Kim │  │         │  │ name='Kim'│  │                │
│  │  │ email:... │  │         │  │ email=... │  │                │
│  │  └───────────┘  │         │  └───────────┘  │                │
│  └─────────────────┘         └─────────────────┘                │
│                                                                  │
│  장점:                       단점:                               │
│  - SQL 직접 작성 불필요      - 러닝 커브                         │
│  - DB 벤더 독립적            - 복잡한 쿼리 최적화 어려움          │
│  - 생산성 향상               - 성능 오버헤드                      │
│  - 타입 안정성               - N+1 문제 등 잠재적 이슈            │
└─────────────────────────────────────────────────────────────────┘
```

### 주요 ORM 프레임워크

| 언어 | ORM 프레임워크 |
|------|---------------|
| Java | Hibernate, JPA, MyBatis |
| Python | SQLAlchemy, Django ORM |
| JavaScript/TypeScript | TypeORM, Prisma, Sequelize |
| Ruby | ActiveRecord |
| Go | GORM, Ent |
| .NET | Entity Framework |

---

## ORM 패턴

### Active Record 패턴

객체가 데이터베이스 행을 감싸고, CRUD 메서드를 직접 포함합니다.

```python
# Python - Django ORM (Active Record)
class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'users'

# 사용법 - 객체가 직접 DB 조작
user = User(name='Kim', email='kim@example.com')
user.save()  # INSERT

user.name = 'Kim Updated'
user.save()  # UPDATE

users = User.objects.filter(name__startswith='Kim')  # SELECT
user.delete()  # DELETE
```

```ruby
# Ruby on Rails - ActiveRecord
class User < ApplicationRecord
  has_many :posts
  validates :email, presence: true, uniqueness: true
end

# 사용
user = User.new(name: 'Kim', email: 'kim@example.com')
user.save

User.find_by(email: 'kim@example.com')
User.where('created_at > ?', 1.day.ago)
```

**Active Record 특징:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Active Record 패턴                            │
│                                                                  │
│  ┌──────────────────────────────────────────┐                   │
│  │              User Model                   │                   │
│  │──────────────────────────────────────────│                   │
│  │  Attributes:                              │                   │
│  │   - id                                    │                   │
│  │   - name                                  │                   │
│  │   - email                                 │                   │
│  │──────────────────────────────────────────│                   │
│  │  Methods:                                 │                   │
│  │   - save()        → INSERT/UPDATE        │                   │
│  │   - delete()      → DELETE               │                   │
│  │   - find()        → SELECT               │                   │
│  │   - update()      → UPDATE               │                   │
│  │──────────────────────────────────────────│                   │
│  │  Business Logic:                          │                   │
│  │   - validate_email()                      │                   │
│  │   - send_welcome_email()                  │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                  │
│  장점: 간단, 빠른 개발                                           │
│  단점: 모델과 DB가 강하게 결합                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Data Mapper 패턴

도메인 객체와 데이터베이스를 분리하고, 별도의 매퍼 클래스가 변환을 담당합니다.

```typescript
// TypeScript - TypeORM (Data Mapper)
// Entity (도메인 모델)
@Entity('users')
class User {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    name: string;

    @Column({ unique: true })
    email: string;

    @CreateDateColumn()
    createdAt: Date;
}

// Repository (데이터 접근)
class UserRepository {
    constructor(private dataSource: DataSource) {}

    async findById(id: number): Promise<User | null> {
        return this.dataSource
            .getRepository(User)
            .findOne({ where: { id } });
    }

    async save(user: User): Promise<User> {
        return this.dataSource
            .getRepository(User)
            .save(user);
    }

    async findByEmail(email: string): Promise<User | null> {
        return this.dataSource
            .getRepository(User)
            .findOne({ where: { email } });
    }
}

// 사용
const userRepo = new UserRepository(dataSource);
const user = new User();
user.name = 'Kim';
user.email = 'kim@example.com';
await userRepo.save(user);
```

```python
# Python - SQLAlchemy (Data Mapper)
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import declarative_base, Session

Base = declarative_base()

# Entity
class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    email = Column(String(255), unique=True)
    created_at = Column(DateTime)

# Repository
class UserRepository:
    def __init__(self, session: Session):
        self.session = session

    def find_by_id(self, id: int) -> User | None:
        return self.session.query(User).filter(User.id == id).first()

    def save(self, user: User) -> User:
        self.session.add(user)
        self.session.commit()
        return user

    def find_by_email(self, email: str) -> User | None:
        return self.session.query(User).filter(User.email == email).first()
```

**Data Mapper 특징:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Data Mapper 패턴                              │
│                                                                  │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐ │
│  │  Domain Model  │    │     Mapper     │    │   Database     │ │
│  │                │    │  (Repository)  │    │                │ │
│  │  User Entity   │◄──►│                │◄──►│  users table   │ │
│  │                │    │  UserRepository│    │                │ │
│  │  - id          │    │  - findById()  │    │  - id          │ │
│  │  - name        │    │  - save()      │    │  - name        │ │
│  │  - email       │    │  - delete()    │    │  - email       │ │
│  │                │    │                │    │                │ │
│  │  비즈니스 로직  │    │   SQL 생성    │    │                │ │
│  └────────────────┘    └────────────────┘    └────────────────┘ │
│                                                                  │
│  장점: 관심사 분리, 테스트 용이, 유연성                         │
│  단점: 복잡성 증가, 더 많은 코드                                │
└─────────────────────────────────────────────────────────────────┘
```

### Active Record vs Data Mapper

| 특성 | Active Record | Data Mapper |
|------|---------------|-------------|
| 결합도 | 모델 = 테이블 | 모델 ≠ 테이블 |
| 복잡성 | 낮음 | 높음 |
| 테스트 | 어려움 (DB 의존) | 용이 (Mock 가능) |
| 적합한 경우 | 간단한 CRUD | 복잡한 도메인 |
| 대표 프레임워크 | Rails, Django | JPA, SQLAlchemy |

---

## N+1 문제와 해결

### N+1 문제란?

```python
# N+1 문제 발생 코드
# 1번 쿼리: 모든 게시글 조회
posts = Post.objects.all()  # SELECT * FROM posts

for post in posts:  # N번의 추가 쿼리
    print(post.author.name)  # SELECT * FROM users WHERE id = ?

# 결과: 게시글 100개면 101번의 쿼리 실행!
```

```
┌─────────────────────────────────────────────────────────────────┐
│                      N+1 문제                                    │
│                                                                  │
│  1. SELECT * FROM posts;  -- 100 rows                           │
│                                                                  │
│  2. SELECT * FROM users WHERE id = 1;   (post 1의 author)       │
│  3. SELECT * FROM users WHERE id = 2;   (post 2의 author)       │
│  4. SELECT * FROM users WHERE id = 1;   (post 3의 author)       │
│  ...                                                             │
│  101. SELECT * FROM users WHERE id = 50; (post 100의 author)    │
│                                                                  │
│  총 101번 쿼리 실행 → 심각한 성능 저하                           │
└─────────────────────────────────────────────────────────────────┘
```

### 해결 방법 1: Eager Loading (즉시 로딩)

```python
# Django - select_related (FK, OneToOne)
posts = Post.objects.select_related('author').all()
# SELECT posts.*, users.* FROM posts
# JOIN users ON posts.author_id = users.id

# Django - prefetch_related (ManyToMany, Reverse FK)
posts = Post.objects.prefetch_related('comments').all()
# SELECT * FROM posts
# SELECT * FROM comments WHERE post_id IN (1, 2, 3, ...)

# 조합 사용
posts = Post.objects.select_related('author').prefetch_related('comments', 'tags').all()
```

```python
# SQLAlchemy - joinedload
from sqlalchemy.orm import joinedload

posts = session.query(Post).options(joinedload(Post.author)).all()

# selectinload (별도 쿼리로 로드)
from sqlalchemy.orm import selectinload

posts = session.query(Post).options(selectinload(Post.comments)).all()
```

```typescript
// TypeORM - relations
const posts = await postRepository.find({
    relations: ['author', 'comments']
});

// QueryBuilder
const posts = await postRepository
    .createQueryBuilder('post')
    .leftJoinAndSelect('post.author', 'author')
    .leftJoinAndSelect('post.comments', 'comments')
    .getMany();
```

### 해결 방법 2: Batch Loading

```python
# Django - Prefetch 객체로 세밀한 제어
from django.db.models import Prefetch

posts = Post.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.filter(is_approved=True).order_by('-created_at')[:5]
    )
).all()
```

```javascript
// DataLoader 패턴 (GraphQL에서 주로 사용)
const userLoader = new DataLoader(async (userIds) => {
    const users = await User.findByIds(userIds);
    const userMap = new Map(users.map(u => [u.id, u]));
    return userIds.map(id => userMap.get(id));
});

// 사용
const user = await userLoader.load(userId);  // 자동으로 배치 처리
```

### 해결 방법 3: Subquery

```python
# Django - Subquery로 필요한 데이터만
from django.db.models import Subquery, OuterRef

latest_comment = Comment.objects.filter(
    post=OuterRef('pk')
).order_by('-created_at')

posts = Post.objects.annotate(
    latest_comment_text=Subquery(latest_comment.values('text')[:1])
)
```

### Lazy Loading 설정

```python
# SQLAlchemy - 기본 로딩 전략 설정
class Post(Base):
    # lazy='select' (기본값): 접근 시 쿼리
    # lazy='joined': 항상 JOIN
    # lazy='subquery': 항상 서브쿼리
    # lazy='selectin': 항상 IN 쿼리
    author = relationship('User', lazy='joined')
    comments = relationship('Comment', lazy='selectin')
```

```typescript
// TypeORM - 기본 eager 설정
@Entity()
class Post {
    @ManyToOne(() => User, { eager: true })  // 항상 로드
    author: User;

    @OneToMany(() => Comment, comment => comment.post, { lazy: true })  // 접근 시 로드
    comments: Promise<Comment[]>;
}
```

---

## ERD 설계

### 관계 유형

```
┌─────────────────────────────────────────────────────────────────┐
│                      관계 유형                                   │
│                                                                  │
│  1:1 (One-to-One)                                               │
│  ┌──────────┐         ┌──────────┐                              │
│  │   User   │─────────│ Profile  │                              │
│  └──────────┘         └──────────┘                              │
│   user_id (PK)         user_id (PK, FK)                         │
│                                                                  │
│  1:N (One-to-Many)                                              │
│  ┌──────────┐         ┌──────────┐                              │
│  │   User   │────────<│   Post   │                              │
│  └──────────┘         └──────────┘                              │
│   id (PK)              author_id (FK)                           │
│                                                                  │
│  N:M (Many-to-Many)                                             │
│  ┌──────────┐    ┌────────────┐    ┌──────────┐                 │
│  │   Post   │───<│ post_tags  │>───│   Tag    │                 │
│  └──────────┘    └────────────┘    └──────────┘                 │
│   id (PK)         post_id (FK)      id (PK)                     │
│                   tag_id (FK)                                    │
│                                                                  │
│  Self-Referencing                                               │
│  ┌──────────────────────┐                                       │
│  │      Employee        │                                       │
│  │ ─────────────────── │                                       │
│  │ id (PK)              │                                       │
│  │ manager_id (FK → id) │◄──┐                                   │
│  └──────────────────────┘   │                                   │
│            │                │                                    │
│            └────────────────┘                                    │
└─────────────────────────────────────────────────────────────────┘
```

### ERD 표기법 (Crow's Foot)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Crow's Foot 표기법                            │
│                                                                  │
│  ───────────────  One (1)                                       │
│  ──────────○────  Zero or One (0..1)                           │
│  ──────────<────  Many (N)                                      │
│  ──────────○<───  Zero or Many (0..N)                          │
│  ──────────│<───  One or Many (1..N)                           │
│                                                                  │
│  예시: 사용자와 주문                                            │
│                                                                  │
│  ┌──────────┐                    ┌──────────┐                   │
│  │   User   │──────────○<────────│  Order   │                   │
│  └──────────┘                    └──────────┘                   │
│                                                                  │
│  "사용자는 0개 이상의 주문을 가질 수 있다"                       │
│  "주문은 정확히 1명의 사용자에게 속한다"                         │
└─────────────────────────────────────────────────────────────────┘
```

### 실무 ERD 예시: 이커머스

```sql
-- 핵심 엔티티
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(12, 2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    category_id BIGINT REFERENCES categories(id),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    total_amount DECIMAL(15, 2) NOT NULL,
    shipping_address_id BIGINT REFERENCES addresses(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CHECK (status IN ('pending', 'paid', 'shipped', 'delivered', 'cancelled'))
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(12, 2) NOT NULL,
    UNIQUE (order_id, product_id)
);

-- N:M 관계
CREATE TABLE product_tags (
    product_id BIGINT REFERENCES products(id) ON DELETE CASCADE,
    tag_id BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);

-- 셀프 참조 (카테고리 계층)
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id BIGINT REFERENCES categories(id),
    depth INTEGER NOT NULL DEFAULT 0
);
```

---

## 실무 모델링 패턴

### Soft Delete

```python
# 물리 삭제 대신 논리 삭제
class User(Base):
    __tablename__ = 'users'

    id = Column(BigInteger, primary_key=True)
    email = Column(String(255), unique=True)
    is_deleted = Column(Boolean, default=False)
    deleted_at = Column(DateTime, nullable=True)

# 기본 쿼리에 필터 적용
class UserRepository:
    def find_active_users(self):
        return session.query(User).filter(User.is_deleted == False).all()
```

### Audit Trail (감사 로그)

```python
# 변경 이력 추적
class AuditMixin:
    created_at = Column(DateTime, default=datetime.utcnow)
    created_by = Column(BigInteger)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    updated_by = Column(BigInteger)

class User(Base, AuditMixin):
    __tablename__ = 'users'
    id = Column(BigInteger, primary_key=True)
    # ...

# 또는 별도 이력 테이블
CREATE TABLE user_history (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    action VARCHAR(20) NOT NULL,  -- 'INSERT', 'UPDATE', 'DELETE'
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by BIGINT
);
```

### Polymorphic Association (다형성 관계)

```python
# 여러 엔티티에 공통 관계 (예: 댓글)
class Comment(Base):
    __tablename__ = 'comments'

    id = Column(BigInteger, primary_key=True)
    content = Column(Text, nullable=False)
    # 다형성 관계
    commentable_type = Column(String(50))  # 'Post', 'Product', 'Article'
    commentable_id = Column(BigInteger)
    created_at = Column(DateTime, default=datetime.utcnow)

# 사용
post_comments = session.query(Comment).filter(
    Comment.commentable_type == 'Post',
    Comment.commentable_id == post_id
).all()
```

### Value Object 패턴

```python
# 복합 값 객체
class Address(Base):
    __tablename__ = 'addresses'

    id = Column(BigInteger, primary_key=True)
    street = Column(String(255))
    city = Column(String(100))
    state = Column(String(100))
    postal_code = Column(String(20))
    country = Column(String(100))

class User(Base):
    # Embedded Value Object
    home_address_id = Column(BigInteger, ForeignKey('addresses.id'))
    work_address_id = Column(BigInteger, ForeignKey('addresses.id'))

    home_address = relationship('Address', foreign_keys=[home_address_id])
    work_address = relationship('Address', foreign_keys=[work_address_id])
```

---

## 참고 자료

- [Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/) - Martin Fowler
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)
- [TypeORM Documentation](https://typeorm.io/)
- [Django ORM Optimization](https://docs.djangoproject.com/en/4.2/topics/db/optimization/)
- Designing Data-Intensive Applications by Martin Kleppmann
