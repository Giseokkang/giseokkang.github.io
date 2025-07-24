---
layout: single
title: "GraphQL에서 tRPC로: 타입 안전성과 개발 생산성을 동시에 잡은 마이그레이션 이야기"
date: 2025-07-25
categories: [백엔드, 프론트엔드]
tags: [tRPC, GraphQL, TypeScript, React, NestJS, API, 마이그레이션]
---

# GraphQL에서 tRPC로: 타입 안전성과 개발 생산성을 동시에 잡은 마이그레이션 이야기

> 🚀 **마이그레이션 요약**  
> GraphQL의 복잡성과 개발 피로도 문제를 해결하기 위해 tRPC로 전환하여 개발 생산성과 타입 안전성을 동시에 향상시킨 경험을 공유합니다.

---

## 🤔 왜 GraphQL을 버리고 tRPC를 선택했을까?

### GraphQL, 처음엔 좋았는데...

GraphQL을 도입했을 때만 해도 "이거다!"라는 생각이었습니다. 클라이언트가 필요한 데이터만 요청할 수 있고, 단일 엔드포인트로 모든 것을 해결할 수 있다는 점이 매력적이었죠.

하지만 프로젝트가 커지고 팀이 성장하면서 여러 문제점들이 드러나기 시작했습니다.

### 🚨 GraphQL의 현실적인 한계들

#### 1. **도메인 간 의존성 지옥**

```graphql
# 모든 도메인이 하나의 스키마에 얽혀있어서...
type User {
  id: ID!
  profile: UserProfile
  orders: [Order!]! # 주문 도메인에 의존
  payments: [Payment!]! # 결제 도메인에 의존
  reviews: [Review!]! # 리뷰 도메인에 의존
}
```

사용자 도메인을 수정하려면 주문, 결제, 리뷰 팀과 모두 조율해야 했습니다. 😵

#### 2. **Code Generation 피로도**

```bash
# 매번 반복되는 의식(?)
$ npm run codegen
$ npm run build
# 😱 타입 에러 100개 발생
$ npm run codegen
$ npm run build
# 🎉 겨우 성공
```

스키마가 변경될 때마다 코드 생성 → 빌드 → 에러 수정의 무한 반복이었습니다.

#### 3. **복잡한 데이터 구조**

```typescript
// GraphQL 응답은 항상 중첩이 깊어서...
const userData = data?.getUser?.profile?.basicInfo?.name;
// undefined가 될 수 있는 체인이 너무 길어요 😱
```

#### 4. **Apollo Client 캐시의 복잡성**

```typescript
// 캐시 업데이트가 이렇게 복잡할 일인가요?
const [updateUser] = useUpdateUserMutation({
  update(cache, { data }) {
    cache.writeQuery({
      query: GET_USER_QUERY,
      data: {
        getUser: {
          ...existingUser,
          ...data.updateUser,
        },
      },
    });
  },
});
```

---

## 💡 tRPC라는 새로운 희망

### "Code Generation이 필요 없다고?!"

tRPC를 처음 알게 된 건 동료 개발자의 추천이었습니다. "GraphQL처럼 타입 안전하면서도 REST API처럼 간단하다"는 말에 호기심이 생겼죠.

```typescript
// tRPC는 이렇게 간단합니다
const { data } = trpc.user.getProfile.useQuery();
// 타입이 자동으로 추론되고, 코드 생성은 필요 없어요! 🎉
```

### 🎯 tRPC 도입 목표

1. **도메인별 API 분리**: 독립적인 개발과 배포 가능
2. **개발 생산성 향상**: Code Generation 제거
3. **타입 안전성 유지**: TypeScript의 강력한 타입 시스템 활용
4. **캐시 관리 단순화**: TanStack Query의 직관적인 캐시

---

## 🏗️ tRPC 아키텍처 설계

### 백엔드: 도메인 중심 설계

```
apps/apigateway/src/v2/
├── trpc/                    # tRPC 핵심 설정
├── common/                  # 공통 서비스 (JWT, 유틸리티)
├── solve-books/            # 도메인별 모듈
│   ├── user/
│   │   ├── user.router.ts
│   │   ├── user.service.ts
│   │   └── user.schema.ts
│   └── exam/
└── admin/                  # 관리자 도메인
```

각 도메인이 독립적인 엔드포인트를 가지도록 설계했습니다:

- `localhost:5500/trpc/solve-books` - 사용자 API
- `localhost:5500/trpc/admin` - 관리자 API

### 프론트엔드: 간단하고 강력한 구조

```typescript
// utils/trpc.ts - 메인 설정
export const trpc = createTRPCReact<AppRouter>();

// 사용법이 이렇게 간단해졌어요!
const { data, isLoading } = trpc.user.getProfile.useQuery();
```

---

## 🔧 핵심 구현 포인트

### 1. 타입 안전한 스키마 정의

