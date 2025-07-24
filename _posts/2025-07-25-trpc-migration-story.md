---
layout: single
title: "복잡한 마이크로서비스에서 NestJS + tRPC로: 개발 병목을 해결한 아키텍처 개선기"
date: 2025-07-25
categories: [백엔드, 아키텍처]
tags: [tRPC, GraphQL, NestJS, 마이크로서비스, 아키텍처, 리팩토링, Go, gRPC]
---

# 복잡한 마이크로서비스에서 NestJS + tRPC로: 개발 병목을 해결한 아키텍처 개선기

> 🚀 **프로젝트 요약**  
> Go 기반 마이크로서비스의 복잡성과 GraphQL Code Generation의 의존성 문제를 해결하기 위해 NestJS + tRPC 하이브리드 아키텍처로 전환하여 개발 생산성을 대폭 향상시킨 경험을 공유합니다.

---

## 😵 기존 아키텍처의 복잡성 지옥

### 당시 우리의 마이크로서비스 구조

```
go-server → grpc (protobuf) → idl → bff → client
```

언뜻 보면 깔끔해 보이지만, 실제로는 **간단한 API 하나 만드는 것도 지옥**이었습니다.

### 🔥 실제로 겪었던 문제들

#### 1. **간단한 로직도 Go 서버까지 가야 하는 문제**

사용자 프로필에 새로운 필드 하나를 추가하려고 해도:

```go
// 1. Go 서버에 비즈니스 로직 추가
func (s *UserService) GetUserProfile(ctx context.Context, req *pb.GetUserRequest) (*pb.UserResponse, error) {
    // 복잡한 비즈니스 로직을 Go로 구현해야 함
    user := s.userRepo.FindByID(req.UserId)
    profile := s.buildUserProfile(user) // 이런 간단한 것도...
    return &pb.UserResponse{Profile: profile}, nil
}
```

```typescript
// 2. BFF에서는 단순 중계만
async getUserProfile(userId: string) {
  const response = await this.grpcClient.getUserProfile({ userId });
  return response.profile; // 그냥 전달만...
}
```

프론트엔드 개발자가 간단한 데이터 가공도 직접 할 수 없었습니다. 😤

#### 2. **GraphQL Code Generation의 의존성 지옥**

더 큰 문제는 모든 프로젝트가 **하나의 GraphQL 스키마를 공유**한다는 점이었습니다:

```typescript
// 모든 프로젝트에서 이 타입들을 import
import {
  User,
  Product,
  Order,
  Payment,
  // ... 수백 개의 타입들
} from "@shared/graphql-types";
```

**한 팀이 스키마를 수정하면?**

1. 🔄 Code Generation 실행
2. 🧪 모든 프로젝트에서 Build Test 필요
3. 💥 어딘가에서 타입 에러 발생
4. 🔁 다시 수정하고 반복...

#### 3. **실제 일어났던 사건**

> 💬 **팀원 A**: "상품 스키마만 살짝 수정했는데..."  
> 💬 **팀원 B**: "우리 주문 시스템 빌드가 깨졌어요!"  
> 💬 **팀원 C**: "결제 쪽도 에러 나네요..."  
> 💬 **나**: "... 😵‍💫"

**결과**: 간단한 수정 하나가 전체 팀의 하루를 날려버렸습니다.

---

## 💡 "이대로는 안 되겠다" - 개선 제안

### 첫 번째 제안: BFF를 진짜 백엔드로 만들자

```typescript
// 기존: 단순 중계 역할
app.get("/user/:id", async (req, res) => {
  const grpcResponse = await grpcClient.getUser(req.params.id);
  res.json(grpcResponse); // 그냥 전달
});

// 제안: BFF에서 직접 비즈니스 로직 처리
app.get("/user/:id", async (req, res) => {
  const user = await userRepository.findById(req.params.id);
  const profile = {
    ...user,
    displayName: user.nickname || user.name, // 이런 로직도 여기서!
    isVip: user.totalOrders > 100,
  };
  res.json(profile);
});
```

**BFF의 역할 확장**:

- ❌ 단순 중계 역할
- ✅ 실제 백엔드 서버 역할
- ✅ DB 직접 연결
- ✅ 프론트엔드 개발자가 독립적으로 API 개발

### 두 번째 제안: GraphQL 의존성 문제 해결

#### 방안 1: GraphQL 유지하되 도메인별 분리

기존 GraphQL 방식:

```typescript
// 모든 도메인이 하나의 리졸버에 섞여있음
@Query(() => GetAppVersionByPlatformResponseSchema)
async getAppVersionByPlatform(@Args() args: GetAppVersionByPlatformRequestSchema) {
    const { response } = await this.appVersionService.getAppVersionByPlatform(args);
    return response;
}

@Query(() => GetUserProfileResponseSchema)
async getUserProfile(@Args() args: GetUserProfileRequestSchema) {
    const { response } = await this.userService.getUserProfile(args);
    return response;
}
```

