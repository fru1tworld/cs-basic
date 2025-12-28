# GraphQL

## 목차
1. [GraphQL 개요](#graphql-개요)
2. [Schema Definition Language (SDL)](#schema-definition-language-sdl)
3. [Query, Mutation, Subscription](#query-mutation-subscription)
4. [Resolver 패턴](#resolver-패턴)
5. [N+1 문제와 DataLoader](#n1-문제와-dataloader)
6. [REST vs GraphQL 비교](#rest-vs-graphql-비교)
7. [실전 구현 예제](#실전-구현-예제)
---

## GraphQL 개요

GraphQL은 Facebook이 2012년에 개발하고 2015년에 오픈소스로 공개한 **API를 위한 쿼리 언어**입니다. 클라이언트가 필요한 데이터를 정확하게 요청할 수 있게 해줍니다.

### 핵심 특징

```graphql
# 클라이언트가 필요한 데이터만 요청
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}
```

| 특징 | 설명 |
|------|------|
| 강력한 타입 시스템 | 스키마로 API 구조 정의 |
| 단일 엔드포인트 | 모든 요청이 `/graphql`로 |
| 클라이언트 주도 | 클라이언트가 필요한 필드 선택 |
| 인트로스펙션 | 스키마 자체 조회 가능 |

### GraphQL vs REST 요청 비교

```
# REST - 여러 번의 요청 필요
GET /users/123
GET /users/123/posts
GET /users/123/followers

# GraphQL - 단일 요청으로 해결
POST /graphql
{
  user(id: "123") {
    name
    posts { title }
    followers { name }
  }
}
```

---

## Schema Definition Language (SDL)

SDL은 GraphQL 스키마를 정의하는 언어입니다.

### 스칼라 타입

```graphql
# 내장 스칼라 타입
type Example {
  id: ID!           # 고유 식별자
  name: String!     # 문자열 (필수)
  age: Int          # 정수 (nullable)
  score: Float      # 부동소수점
  isActive: Boolean # 불리언
}

# 커스텀 스칼라 타입 정의
scalar DateTime
scalar JSON
scalar Email
```

### 객체 타입

```graphql
type User {
  id: ID!
  username: String!
  email: String!
  createdAt: DateTime!
  profile: Profile
  posts: [Post!]!       # Post 배열, 배열과 요소 모두 non-null
  followers: [User!]    # 배열은 nullable, 요소는 non-null
}

type Profile {
  bio: String
  avatar: String
  website: String
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  createdAt: DateTime!
  updatedAt: DateTime
}

type Comment {
  id: ID!
  text: String!
  author: User!
  post: Post!
  createdAt: DateTime!
}
```

### Non-null 표기법

```graphql
# ! 의미 이해
field: String      # nullable - null 반환 가능
field: String!     # non-null - null 반환 불가
field: [String]    # nullable 배열, nullable 요소
field: [String]!   # non-null 배열, nullable 요소
field: [String!]   # nullable 배열, non-null 요소
field: [String!]!  # non-null 배열, non-null 요소
```

### Enum 타입

```graphql
enum UserRole {
  ADMIN
  MODERATOR
  USER
  GUEST
}

enum PostStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

type User {
  id: ID!
  role: UserRole!
}
```

### Interface 타입

```graphql
interface Node {
  id: ID!
}

interface Timestamped {
  createdAt: DateTime!
  updatedAt: DateTime
}

type User implements Node & Timestamped {
  id: ID!
  name: String!
  createdAt: DateTime!
  updatedAt: DateTime
}

type Post implements Node & Timestamped {
  id: ID!
  title: String!
  createdAt: DateTime!
  updatedAt: DateTime
}
```

### Union 타입

```graphql
union SearchResult = User | Post | Comment

type Query {
  search(query: String!): [SearchResult!]!
}

# 클라이언트에서 사용
query {
  search(query: "graphql") {
    ... on User {
      name
      email
    }
    ... on Post {
      title
      content
    }
    ... on Comment {
      text
    }
  }
}
```

### Input 타입

```graphql
input CreateUserInput {
  username: String!
  email: String!
  password: String!
  profile: ProfileInput
}

input ProfileInput {
  bio: String
  avatar: String
  website: String
}

input UpdateUserInput {
  username: String
  email: String
  profile: ProfileInput
}

input PostFilterInput {
  status: PostStatus
  authorId: ID
  createdAfter: DateTime
  createdBefore: DateTime
}
```

### Directive

```graphql
# 내장 Directive
type User {
  id: ID!
  name: String!
  email: String! @deprecated(reason: "Use 'emailAddress' instead")
  emailAddress: String!
  secretField: String @skip(if: true)
}

# 커스텀 Directive 정의
directive @auth(requires: UserRole!) on FIELD_DEFINITION
directive @cacheControl(maxAge: Int!) on FIELD_DEFINITION

type Query {
  me: User @auth(requires: USER)
  adminDashboard: Dashboard @auth(requires: ADMIN)
  popularPosts: [Post!]! @cacheControl(maxAge: 300)
}
```

---

## Query, Mutation, Subscription

### Query - 데이터 조회

```graphql
type Query {
  # 단일 리소스 조회
  user(id: ID!): User
  post(id: ID!): Post

  # 컬렉션 조회
  users(
    first: Int
    after: String
    filter: UserFilterInput
  ): UserConnection!

  posts(
    first: Int
    after: String
    filter: PostFilterInput
    orderBy: PostOrderByInput
  ): PostConnection!

  # 검색
  search(query: String!): [SearchResult!]!

  # 현재 사용자
  me: User
}

# Relay 스타일 페이지네이션
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

**Query 실행 예시:**

```graphql
query GetUserWithPosts($userId: ID!, $first: Int = 10) {
  user(id: $userId) {
    id
    name
    email
    posts(first: $first) {
      edges {
        node {
          id
          title
          content
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

### Mutation - 데이터 변경

```graphql
type Mutation {
  # 생성
  createUser(input: CreateUserInput!): CreateUserPayload!
  createPost(input: CreatePostInput!): CreatePostPayload!

  # 수정
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
  updatePost(id: ID!, input: UpdatePostInput!): UpdatePostPayload!

  # 삭제
  deleteUser(id: ID!): DeleteUserPayload!
  deletePost(id: ID!): DeletePostPayload!

  # 인증
  login(email: String!, password: String!): AuthPayload!
  logout: Boolean!
  refreshToken: AuthPayload!
}

# Payload 패턴 - 에러 처리와 확장성
type CreateUserPayload {
  user: User
  errors: [Error!]
  success: Boolean!
}

type AuthPayload {
  token: String
  refreshToken: String
  user: User
  errors: [Error!]
}

type Error {
  field: String
  message: String!
  code: String!
}
```

**Mutation 실행 예시:**

```graphql
mutation CreateNewUser($input: CreateUserInput!) {
  createUser(input: $input) {
    success
    user {
      id
      name
      email
    }
    errors {
      field
      message
      code
    }
  }
}

# Variables
{
  "input": {
    "username": "johndoe",
    "email": "john@example.com",
    "password": "securePassword123",
    "profile": {
      "bio": "Software Engineer"
    }
  }
}
```

### Subscription - 실시간 데이터

```graphql
type Subscription {
  # 새 메시지 수신
  messageAdded(channelId: ID!): Message!

  # 게시물 변경 감지
  postUpdated(postId: ID!): Post!

  # 온라인 상태 변경
  userStatusChanged(userId: ID!): UserStatus!

  # 알림
  notificationReceived: Notification!
}

type Message {
  id: ID!
  content: String!
  sender: User!
  createdAt: DateTime!
}

type UserStatus {
  user: User!
  status: OnlineStatus!
  lastSeen: DateTime
}

enum OnlineStatus {
  ONLINE
  AWAY
  OFFLINE
}
```

**클라이언트에서 Subscription 사용:**

```javascript
// Apollo Client
const MESSAGE_SUBSCRIPTION = gql`
  subscription OnMessageAdded($channelId: ID!) {
    messageAdded(channelId: $channelId) {
      id
      content
      sender {
        id
        name
      }
      createdAt
    }
  }
`;

function ChatRoom({ channelId }) {
  const { data, loading } = useSubscription(MESSAGE_SUBSCRIPTION, {
    variables: { channelId },
  });

  useEffect(() => {
    if (data?.messageAdded) {
      // 새 메시지 처리
      addMessage(data.messageAdded);
    }
  }, [data]);
}
```

---

## Resolver 패턴

Resolver는 스키마의 각 필드에 대한 데이터를 가져오는 함수입니다.

### Resolver 구조

```javascript
// resolver 함수 시그니처
const resolver = (parent, args, context, info) => {
  // parent: 부모 타입의 resolver가 반환한 값
  // args: 필드에 전달된 인수
  // context: 모든 resolver가 공유하는 객체 (인증 정보, DB 연결 등)
  // info: 쿼리 실행 정보 (필드 이름, 경로 등)
};
```

### 기본 Resolver 구현

```javascript
const resolvers = {
  Query: {
    // 단일 조회
    user: async (_, { id }, { dataSources }) => {
      return dataSources.userAPI.getUser(id);
    },

    // 컬렉션 조회
    users: async (_, { first, after, filter }, { dataSources }) => {
      return dataSources.userAPI.getUsers({ first, after, filter });
    },

    // 현재 사용자 (인증 필요)
    me: async (_, __, { user }) => {
      if (!user) {
        throw new AuthenticationError('Not authenticated');
      }
      return user;
    },
  },

  Mutation: {
    createUser: async (_, { input }, { dataSources }) => {
      try {
        const user = await dataSources.userAPI.createUser(input);
        return { success: true, user, errors: [] };
      } catch (error) {
        return {
          success: false,
          user: null,
          errors: [{ message: error.message, code: 'CREATE_FAILED' }]
        };
      }
    },

    login: async (_, { email, password }, { dataSources }) => {
      const user = await dataSources.userAPI.login(email, password);
      if (!user) {
        return { errors: [{ message: 'Invalid credentials', code: 'AUTH_FAILED' }] };
      }
      const token = generateToken(user);
      return { token, user, errors: [] };
    },
  },

  // 타입별 필드 Resolver
  User: {
    // 관계 필드 - 별도의 데이터 소스에서 조회
    posts: async (user, { first }, { dataSources }) => {
      return dataSources.postAPI.getPostsByAuthor(user.id, { first });
    },

    followers: async (user, _, { dataSources }) => {
      return dataSources.userAPI.getFollowers(user.id);
    },

    // 계산 필드
    fullName: (user) => {
      return `${user.firstName} ${user.lastName}`;
    },

    // 비동기 계산 필드
    followerCount: async (user, _, { dataSources }) => {
      return dataSources.userAPI.getFollowerCount(user.id);
    },
  },

  Post: {
    author: async (post, _, { dataSources }) => {
      return dataSources.userAPI.getUser(post.authorId);
    },

    comments: async (post, { first }, { dataSources }) => {
      return dataSources.commentAPI.getCommentsByPost(post.id, { first });
    },
  },
};
```

### Context 설정

```javascript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: async ({ req }) => {
    // 토큰에서 사용자 추출
    const token = req.headers.authorization?.replace('Bearer ', '');
    let user = null;

    if (token) {
      try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        user = await User.findById(decoded.userId);
      } catch (error) {
        // 토큰 검증 실패 - user는 null로 유지
      }
    }

    return {
      user,
      dataSources: {
        userAPI: new UserAPI(),
        postAPI: new PostAPI(),
        commentAPI: new CommentAPI(),
      },
    };
  },
});
```

### 에러 처리 패턴

```javascript
import {
  ApolloError,
  AuthenticationError,
  ForbiddenError,
  UserInputError
} from 'apollo-server';

const resolvers = {
  Query: {
    user: async (_, { id }, { dataSources, user }) => {
      // 인증 확인
      if (!user) {
        throw new AuthenticationError('Must be logged in');
      }

      // 입력 검증
      if (!id || id.length < 1) {
        throw new UserInputError('Invalid user ID', {
          argumentName: 'id',
        });
      }

      const foundUser = await dataSources.userAPI.getUser(id);

      // 리소스 없음
      if (!foundUser) {
        throw new ApolloError('User not found', 'USER_NOT_FOUND', {
          id,
        });
      }

      // 권한 확인
      if (foundUser.isPrivate && foundUser.id !== user.id) {
        throw new ForbiddenError('Cannot access private profile');
      }

      return foundUser;
    },
  },
};
```

---

## N+1 문제와 DataLoader

### N+1 문제란?

```graphql
query {
  posts {       # 1번의 쿼리로 N개의 게시물 조회
    id
    title
    author {    # 각 게시물마다 1번씩 저자 조회 → N번 추가 쿼리
      name
    }
  }
}
```

```sql
-- 실제 실행되는 쿼리 (N+1 문제 발생)
SELECT * FROM posts;                    -- 1번
SELECT * FROM users WHERE id = 1;       -- +N번
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 3;
...
```

### DataLoader로 해결

DataLoader는 요청을 배치(batch)하고 캐싱하여 N+1 문제를 해결합니다.

```javascript
import DataLoader from 'dataloader';

// DataLoader 생성
const createLoaders = () => ({
  // 사용자 로더
  userLoader: new DataLoader(async (userIds) => {
    // 한 번의 쿼리로 모든 사용자 조회
    const users = await User.find({ _id: { $in: userIds } });

    // 입력 순서대로 결과 정렬 (DataLoader 요구사항)
    const userMap = new Map(users.map(user => [user.id.toString(), user]));
    return userIds.map(id => userMap.get(id.toString()) || null);
  }),

  // 게시물 로더 (저자별)
  postsByAuthorLoader: new DataLoader(async (authorIds) => {
    const posts = await Post.find({ authorId: { $in: authorIds } });

    // authorId별로 그룹화
    const postsByAuthor = new Map();
    posts.forEach(post => {
      const authorId = post.authorId.toString();
      if (!postsByAuthor.has(authorId)) {
        postsByAuthor.set(authorId, []);
      }
      postsByAuthor.get(authorId).push(post);
    });

    return authorIds.map(id => postsByAuthor.get(id.toString()) || []);
  }),
});

// Context에서 요청마다 새 DataLoader 생성
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => ({
    user: getUser(req),
    loaders: createLoaders(),  // 요청별 새 인스턴스
  }),
});

// Resolver에서 DataLoader 사용
const resolvers = {
  Post: {
    author: (post, _, { loaders }) => {
      return loaders.userLoader.load(post.authorId);
    },
  },

  User: {
    posts: (user, _, { loaders }) => {
      return loaders.postsByAuthorLoader.load(user.id);
    },
  },
};
```

### DataLoader 동작 원리

```
1. 이벤트 루프 틱 내의 모든 .load() 호출 수집
2. 다음 틱에서 batchLoadFn 한 번 호출
3. 결과를 각 .load() 호출에 매핑

Timeline:
┌─────────────────────────────────────────────────────────┐
│ Tick 1: load(1), load(2), load(3) 수집                  │
├─────────────────────────────────────────────────────────┤
│ Tick 2: batchLoadFn([1, 2, 3]) 실행 → 단일 DB 쿼리      │
├─────────────────────────────────────────────────────────┤
│ Tick 3: 각 Promise에 결과 전달                          │
└─────────────────────────────────────────────────────────┘
```

### DataLoader 모범 사례

```javascript
// 1. 요청마다 새 DataLoader 인스턴스 생성
// - 캐시 격리
// - 메모리 누수 방지
context: () => ({ loaders: createLoaders() })

// 2. 결과 순서 보장
const batchLoadFn = async (keys) => {
  const items = await Model.find({ _id: { $in: keys } });
  const itemMap = new Map(items.map(item => [item.id, item]));

  // keys 순서대로 반환 (필수!)
  return keys.map(key => itemMap.get(key) || null);
};

// 3. 에러 처리
const userLoader = new DataLoader(async (ids) => {
  const users = await User.find({ _id: { $in: ids } });
  const userMap = new Map(users.map(u => [u.id, u]));

  return ids.map(id => {
    const user = userMap.get(id);
    if (!user) {
      // 개별 키에 대한 에러는 Error 객체 반환
      return new Error(`User not found: ${id}`);
    }
    return user;
  });
});

// 4. 캐싱 옵션
const userLoader = new DataLoader(batchFn, {
  cache: true,         // 기본값: true
  cacheMap: new Map(), // 커스텀 캐시 맵
  batchScheduleFn: (callback) => setTimeout(callback, 10), // 배치 스케줄링
  maxBatchSize: 100,   // 최대 배치 크기
});
```

---

## REST vs GraphQL 비교

### 요청/응답 비교

```
┌─────────────────────────────────────────────────────────┐
│                        REST                              │
├─────────────────────────────────────────────────────────┤
│ GET /users/123                                          │
│ GET /users/123/posts                                    │
│ GET /users/123/followers                                │
│                                                         │
│ → 3번의 요청, 불필요한 데이터 포함 가능                    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                       GraphQL                            │
├─────────────────────────────────────────────────────────┤
│ POST /graphql                                           │
│ {                                                       │
│   user(id: "123") {                                     │
│     name                                                │
│     posts { title }                                     │
│     followers { name }                                  │
│   }                                                     │
│ }                                                       │
│                                                         │
│ → 1번의 요청, 필요한 데이터만 수신                        │
└─────────────────────────────────────────────────────────┘
```

### 상세 비교표

| 측면 | REST | GraphQL |
|------|------|---------|
| **엔드포인트** | 다중 엔드포인트 | 단일 엔드포인트 |
| **데이터 페칭** | Over/Under-fetching 발생 | 정확한 데이터 페칭 |
| **버전 관리** | URI 또는 헤더 버전 | 스키마 진화 (additive) |
| **캐싱** | HTTP 캐싱 용이 | 복잡한 캐싱 전략 필요 |
| **타입 시스템** | 별도 문서/스키마 필요 | 내장 스키마 시스템 |
| **에러 처리** | HTTP 상태 코드 | 항상 200, errors 필드 |
| **파일 업로드** | 네이티브 지원 | 별도 스펙 필요 |
| **실시간** | 별도 구현 (WebSocket) | Subscription 내장 |
| **학습 곡선** | 낮음 | 높음 |
| **도구 생태계** | 매우 성숙 | 빠르게 성장 중 |

### Over-fetching & Under-fetching

```javascript
// Over-fetching (REST)
// 사용자 이름만 필요한데 전체 데이터 수신
GET /users/123
Response: {
  id: 123,
  name: "John",
  email: "john@example.com",  // 불필요
  address: { ... },           // 불필요
  preferences: { ... },       // 불필요
  createdAt: "..."            // 불필요
}

// Under-fetching (REST)
// 관련 데이터를 위해 추가 요청 필요
GET /users/123          // 1차
GET /users/123/posts    // 2차
GET /posts/1/comments   // 3차

// GraphQL - 정확한 데이터만 요청
query {
  user(id: "123") {
    name
    posts {
      title
      comments {
        text
      }
    }
  }
}
```

### 캐싱 비교

```javascript
// REST - HTTP 캐싱 활용 용이
GET /users/123 HTTP/1.1
Cache-Control: max-age=3600

// GraphQL - 캐싱 전략이 복잡
// 1. 응답 레벨 캐싱 (제한적)
POST /graphql
Query: { user(id: 123) { ... } }
// 동일 쿼리에 대해서만 캐시 적중

// 2. 정규화된 캐시 (Apollo Client)
// 엔티티별로 캐시하고 참조로 연결
cache: {
  'User:123': { id: 123, name: 'John', posts: [Ref('Post:1'), Ref('Post:2')] },
  'Post:1': { id: 1, title: 'First Post' },
  'Post:2': { id: 2, title: 'Second Post' }
}

// 3. 서버 사이드 캐싱 (Persisted Queries)
// 쿼리 해시로 캐시 키 생성
GET /graphql?extensions={"persistedQuery":{"sha256Hash":"..."}}
```

### 언제 무엇을 선택할까?

**REST가 적합한 경우:**
- 리소스 중심의 단순한 CRUD API
- HTTP 캐싱이 중요한 경우
- 파일 업로드/다운로드가 많은 경우
- 팀이 REST에 익숙한 경우
- 퍼블릭 API (광범위한 클라이언트 지원)

**GraphQL이 적합한 경우:**
- 복잡한 데이터 관계
- 다양한 클라이언트 (웹, 모바일, IoT)
- 빠르게 변하는 요구사항
- 프론트엔드 주도 개발
- 실시간 기능이 필요한 경우

---

## 실전 구현 예제

### Apollo Server 전체 구조

```javascript
// src/index.js
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import express from 'express';
import { typeDefs } from './schema';
import { resolvers } from './resolvers';
import { createLoaders } from './loaders';
import { authenticate } from './auth';

const app = express();

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    // 로깅 플러그인
    {
      requestDidStart: async () => ({
        didResolveOperation: async (context) => {
          console.log(`Operation: ${context.operationName}`);
        },
        didEncounterErrors: async (context) => {
          console.error('Errors:', context.errors);
        },
      }),
    },
  ],
});

await server.start();

app.use(
  '/graphql',
  express.json(),
  expressMiddleware(server, {
    context: async ({ req }) => ({
      user: await authenticate(req),
      loaders: createLoaders(),
    }),
  })
);

app.listen(4000, () => {
  console.log('Server ready at http://localhost:4000/graphql');
});
```

### 스키마 분리

```javascript
// src/schema/user.js
export const userTypeDefs = `
  type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
  }

  input CreateUserInput {
    name: String!
    email: String!
    password: String!
  }

  extend type Query {
    user(id: ID!): User
    users: [User!]!
    me: User
  }

  extend type Mutation {
    createUser(input: CreateUserInput!): User!
  }
`;

// src/schema/post.js
export const postTypeDefs = `
  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    createdAt: DateTime!
  }

  extend type Query {
    post(id: ID!): Post
    posts(authorId: ID): [Post!]!
  }
`;

// src/schema/index.js
import { userTypeDefs } from './user';
import { postTypeDefs } from './post';

const baseTypeDefs = `
  scalar DateTime

  type Query {
    _empty: String
  }

  type Mutation {
    _empty: String
  }
`;

export const typeDefs = [baseTypeDefs, userTypeDefs, postTypeDefs];
```

### Resolver 분리

```javascript
// src/resolvers/user.js
export const userResolvers = {
  Query: {
    user: async (_, { id }, { loaders }) => loaders.userLoader.load(id),
    users: async (_, __, { dataSources }) => dataSources.userAPI.getAll(),
    me: (_, __, { user }) => user,
  },

  Mutation: {
    createUser: async (_, { input }, { dataSources }) => {
      return dataSources.userAPI.create(input);
    },
  },

  User: {
    posts: (user, _, { loaders }) => loaders.postsByAuthorLoader.load(user.id),
  },
};

// src/resolvers/post.js
export const postResolvers = {
  Query: {
    post: async (_, { id }, { loaders }) => loaders.postLoader.load(id),
    posts: async (_, { authorId }, { dataSources }) => {
      if (authorId) {
        return dataSources.postAPI.getByAuthor(authorId);
      }
      return dataSources.postAPI.getAll();
    },
  },

  Post: {
    author: (post, _, { loaders }) => loaders.userLoader.load(post.authorId),
  },
};

// src/resolvers/index.js
import { mergeResolvers } from '@graphql-tools/merge';
import { userResolvers } from './user';
import { postResolvers } from './post';

export const resolvers = mergeResolvers([userResolvers, postResolvers]);
```

---

## 참고 자료

- [GraphQL 공식 문서](https://graphql.org/learn/)
- [Apollo GraphQL Documentation](https://www.apollographql.com/docs/)
- [GraphQL Specification](https://spec.graphql.org/)
- [DataLoader Documentation](https://github.com/graphql/dataloader)
- [Relay GraphQL Server Specification](https://relay.dev/docs/guides/graphql-server-specification/)
- [How to GraphQL](https://www.howtographql.com/)