```typescript
// user.schema.ts
import { z } from "zod";

export const UserSchema = z.object({
  id: z.coerce.number(),
  email: z.string().email(),
  name: z.string(),
  createdAt: z.date(),
});

export const SignUpInputSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(2),
});

export type User = z.infer<typeof UserSchema>;
export type SignUpInput = z.infer<typeof SignUpInputSchema>;
```

### 2. 라우터와 프로시저

```typescript
// user.router.ts
@Injectable()
export class UserRouter {
  constructor(
    private readonly trpc: TrpcService,
    private readonly userService: UserService
  ) {}

  get router() {
    return this.trpc.router({
      // 인증이 필요한 API
      getProfile: this.trpc.protectedProcedure.query(async ({ ctx }) => {
        return this.userService.findById(ctx.userId);
      }),

      // 회원가입 (인증 불필요)
      signup: this.trpc.procedure
        .input(SignUpInputSchema)
        .mutation(async ({ input }) => {
          return this.userService.create(input);
        }),
    });
  }
}
```

### 3. 자동 인증 재시도 링크

인증 토큰 만료 시 자동으로 갱신하는 로직을 구현했습니다:

```typescript
// trpc-auth-link.ts
export function authRetryLink<TRouter extends AnyRouter>(): TRPCLink<TRouter> {
  return () => {
    return ({ next, op }) => {
      return observable((observer) => {
        const executeRequest = async (retryCount = 0) => {
          subscription = next(op).subscribe({
            error: async (err) => {
              // 401 에러 시 토큰 갱신 후 재시도
              if (err.data?.httpStatus === 401 && retryCount === 0) {
                const newToken = await refreshAccessToken();
                if (newToken) {
                  setTimeout(() => executeRequest(1), 100);
                  return;
                }
              }
              observer.error(err);
            },
          });
        };
        executeRequest();
      });
    };
  };
}
```

---

## 📊 마이그레이션 결과

### Before vs After 비교

#### GraphQL 시절 😰

```typescript
// 1. 스키마 변경
// 2. Code Generation 실행
$ npm run codegen
// 3. 타입 에러 수정
// 4. 빌드 다시 시도
$ npm run build
// 5. 또 에러... 반복

const { data, loading, error } = useFindUserQuery({
  fetchPolicy: 'no-cache',
  variables: { userId: 123 },
});

const userName = data?.findUser?.profile?.basicInfo?.name;
```

#### tRPC 도입 후 🎉

```typescript
// 스키마 변경하면 타입이 바로 동기화!
const { data, isLoading } = trpc.user.getProfile.useQuery({ userId: 123 });

const userName = data?.name; // 깔끔하고 타입 안전!
```

### 🚀 정량적 개선 결과

| 항목                | GraphQL      | tRPC         | 개선율             |
| ------------------- | ------------ | ------------ | ------------------ |
| **빌드 시간**       | 평균 45초    | 평균 15초    | **67% 단축**       |
| **Code Generation** | 필수         | 불필요       | **100% 제거**      |
| **타입 에러**       | 빌드 시 발견 | 실시간 감지  | **개발 경험 향상** |
| **API 응답 depth**  | 평균 3-4단계 | 평균 1-2단계 | **단순화**         |

---

## 🛠️ 실제 사용 패턴

### 1. 사용자 관리 훅

```typescript
// hooks/useUser.ts
export const useUser = () => {
  const trpc = useTRPCClient();
  const queryClient = useQueryClient();

  const { data: user, isLoading } = trpc.user.getProfile.useQuery(undefined, {
    enabled: !!getAccessToken(),
  });

  const { mutateAsync: logout } = trpc.user.logout.useMutation({
    onSuccess: () => {
      removeAccessToken();
      queryClient.clear(); // 모든 캐시 제거
    },
  });

  return { user, isLoading, logout };
};
```

### 2. 조건부 쿼리 실행

```typescript
// 토큰이 있을 때만 사용자 정보 조회
const { data } = trpc.user.getProfile.useQuery(undefined, {
  enabled: !!getAccessToken(),
});

// 카테고리가 선택된 경우만 시험 목록 조회
const { data: exams } = trpc.exam.list.useQuery(
  { categoryId },
  { enabled: !!categoryId }
);
```

### 3. 낙관적 업데이트

```typescript
const { mutate: updateProfile } = trpc.user.updateProfile.useMutation({
  onMutate: async (newData) => {
    // 낙관적 업데이트
    const previousData = queryClient.getQueryData(
      trpc.user.getProfile.getQueryKey()
    );

    queryClient.setQueryData(trpc.user.getProfile.getQueryKey(), {
      ...previousData,
      ...newData,
    });

    return { previousData };
  },
  onError: (err, newData, context) => {
    // 에러 시 롤백
    queryClient.setQueryData(
      trpc.user.getProfile.getQueryKey(),
      context?.previousData
    );
  },
});
```

---

## 🚧 마이그레이션 과정의 도전과 해결

### 1. GraphQL → tRPC 타입 변환

