# NextJS App router로 migration

## Next.js 14: Page 디렉토리에서 App Router로 마이그레이션 하는 이유

### 1. 최신 기능 활용

Next.js 14의 App Router는 최신 기능을 제공함. 주요 기능:

- **중첩된 라우트(Nested Routes):** 직관적이고 구조화된 URL 패턴 설정 가능.
- **레이아웃(Layouts):** 페이지 레벨에서 레이아웃 공유 가능, 공통 레이아웃 관리 쉬움.
- **서버 컴포넌트(Server Components):** 서버 사이드와 클라이언트 사이드 렌더링 혼합 용이.
- **데이터 패칭(Data Fetching):** 유연하고 강력한 데이터 패칭 전략 사용 가능.

### 2. 성능 향상

App Router는 성능 최적화 기능이 추가되어 더 빠른 페이지 로딩 속도 제공함. 자동 코드 분할과 정적 사이트 생성 기능 강화됨.
또한, 구직자의 경우 server side에서 api 호출하고 캐싱하는 경우가 많아 구직자만 App router 따로 적용.

### 3. 코드 유지보수성 향상

App Router는 코드 모듈화를 촉진하여 유지보수성 향상시킴. 대규모 애플리케이션의 코드 관리 용이함.

- **폴더 구조 일관성:** 명확하고 일관된 폴더 구조로 파일 탐색 용이.
- **중복 코드 감소:** 공통 레이아웃과 컴포넌트 공유 쉬워 중복 코드 줄일 수 있음.

### 4. 커뮤니티와 일치

Next.js는 빠르게 진화하는 프레임워크로, 커뮤니티와 일치시키기 위해 최신 기능 도입하고 사용해야 함. App Router로의 마이그레이션은 커뮤니티의 모범 사례 따르는 것이며, 장기적으로 더 많은 지원과 리소스 활용 가능.

### **결론**

Page 디렉토리에서 App Router로의 마이그레이션은 성능 향상, 유지보수성 증대, 개발자 경험 개선, 최신 기능 활용, 그리고 커뮤니티와 일치를 위해 필수적인 과정임. 이러한 이유로 Next.js 14버전의 App Router로의 마이그레이션 추진함.

이 문서를 기반으로 마이그레이션 이유 명확히 전달 가능함. 회사의 특정 요구사항이나 추가 이유 포함하여 문서 완성 바람.

## 현재 이슈

1. static, SSR, ISR 등 Server Side API호출 및 상태관리에 어려움이 있음.
2. React Server Component에서 CSS-IN-JS를 사용하지 못함.
3. page directory에서 쓰던 useRouter를 못쓰게 됨. 그에 따라 query를 다루기 어려워졌음
4. Router.event에서 지원하던 몇몇 event들이 없어짐.
5. 공식문서에서 지향하는 폴더구조가 명확하지 않아, 폴더구조를 명확하게 해야함.
6. Page Directory에서 사용하던 WithHead 컴포넌트를 더 이상 사용하지 못함.

## 해결 방안

### 1. static, SSR, ISR 등 Server Side API호출 및 상태관리에 어려움이 있음.

기존 SSR을 위해 사용하던 server side Redux가 제거 됐음. 그로인해 redux는 client side에서만 활용해야 함.
기존 SSR을 위한 데이터 패칭과 상태관리는 아래와 같다.

```tsx
export const getServerSideProps = wrapper.getServerSideProps(
  (store) => async () => {
    try {
      const adminsManage = await getAdminsAPI();
      store.dispatch(adminManageActions.setAdmins(adminsManage.data));
    } catch (e) {
      sendErrorToSentry(e);
    }
    return {
      props: {},
    };
  }
);
```

서버사이드에서 redux가 제거되었기에 swr로 대체하고 최상단에서 fetch로 호출 후 SWRConfig fallback으로 전달하고 페이지 하위 컴포넌트들에서 useSWR로 데이터 상태를 관리하는 방식으로 대체한다.

```tsx
import {
  fetchRecruitmentAPI,
  getRecruitmentApiKey,
} from "#lib/api/recruitment";
import { SWRProvider } from "#providers/SWRProvider";
import { NextAppRouterPage, NextPageContext } from "#types/type";
import { NextPage } from "next";
import RecruitmentDetail from "./_components/RecruitmentDetail";
import { sendErrorToSentry } from "@lib/error";
import { notFound } from "next/navigation";

const fetchRecruitmentDetail = async (id: string) => {
  try {
    const res = await fetchRecruitmentAPI(id, {
      next: {
        revalidate: 600,
        tags: ["recruitment", id],
      },
    });
    return res;
  } catch (error) {
    sendErrorToSentry(error);
  }
};

const page: NextAppRouterPage = async ({ params: { id } }) => {
  const res = await fetchRecruitmentDetail(id);

  if (!res) {
    notFound();
  }

  return (
    <SWRProvider
      value={{
        // swr, fetch, axios에서 api key값을 공유하기 위해 getKey 함수 활용
        fallback: {
          [getRecruitmentApiKey(id)]: fetchRecruitmentDetail(id),
        },
      }}
    >
      <RecruitmentDetail />
    </SWRProvider>
  );
};

export default page;
```

