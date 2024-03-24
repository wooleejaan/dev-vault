# GetStarted - Next.js 서버 컴포넌트와 Tanstack Query의 “Hydrate”를 같이 사용해보기

### 서버 컴포넌트의 아쉬움 : Next js 13의 React Server Component

Next.js 13 app router를 사용하면, 더 이상 getServerSideProps, getStaticProps 등을 사용하지 않아도 된다(페이지 최상단에서 데이터를 가져와 prop drilling을 하거나 불필요하게 전역 상태로 관리하지 않아도 된다). 서버컴포넌트에서 그냥 데이터를 받아오고, revalidate 값을 조절하거나 fetch를 사용한다면 옵션 값을 통해 쉽게 ssr, ssg, isr 등을 구현할 수 있다([공식문서 참고](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating)).

심지어 서버컴포넌트에서 가져온 데이터를 클라이언트로 넘겨줄 때, Nextjs에서 제안하는 [Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns)을 사용한다면 prop drilling도 그렇게 큰 문제가 되지 않는 것처럼 보인다. 정말 그럴까?

app router 환경에서 개발을 하다보면, 생각보다 컴포넌트 구조가 복잡해지는 경우가 종종 있다. 서버 컴포넌트에서는 기존의 리액트 훅이나 브라우저 api 들을 사용하지 못하다 보니, 그냥 “use client”로 도배하고 싶을 때가 많다.

### 보완할 수 있는 방법 : Tanstack Query의 “Hydrate”

이럴 때, 보조로 혹은 메인으로 Tanstack Query의 [Hydrate](https://tanstack.com/query/v4/docs/react/guides/ssr#using-the-app-directory-in-nextjs-13)을 사용하면 쉽게 prop drilling을 피하면서도 서버 컴포넌트에서 받아온 데이터를 클라이언트에서 사용할 수 있다. 서버 컴포넌트를 쓰려고 컴포넌트 구조를 굳이 복잡하게 만드는 짓을 하지 않아도 된다.

공식문서에서 설명하는 Hydrate은 다음과 같은 과정을 거친다.

- QueryClient 인스턴스를 찾고, prefetchQuery 메서드로 데이터를 prefetch한다.
- 서버에서 쿼리를 prefetch한 캐시를 dehyrate한다.
- 그리고 나서 클라이언트로 rehydrate을 해준다.
- 클라이언트에서는 일반적인 방식으로 useQuery 같은 훅을 사용해 데이터를 요청하면, 서버에서 prefetcing한 캐싱 데이터를 찾아 가져온다.

### “Hydrate” 사용 방법

먼저, Composition Patterns을 사용해 provider를 만들어주고 최상단에 심어준다.

```js
// provider.tsx

"use client";
import { useState } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { queryClientOptions } from "@/api/tanstack/tanstack.helpers";
import { ITanstackProviderProps } from "./tanstack.types";
export default function TanstackProvider({ children }: ITanstackProviderProps) {
  const [queryClient] = useState(() => new QueryClient(queryClientOptions));
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

이렇게 만들어준 provider를 layout.tsx에 심어준다.

```js
// layout.tsx

import TanstackProvider from "@/api/tanstack/tanstackProvider.context";
import { META_ROOT } from "./_meta";
export const metadata = META_ROOT;
export default function RootLayout({
  children,
}: {
  children: React.ReactNode,
}) {
  return (
    <html lang="ko" className={gmarketSans.className}>
      <body>
        <TanstackProvider>
          <main>{children}</main>
        </TanstackProvider>
      </body>
    </html>
  );
}
```

그리고 나서 QueryClient의 싱글톤 인스턴스를 생성해준다.

이렇게 캐싱 처리를 해줘야, 서로 다른 요청 간 데이터가 공유되지 않는 QueryClient가 된다.

```js
import { cache } from "react";

import { QueryClient } from "@tanstack/react-query";
// caching query client
const getQueryClient = cache(() => new QueryClient(queryClientOptions));
export { getQueryClient };
```

이제 여기서 만든 getQueryClient의 prefetchQuery를 사용해 서버 컴포넌트에서 데이터를 불러온다.

```js
// server.tsx

import QueryHydrate from "@/api/tanstack/queryHydrate.context";
export default async function ServerComponent() {
  const queryClient = getQueryClient();
  await queryClient.prefetchQuery(["mykey"], () => getApi());
  const dehydratedState = dehydrate(queryClient);
  return (
    <QueryHydrate state={dehydratedState}>{/* 하위 컴포넌트 */}</QueryHydrate>
  );
}
```

prefetchQuery 메서드로 데이터를 가져온 queryClient를 dehydrate으로 감싸고 QueryHydrate의 state로 넘겨준다.

원래의 return하던 컴포넌트들을 QueryHydrate으로 감싸주면, 감싸진 영역 안에서 prefetch된 데이터를 가져올 수 있게 된다.

```js
// client.tsx

"use client";
export default function ClientComponent() {
  const { data, isLoading } = useQuery({
    queryKey: ["mykey"],
    queryFn: () => getApi(),
  });
}
```

그럼 위와 같이 QueryHydrate으로 감싸진 영역 하위에 있는 클라이언트 컴포넌트에서 똑같이 useQuery를 사용했을 때, 서버 컴포넌트에서 prefetch된 데이터를 prop drilling 없이 가져올 수 있게 된다.

참고로 QueryHydrate는 아래와 같이 커스텀한 것이며, 그냥 Hydrate을 가져와서 감싸도 상관없다.

```js
"user client";

import { HydrateProps, Hydrate as RQHydrate } from "@tanstack/react-query";
export default function QueryHydrate(props: HydrateProps) {
  return <RQHydrate {...props} />;
}
```

### 참고 문서

[Next.js App Router의 Composition Patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns)<br>
[Using the app Directory in Next.js 13 | Tanstack Query](https://tanstack.com/query/v4/docs/react/guides/ssr#using-the-app-directory-in-nextjs-13)<br>
[Next.js 13에서 React Query SSR 적용하는 방법](https://velog.io/@ckstn0777/Next.js-13%EC%97%90%EC%84%9C-React-Query-SSR-%EC%A0%81%EC%9A%A9%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95)<br>
[hydrate | Tanstack Query](https://tanstack.com/query/v4/docs/react/reference/hydration)<br>
[Next) 서버 컴포넌트(React Server Component)에 대한 고찰](https://velog.io/@2ast/React-%EC%84%9C%EB%B2%84-%EC%BB%B4%ED%8F%AC%EB%84%8C%ED%8A%B8React-Server-Component%EC%97%90-%EB%8C%80%ED%95%9C-%EA%B3%A0%EC%B0%B0)<br>