**문제**: GraphQL의 `InputMaybe<T>` 타입이 `T | null | undefined`인데, tRPC는 `undefined`만 허용

```typescript
// 해결책: 변환 유틸리티 함수
const convertToTRPCInput = (graphqlInput) => {
  return Object.fromEntries(
    Object.entries(graphqlInput).map(([key, value]) => [
      key,
      value === null ? undefined : value,
    ])
  );
};
```

### 2. 기존 Apollo Client 코드와의 호환성

**문제**: 한 번에 모든 것을 마이그레이션할 수 없었음

**해결**: 점진적 마이그레이션 전략

```typescript
// 1단계: 새로운 기능만 tRPC로
// 2단계: 자주 사용되는 API부터 전환
// 3단계: 레거시 코드 완전 제거
```

### 3. SSR(Server-Side Rendering) 지원

```typescript
// utils/server-trpc.ts
export const helpers = createServerSideHelpers({
  client: trpcClient,
});

// getServerSideProps에서 사용
export async function getServerSideProps(ctx) {
  try {
    const user = await helpers.user.getProfile.fetch();
    return { props: { user } };
  } catch (error) {
    return { props: { user: null } };
  }
}
```

---

## 💭 팀의 반응과 학습 곡선

### 개발자들의 피드백

> 💬 **프론트엔드 개발자**: "Code Generation 안 해도 되니까 너무 좋아요! 타입도 바로바로 업데이트되고."

> 💬 **백엔드 개발자**: "도메인별로 분리되니까 다른 팀 눈치 안 보고 개발할 수 있어서 좋습니다."

> 💬 **QA 엔지니어**: "API 문서가 자동으로 생성되는 패널이 테스트할 때 정말 유용해요."

### 학습 곡선

- **GraphQL 경험자**: 1-2주면 완전 적응
- **REST API만 써본 개발자**: 2-3주 정도 필요
- **TypeScript 미숙자**: Zod 스키마 학습에 추가 시간 필요

---

## 🔮 향후 계획

### 1. 성능 최적화

- [ ] tRPC 배치 요청 최적화
- [ ] 캐시 전략 고도화
- [ ] CDN 레벨 캐싱 도입

### 2. 개발 경험 개선

- [ ] 자동 API 문서 생성
- [ ] 에러 추적 시스템 고도화
- [ ] 실시간 타입 검증 도구

### 3. 확장성 고려

- [ ] 마이크로서비스 아키텍처 대응
- [ ] 다국어 스키마 지원
- [ ] 버전 관리 시스템 도입

---

## 📝 tRPC 도입을 고려한다면?

### ✅ 이런 팀에게 추천

- **TypeScript를 적극 활용**하는 팀
- **풀스택 개발**이 빈번한 프로젝트
- **빠른 프로토타이핑**이 중요한 스타트업
- **타입 안전성**을 중시하는 팀

### ⚠️ 신중하게 고려해야 할 경우

- **다양한 클라이언트**(모바일 앱, 서드파티)를 지원해야 하는 경우
- **GraphQL 생태계**에 깊이 의존하고 있는 경우
- **대규모 레거시 시스템**과 통합해야 하는 경우

---

## 🎯 결론: 선택의 기준

tRPC는 **타입 안전성과 개발 생산성의 완벽한 균형**을 제공합니다. 하지만 만능 해결책은 아닙니다.

### 우리 팀에게 tRPC가 맞았던 이유:

1. **TypeScript 기반**의 풀스택 개발
2. **빠른 기능 개발**이 중요한 환경
3. **도메인별 독립성**이 필요한 구조
4. **개발자 경험** 개선에 대한 높은 우선순위

GraphQL에서 tRPC로의 전환은 단순한 기술 교체가 아니라 **개발 철학의 변화**였습니다. 복잡함을 줄이고 본질에 집중할 수 있게 해준 tRPC 덕분에, 우리 팀은 더 빠르고 안전하게 기능을 개발할 수 있게 되었습니다.

---

## 📚 참고 자료

- [tRPC 공식 문서](https://trpc.io/)
- [TanStack Query 문서](https://tanstack.com/query)
- [Zod 스키마 검증](https://zod.dev/)
- [Next.js tRPC 통합 가이드](https://trpc.io/docs/nextjs)

---

**💻 기술 스택**: TypeScript, tRPC, React, Next.js, NestJS, TanStack Query, Zod  
**📅 마이그레이션 기간**: 2024년 8월 - 12월 (4개월)  
**🎯 핵심 성과**: 개발 생산성 향상, 타입 안전성 강화, 유지보수성 개선  
**👥 팀 구성**: 풀스택 개발자 4명

> 🚀 **마지막 한마디**  
> 기술 선택에 정답은 없지만, 팀의 상황과 목표에 맞는 최선의 선택은 있습니다. tRPC는 우리 팀에게 그런 선택이었고, 여러분의 팀에게도 좋은 선택이 되기를 바랍니다!