## 2. React Server Component에서 CSS-IN-JS를 사용하지 못함

우리 서비스는 현재 CSS-IN-JS인 styled-components 라이브러리를 활용해 스타일링을 하고 있다. 하지만 React Server Component는 CSS-IN-JS를 지원하지 않아 다음과 같은 방식으로 대체한다.

### 1. layout에 다음 Provider를 적용

```tsx
"use client";

import React, { useState } from "react";
import { useServerInsertedHTML } from "next/navigation";
import { ServerStyleSheet, StyleSheetManager } from "styled-components";

export default function StyledComponentsRegistry({
  children,
}: {
  children: React.ReactNode;
}) {
  // Only create stylesheet once with lazy initial state
  // x-ref: https://reactjs.org/docs/hooks-reference.html#lazy-initial-state
  const [styledComponentsStyleSheet] = useState(() => new ServerStyleSheet());

  useServerInsertedHTML(() => {
    const styles = styledComponentsStyleSheet.getStyleElement();
    styledComponentsStyleSheet.instance.clearTag();
    return <>{styles}</>;
  });

  if (typeof window !== "undefined") return <>{children}</>;

  return (
    <StyleSheetManager sheet={styledComponentsStyleSheet.instance}>
      {children as React.ReactChild}
    </StyleSheetManager>
  );
}
```

### 2. Styled Component 분리

```tsx
"use client";
import styled from "styled-components";

const SampleStyeld = styled.div`
  width: 936px;
  padding: 30px 0;
  margin: auto;
  border-radius: 5px;
  @media (max-width: 768px) {
    width: 100%;
    padding: 20px;
  }
`;

export default SampleStyeld;
```

### 3. RSC 제작

```tsx
import { fetchBannersAPI } from "#lib/api/banner";
import SampleStyeld from "./SampleStyled";

const getBanner = async () => {
  try {
    const banners = await fetchBannersAPI(
      { exposed: "project" },
      {
        next: {
          revalidate: 5,
        },
      }
    );
    return banners || [];
  } catch (error) {
    console.log(error);
  }
  return [];
};

const Sample: React.FC = async () => {
  const banner = await getBanner();
  return (
    <SampleStyeld>
      {banner.map((item) => (
        <li key={item.id}>{item.order}</li>
      ))}
    </SampleStyeld>
  );
};

export default Sample;
```

## 3. page directory에서 쓰던 useRouter를 못쓰게 됨. 그에 따라 query를 다루기 어려워졌음

어렵더라도 공식문서 원칙대로 진행 (useSearchParams 활용)
기존 query는 아래와 같이 처리

```tsx
const query = Object.fromEntries(searchParams!.entries());
```

**다만 useEffect 종속성 query는 searchParams로 대체해야함**

## 4. Router.event에서 지원하던 몇몇 event들이 없어짐.

기존 Router를 오버라이딩하여 기존에 사용하던 기능 구현

