이 글에서 설명하는 zustand은 4.4.3 버전이다. [공식문서](https://github.com/pmndrs/zustand/blob/main/docs/integrations/persisting-store-data.md)를 참고해 작성했다.

## Zustand

zustand는 액션 기반의 중앙화된 상태관리 라이브러리다. Redux에 비해 사용하기가 매우 쉽고 직관적이다.

로컬스토리지의 경우, key를 안전하게 관리를 하기 위해 Context api를 사용했었는데, 번거로운 점이 있었다. context api는 재렌더링 관리가 어려워 memo를 적극적으로 사용해야 하는데, 그게 꽤 번거로웠다. 이때 중앙화된 상태관리를 쓰고 싶은데, redux의 보일러플레이팅이 너무 과하다고 느낀다면 zustand도 좋은 선택지가 될 수 있다.

### Store 생성하기

zustand에는 immer, persist라는 내장 미들웨어가 존재한다.<br>
persist를 사용하면 외부 storage를 zustand store로 관리할 수 있다.

또한 immer를 사용하면 객체를 불변성 있게 상태로 아주 쉽게 관리할 수 있다.<br>
하나의 컴포넌트에 여러 개의 Store 프로퍼티들을 가져오다 보면 불필요하게 렌더링이 되는 경우가 있다. 그럴 때 immer를 사용한다.

- immer의 경우 `yarn add immer`을 통해 패키지를 별도로 설치해줘야 한다.

```js
// .../store/localStorage/store.ts

import { create } from "zustand";
import { createJSONStorage, persist } from "zustand/middleware";
import { immer } from "zustand/middleware/immer";
import { PersistTypes, ProductDetailProps } from "./types";
const STORAGE_KEY = "curr-prod";
const useLocalStore = create(
  immer(
    persist <
      PersistTypes >
      ((set) => ({
        product: {},
        actions: {
          setProduct: (product: ProductDetailProps) => {
            set({ product });
          },
          clearProduct: () => {
            set({ product: null });
          },
        },
      }),
      {
        name: STORAGE_KEY,
        storage: createJSONStorage(() => localStorage),
      })
  )
);
export default useLocalStore;
```

내장 미들웨어의 경우 store를 생성할 때 중첩해서 사용할 수 있다.<br>
storage의 경우 createJSONStorage를 사용하면 쉽게 사용할 수 있다.<br>
로컬스토리지 뿐 아니라 세션스토리지, IndexedDB 등 다양한 스토리지를 핸들링할 수 있다.<br>
웹 스토리지의 경우 클라이언트에서 핸들링해야 하기 때문에 hydrate 과정을 포함해 다양한 훅들도 존재한다. ([공식문서](https://github.com/pmndrs/zustand/blob/main/docs/integrations/persisting-store-data.md))

zustand가 중앙 형태의 상태관리 아키텍쳐를 지향하지만, 하나의 store 안에서 부분적으로만 사용하고 싶다면 [partialize](https://github.com/pmndrs/zustand/blob/main/docs/integrations/persisting-store-data.md#partialize) 같은 메서드도 적극적으로 사용하면 좋을 듯 하다. 그만큼 유연하다.

### TypeScript와 함께 사용하기

위 코드에서 persist에 PersistTypes라는 타입을 사용했는데, 이 내부에서 State와 Actions에 대한 타입을 같이 선언해주면 된다. 위에서 사용한 것처럼 분리해도 좋고, 그냥 묶어서 하나의 타입으로 선언해도 좋을 것 같다.

```js
// .../store/localStorage/types.ts

interface ProductDetailProps {
  // TODO: 추후 response data 입력
}
type State<T> = {
  product: T | null,
};
type Actions<T> = {
  actions: {
    setProduct: (product: T) => void,
    clearProduct: () => void,
  },
};
type PersistTypes = State<ProductDetailProps> & Actions<ProductDetailProps>;
export type { ProductDetailProps, PersistTypes };
```

### Hook으로 추상화하기

zustand를 그냥 사용하다보면, store 호출 코드가 쌓이는데, 이걸 매번 반복하기가 생각보다 귀찮다. 컴포넌트 기능이 커지다 보면, 아래처럼 store 구독이 쌓이고 쌓인다.

store 안에서 상태와 액션을 같이 관리하다 보니, 관리할 때는 편리하지만 또 사용할 때는 관심사 분리를 어느 정도 해주면 좋을 때 이렇게 Hook으로 분리하는 것도 좋은 선택지라고 생각한다.

### 추상화 시, 주의해야 할 포인트 2가지

`shallow` : store에서 상태와 액션을 사용할 때는 shallow를 사용해서 불필요한 렌더링을 막으면 좋다. 지금 사용하는 zustand에서는 useShallow로 감싸주면 된다.

`next js에서 사용하는 방법` : [github](https://github.com/pmndrs/zustand/blob/main/docs/integrations/persisting-store-data.md#usage-in-nextjs)에서 아래와 같이 useStore 훅을 사용해 hydration 에러를 방지하는 예시를 보여주고 있다. 그대로 사용했다.

````js
// .../store/localStorage/hooks.ts

import { useShallow } from "zustand/react/shallow";
import useLocalStore from "./store";
import { useEffect, useState } from "react";

/**
 *
 * @description SSR Hydration Error 방지를 위한 클라이언트 훅
 */
const useStore = <T, F>(
  store: (callback: (state: T) => unknown) => unknown,
  callback: (state: T) => F
) => {
  const result = store(callback) as F;
  const [data, setData] = useState<F>();  useEffect(() => {
    setData(result);
  }, [result]);  return data;
};

/**
 *
 * @use
 * ```js
 * const productDetail = useGetProductDetailInLocalStore();
 * ```
 */
const useGetProductDetailInLocalStore = () =>
  useStore(
    useLocalStore,
    useShallow((state) => state.product)
  );/**
 *
 * @use
 * ```js
 * const setProductDetail = useSetProductDetailInLocalStore();
 * setProduct({ .. });
 * ```
 */
const useSetProductDetailInLocalStore = () =>
  useLocalStore(useShallow((state) => state.actions.setProduct));/**
 *
 * @use
 * ```js
 * const clearProductDetail = useClearProductDetailInLocalStore();
 * ```
 */
const useClearProductDetailInLocalStore = () =>
  useLocalStore(useShallow((state) => state.actions.clearProduct));export {
  useGetProductDetailInLocalStore,
  useSetProductDetailInLocalStore,
  useClearProductDetailInLocalStore,
};
````

위 코드에서는 상태와 액션을 분리해 hook으로 만들었지만, 굳이 이렇게 하지 않아도 이미 잘 추상화된 훅이므로 그대로 써도 무방하다.