**문제점**: 여전히 Code Generation 의존성과 도메인 결합 문제가 남아있음 😰

#### 방안 2: RESTful API로 전환

```typescript
// 각 도메인별로 독립적인 REST API
/api/adimn / app - version / api / store / products / api / webview / config;
```

**문제점**: 타입 안전성을 잃어버림 😰

```typescript
// 이런 식으로 타입을 따로 정의해야...
interface AppVersionResponse {
  version: string;
  required: boolean;
  // 백엔드와 동기화하는데 어려움이 있음
}
```

#### 방안 3: tRPC 도입 🎯

"잠깐, tRPC라는 게 있던데... NestJS에서 사용할 수 있을까?"

레퍼런스를 찾아보니 **충분히 가능**하더라고요!

---

## 🏗️ tRPC 아키텍처 설계 및 도메인 분리

### 도메인별 완전 분리 전략

실제 우리 프로젝트 구조:

```
v2/modules/
├── solve-books/          # 스토어 서비스
│   ├── contents/         # 콘텐츠 관리
│   ├── partner/          # 파트너 관리
│   ├── solve/            # 문제 풀이
│   └── store/            # 스토어
│       ├── categories/
│       ├── exams/
│       ├── orders/
│       ├── payments/
│       └── products/
├── admin/               # 관리자 서비스
├── webview-v2/          # 웹뷰 서비스
└── trpc/               # tRPC 설정
```

### 핵심 설계 원칙

#### 1. **DB별 도메인 분리**

- `contents` → 콘텐츠 DB
- `partner` → 파트너 DB
- `solve` → 문제 풀이 DB
- `store` → 스토어 DB

#### 2. **테이블별 세부 모듈 분리**

```typescript
// store 도메인 하위 모듈들
store/
├── categories/           # 카테고리 테이블
├── exams/               # 시험 테이블
├── orders/              # 주문 테이블
├── payments/            # 결제 테이블
└── products/            # 상품 테이블
```

#### 3. **완전한 의존성 격리**

```typescript
// 각 도메인은 독립적인 라우터를 가짐
export const storeRouter = router({
  categories: categoriesRouter,
  exams: examsRouter,
  orders: ordersRouter,
  payments: paymentsRouter,
  products: productsRouter,
});

// 타입 참조도 자신의 도메인에서만!
import { Product } from "./products/product.schema";
// ❌ import { User } from '../solve/user.schema'; // 다른 도메인 참조 금지
```

---

## 🔧 실제 구현 과정

### 1. NestJS 기반 백엔드 플랫폼 구축

#### 기존 GraphQL 방식

```typescript
// GraphQL 리졸버 - 복잡한 데코레이터와 스키마 정의
@Query(() => GetAppVersionByPlatformResponseSchema)
async getAppVersionByPlatform(@Args() args: GetAppVersionByPlatformRequestSchema) {
    const { response } = await this.appVersionService.getAppVersionByPlatform(args);
    return response;
}

@Query(() => GetUserProfileResponseSchema)
async getUserProfile(@Args() args: GetUserProfileRequestSchema) {
    const { response } = await this.userService.getUserProfile(args);
    return response;
}

// 별도의 스키마 파일 필요
export const GetUserProfileResponseSchema = ObjectType(() => ({
  id: () => String,
  name: () => String,
  profile: () => UserProfileSchema,
}));
```

#### 새로운 tRPC 방식

```typescript
// user.schema.ts - Zod로 간단하게!
export const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  profile: UserProfileSchema,
});

// user.router.ts
export const userRouter = router({
  getProfile: procedure.input(z.string()).query(async ({ input: userId }) => {
    // 직접 DB 접근
    const user = await userRepository.findById(userId);
    return {
      ...user,
      displayName: user.nickname || user.name, // 프론트에서 원하는 로직!
    };
  }),
});
```

### 2. 클라이언트에서의 타입 안전성

#### 기존 GraphQL Code Generation

```typescript
// 1. 스키마 변경
// 2. Code Generation 실행
$ npm run codegen

// 3. 모든 프로젝트에서 타입 체크
$ npm run type-check
// 💥 Error in 15 files...

// 4. 하나씩 수정하고 다시...
$ npm run codegen
$ npm run type-check

// 5. 실제 사용 코드
const { data, loading, error } = useGetUserProfileQuery({
  variables: { userId: "123" },
  fetchPolicy: 'cache-and-network'
});

// 복잡한 타입 체인
const userName = data?.getUserProfile?.profile?.basicInfo?.name;
```

#### 새로운 tRPC 방식