```tsx
"use client";

import { useCallback, useEffect, useRef } from "react";

import _ from "lodash";

import type { RouterEvent } from "#types/AppRouterContext";
import {
  AppRouterContext,
  AppRouterInstance,
} from "next/dist/shared/lib/app-router-context.shared-runtime";

const coreMethodFields = [
  "push",
  "replace",
  "refresh",
  "back",
  "forward",
] as const;

const routerEvents: Record<
  RouterEvent,
  Array<(url: string, options?: any) => void>
> = {
  routeChangeStart: [],
  routeChangeComplete: [],
};

/**
 * @see https://velog.io/@khxxjxx/NextJS-14-router.events
 */
const AppRouterConsumer: React.FC<React.PropsWithChildren> = ({ children }) => {
  const index = useRef(0);
  const routeEventInfo = useRef<{
    direction: "back" | "forward";
    url: string;
  }>();

  const events: AppRouterInstance["events"] = {
    on(event: RouterEvent, cb: (url: string, options?: any) => void) {
      routerEvents[event] = [...routerEvents[event], cb];
    },
    off(event: RouterEvent, cb: (url: string, options?: any) => void) {
      routerEvents[event] = _.without(routerEvents[event], cb);
    },
  };

  function proxy(
    router: AppRouterInstance,
    field: (typeof coreMethodFields)[number]
  ) {
    const method = router[field];

    Object.defineProperty(router, field, {
      get: () => {
        return async (url: string, options?: any) => {
          try {
            if (!_.isEmpty(routerEvents.routeChangeStart)) {
              const promiseList = routerEvents.routeChangeStart.map((cb) =>
                cb(url, options)
              );
              await Promise.all(promiseList);
            }
            method(url, options);
          } catch (e) {
            console.error(e);
          }
        };
      },
    });
  }

  const eventListenerHandler = useCallback(
    (listener: EventListenerOrEventListenerObject) => async (event: Event) => {
      const eventListener =
        "handleEvent" in listener ? listener.handleEvent : listener;

      if (event.type === "popstate") {
        if (!_.isEmpty(routerEvents.routeChangeStart)) {
          const historyIndex = window.history.state?.index ?? 0;
          const routeInfo = routeEventInfo.current;

          if (routeInfo && historyIndex === index.current) {
            try {
              const { url, direction } = routeInfo;

              const promiseList = routerEvents.routeChangeStart.map((cb) =>
                cb(url, event)
              );
              await Promise.all(promiseList);

              return window.history[direction]();
            } catch (e) {
              routeEventInfo.current = undefined;
              return console.error(e);
            }
          } else if (routeInfo) {
            routeEventInfo.current = undefined;
          } else {
            const backEvent = index.current > historyIndex;
            const forwardEvent = index.current < historyIndex;

            const pathname = window.location.pathname;
            const query = window.location.search;
            const url = pathname + query;

            routeEventInfo.current = {
              direction: backEvent ? "back" : "forward",
              url,
            };
            if (backEvent) window.history.forward();
            else if (forwardEvent) window.history.back();

            return;
          }
        }
      }

      eventListener(event);
    },
    []
  );

  useEffect(() => {
    const originAddEventListener = window.addEventListener;
    const originRemoveEventListener = window.removeEventListener;

    const eventListeners = new Map();

    const addListener = function <K extends keyof WindowEventMap>(
      type: K,
      listener: EventListenerOrEventListenerObject,
      options?: boolean | AddEventListenerOptions
    ) {
      const wrappedListener = eventListenerHandler(listener);
      eventListeners.set(listener, wrappedListener);
      originAddEventListener(type, wrappedListener, options);
    };

    const removeListener = function <K extends keyof WindowEventMap>(
      type: K,
      listener: EventListenerOrEventListenerObject,
      options?: boolean | AddEventListenerOptions
    ) {
      const wrappedListener = eventListeners.get(listener);
      if (wrappedListener) {
        originRemoveEventListener(type, wrappedListener, options);
        eventListeners.delete(listener);
      }
    };

    window.addEventListener = addListener;
    window.removeEventListener = removeListener;

    return () => {
      window.addEventListener = originAddEventListener;
      window.removeEventListener = originRemoveEventListener;
    };
  }, [eventListenerHandler]);

  useEffect(() => {
    const originalPushState = window.history.pushState;
    const originalReplaceState = window.history.replaceState;

    index.current = window.history.state?.index ?? 0;

    window.history.pushState = (
      data: any,
      _: string,
      url?: string | URL | null
    ) => {
      const historyIndex = window.history.state?.index ?? 0;
      const nextIndex = historyIndex + 1;
      const state = { ...data, index: nextIndex };

      index.current = nextIndex;

      return History.prototype.pushState.apply(window.history, [state, _, url]);
    };
    window.history.replaceState = (
      data: any,
      _: string,
      url?: string | URL | null
    ) => {
      const historyIndex = window.history.state?.index ?? 0;
      const state = { ...data, index: historyIndex };

      index.current = historyIndex;

      return History.prototype.replaceState.apply(window.history, [
        state,
        _,
        url,
      ]);
    };

    return () => {
      window.history.pushState = originalPushState;
      window.history.replaceState = originalReplaceState;
    };
  }, []);

  return (
    <AppRouterContext.Consumer>
      {(router) => {
        if (router) {
          router.events = events;
          coreMethodFields.forEach((field) => proxy(router, field));
        }
        return children;
      }}
    </AppRouterContext.Consumer>
  );
};

export default AppRouterConsumer;
```

## 5. 공식문서에서 지향하는 폴더구조가 명확하지 않아, 폴더구조를 명확하게 해야함.

1. 디랙토리 구조 그대로 가져오기 ex) components ⇒ \_ components
2. 페이지에 종속하는 컴포넌트 관리를 위해 다음과 같은 폴더 구조를 따름
3. 공용컴포넌트는 상위 폴더로 이동
4. 각 컴포넌트가 독립적으로 사용 될 수 있도록 컴포넌트 구조를 아래와 같이 지향

컴포넌트 구조는 아래와 같다.

```
├── apps
│   ├── _components
│   ├── (pages)
│   │    ├── (list)
│   │    │     ├── _components
│   │    │     └── _hooks
│   │    └── [id]
│   │    │     ├── _components
│   │    │     └── _hooks
```

## 5. Page Directory에서 사용하던 WithHead 컴포넌트를 더 이상 사용하지 못함.

기존 h 태그를 분리하여 각 페이지에 다음과 같이 추가한다.

```tsx
<h2 className="hidden-title">{title}</h2>
```