```typescript
// 백엔드 스키마 변경하면 즉시 타입 동기화!
const { data } = trpc.user.getProfile.useQuery({ userId: "123" });
//     ^? User 타입이 자동으로 추론됨 ✨

// 백엔드에서 필드 추가하면 여기서도 바로 사용 가능!
const userName = data?.displayName; // 타입 안전하고 간단!
```

### 3. TanStack Query와의 조합

```typescript
// 기존 Apollo Client
const { data, loading, error, refetch } = useQuery(GET_PRODUCTS, {
  variables: { categoryId },
  fetchPolicy: "cache-and-network",
  // 복잡한 캐시 설정...
});

// 새로운 TanStack Query + tRPC
const { data, isLoading, error, refetch } = trpc.store.products.list.useQuery(
  { categoryId },
  {
    staleTime: 5 * 60 * 1000, // 5분간 신선
    enabled: !!categoryId, // 조건부 실행
  }
);
```

---

## 📊 마이그레이션 성과

### 정량적 개선 결과

| 지표                     | 이전      | 이후       | 개선율        |
| ------------------------ | --------- | ---------- | ------------- |
| **API 개발 시간**        | 평균 2일  | 평균 4시간 | **75% 단축**  |
| **빌드 에러**            | 주 5-10회 | 주 0-1회   | **90% 감소**  |
| **Cross-team 소통**      | 일 3-5회  | 일 0-1회   | **80% 감소**  |
| **Code Generation 시간** | 30초      | 0초        | **100% 제거** |

### 개발 워크플로우 개선

#### Before: 복잡한 프로세스 😰

```
1. 요구사항 → GraphQL 스키마 수정 논의
2. 스키마 변경 → Code Generation 실행
3. 모든 프로젝트 빌드 테스트 → 타입 에러 수정
4. 에러 발생 → 다시 1번부터...
⏱️ 평균 2-3일 소요
```

#### After: 독립적인 개발 🎉

```
1. 요구사항 → 바로 NestJS에서 구현
2. tRPC 스키마 정의 → 타입 자동 동기화
3. 프론트엔드에서 바로 사용
⏱️ 평균 2-4시간 소요
```

---

## 🛠️ 실제 사용 패턴 & 베스트 프랙티스

### 1. 도메인별 인증 시스템

```typescript
// 각 도메인별로 독립적인 인증 로직
export const storeRouter = router({
  // 인증 필요한 API
  getUserOrders: protectedProcedure.query(async ({ ctx }) => {
    return orderRepository.findByUserId(ctx.userId);
  }),

  // 공개 API
  getProducts: publicProcedure
    .input(GetProductsSchema)
    .query(async ({ input }) => {
      return productRepository.findMany(input);
    }),
});
```

### 2. 효율적인 캐시 전략

```typescript
// 도메인별 캐시 정책
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 기본 5분
    },
  },
});

// 상품 목록은 더 오래 캐시
const { data: products } = trpc.store.products.list.useQuery(
  { categoryId },
  { staleTime: 30 * 60 * 1000 } // 30분
);

// 주문 내역은 실시간성 중요
const { data: orders } = trpc.store.orders.list.useQuery(
  { userId },
  { staleTime: 0 } // 항상 최신
);
```

### 3. 에러 처리 & 재시도

```typescript
// 도메인별 에러 처리 전략
const { mutate: createOrder } = trpc.store.orders.create.useMutation({
  onError: (error) => {
    if (error.code === "INSUFFICIENT_STOCK") {
      toast.error("재고가 부족합니다");
    } else if (error.code === "PAYMENT_FAILED") {
      toast.error("결제에 실패했습니다");
    }
  },
  onSuccess: () => {
    // 관련 쿼리 무효화
    queryClient.invalidateQueries(["store", "orders"]);
    queryClient.invalidateQueries(["store", "products"]);
  },
});
```

---

## 💭 팀의 변화와 반응

### 개발자들의 실제 피드백

> 💬 **프론트엔드 개발자**: "이제 백엔드팀 눈치 안 보고 API 만들 수 있어서 너무 좋아요!"

> 💬 **백엔드 개발자**: "Go 서버는 정말 성능이 중요한 것만 담당하고, 나머지는 NestJS에서 해결하니까 훨씬 편해졌어요."

> 💬 **팀 리드**: "개발자 간 작업 충돌이 거의 사라졌고, 각 도메인 담당자가 자율적으로 개발할 수 있게 되었습니다."

### 학습 곡선과 적응 과정

**주차별 적응도**:

- **1주차**: tRPC 기본 개념 학습
- **2주차**: NestJS와 tRPC 통합 마스터
- **3주차**: 도메인 분리 전략 이해
- **4주차**: 실전 투입 가능

---

## 🚧 마이그레이션 과정의 도전과 해결

### 1. 기존 Go 서버와의 호환성

**문제**: 성능이 중요한 API는 여전히 Go 서버를 사용해야 함

**해결**: 하이브리드 아키텍처 구축

```typescript
// 성능 중요: Go 서버 유지
app.get("/api/heavy-calculation", async (req, res) => {
  const result = await grpcClient.heavyCalculation(req.body);
  res.json(result);
});

// 일반 비즈니스 로직: NestJS + tRPC
export const businessRouter = router({
  processOrder: procedure.input(OrderSchema).mutation(async ({ input }) => {
    // 직접 DB 처리
    return await orderService.process(input);
  }),
});
```

### 2. 데이터베이스 마이그레이션

**문제**: 기존 Go 서버의 DB 스키마와 호환 필요

**해결**: 점진적 마이그레이션

```typescript
// 1단계: 읽기 전용 API부터 시작
export const readOnlyRouter = router({
  getUser: procedure.query(async ({ input }) => {
    // 기존 DB 스키마 그대로 사용
    return await legacyUserRepository.findById(input.id);
  }),
});

// 2단계: 새로운 기능은 새로운 스키마로
export const newFeatureRouter = router({
  createUserProfile: procedure.mutation(async ({ input }) => {
    // 새로운 최적화된 스키마 사용
    return await newUserRepository.create(input);
  }),
});
```

---

## 🔮 향후 개선 계획

### 1. 성능 최적화 (진행중)

- [ ] tRPC 배치 요청 도입
- [ ] Redis 캐시 레이어 추가
- [ ] DB 쿼리 최적화

### 2. 모니터링 & 관측성

- [ ] 도메인별 API 사용량 대시보드
- [ ] 에러 추적 시스템 구축
- [ ] 성능 메트릭 수집

### 3. 개발자 경험 개선

- [ ] API 문서 자동 생성
- [ ] 개발 환경 Hot Reload 최적화
- [ ] 타입 검증 도구 개발

---

## 📝 아키텍처 개선을 고려한다면?

### ✅ 이런 상황에 추천

- **복잡한 마이크로서비스**로 인한 개발 병목 현상
- **GraphQL Code Generation** 의존성 문제
- **도메인 간 과도한 결합**으로 인한 협업 이슈
- **프론트엔드 개발자의 백엔드 의존성** 해소 필요

### ⚠️ 신중하게 고려해야 할 경우

- **초고성능이 필요한 모든 API**
- **레거시 시스템과의 깊은 통합** 필요
- **팀의 TypeScript 역량**이 부족한 경우

---

## 🎯 결론: 복잡함에서 단순함으로

### 핵심 성과 요약

1. **개발 생산성**: API 개발 시간 75% 단축
2. **협업 효율성**: Cross-team 소통 80% 감소
3. **시스템 안정성**: 빌드 에러 90% 감소
4. **개발자 만족도**: 자율성과 독립성 대폭 향상

### 가장 중요한 깨달음

> "복잡한 아키텍처가 항상 좋은 것은 아니다. **팀의 생산성과 행복**이 가장 중요한 지표다."

기존의 **"완벽한" 마이크로서비스 아키텍처**보다, **팀이 효율적으로 일할 수 있는 구조**가 훨씬 가치 있다는 것을 깨달았습니다.

### 다른 팀에게 전하고 싶은 메시지

아키텍처는 **목적을 위한 수단**이지, 그 자체가 목적이 되어서는 안 됩니다. 우리 팀에게는 tRPC + NestJS 조합이 최선의 선택이었지만, 여러분의 팀에게는 다른 해답이 있을 수 있습니다.

중요한 것은 **현재 팀이 겪고 있는 구체적인 문제를 정확히 파악**하고, 그에 맞는 **실용적인 해결책**을 찾는 것입니다.

---

## 📚 참고 자료

- [tRPC 공식 문서](https://trpc.io/)
- [NestJS tRPC 통합 가이드](https://docs.nestjs.com/recipes/trpc)
- [TanStack Query 문서](https://tanstack.com/query)
- [마이크로서비스 아키텍처 패턴](https://microservices.io/patterns/)

---

**💻 기술 스택**: TypeScript, tRPC, NestJS, React, Next.js, TanStack Query, Go, gRPC, Zod  
**📅 마이그레이션 기간**: 2025년 4월 - 6월 (3개월)  
**🎯 핵심 성과**: 개발 생산성 75% 향상, 빌드 에러 90% 감소, 협업 효율성 대폭 개선  
**👥 팀 구성**: 풀스택 개발자 6명, 백엔드 개발자 3명

> 🌟 **마지막 한마디**  
> "완벽한 아키텍처보다 팀이 행복하게 일할 수 있는 아키텍처가 진짜 좋은 아키텍처입니다!"
