# 리액트 커스텀 훅 모음

### useDisableScroll

개발을 하다 보면, 스크롤을 막아야 할 때가 종종 있다. 특히 Modal 같은 데서 유용하게 썼다.

```ts
import { useEffect } from "react";

const useDisableScroll = (): void => {
  useEffect(() => {
    const disableScroll = (event: Event): void => {
      event.preventDefault();
    };
    document.addEventListener("wheel", disableScroll, { passive: false });
    document.addEventListener("touchmove", disableScroll, { passive: false });
    return () => {
      document.removeEventListener("wheel", disableScroll);
      document.removeEventListener("touchmove", disableScroll);
    };
  }, []);
};
export default useDisableScroll;
```

### useDisableToBack

모바일 기기에서 뒤로가기 이벤트가 말썽을 부릴 때가 있다.
그럴 때 임시방편으로 뒤로가기 대신 특정 모달을 닫는 식으로 해결하고 싶을 때 쓰면 좋다.

```ts
import { useEffect } from "react";

const useDisableToBack = (onClickCloseModal: () => void) => {
  useEffect(() => {
    const handlePopstateBackward = () => {
      window.history.pushState(null, document.title, window.location.href);
      onClickCloseModal();
    };
    window.history.pushState(null, document.title, window.location.href);
    window.addEventListener("popstate", handlePopstateBackward);
    return () => {
      window.removeEventListener("popstate", handlePopstateBackward);
    };
  }, [onClickCloseModal]);
};

export default useDisableToBack;
```

### useMediaQuery

반응형 구현할 때 사용하는 미디어쿼리 훅이다.
SSR 환경일 때 막는 로직도 추가되어 있다.

````ts
import { useEffect, useState } from "react";

/**  
 *  
 * @example 사용 예시  
 * ```  
 * const isMobile = useMediaQuery('(max-width: 768px)')  
  const isTablet = useMediaQuery('(max-width: 1200px)')  
  useEffect(() => {  
    if (isMobile && isTablet) {  
      console.log('모바일')  
    } else if (!isMobile && isTablet) {  
      console.log('태블릿')  
    } else {  
      console.log('데스크탑')  
    }  
  }, [isMobile, isTablet])  
 * ```  
 *  
 * @param query `(max-width: ${breakpoint}px)`  
 * @returns boolean  
 */
export function useMediaQuery(query: string): boolean {
  const getMatches = (q: string): boolean => {
    // Prevents SSR issues
    if (typeof window !== "undefined") {
      return window.matchMedia(q).matches;
    }
    return false;
  };
  const [matches, setMatches] = useState<boolean>(false);
  function handleChange() {
    setMatches(getMatches(query));
  }
  useEffect(() => {
    const matchMedia = window.matchMedia(query);
    // Triggered at the first client-side load and if query changes
    handleChange();
    // Listen matchMedia
    if (matchMedia.addEventListener) {
      matchMedia.addEventListener("change", handleChange);
    } else {
      matchMedia.addEventListener("change", handleChange);
    }
    return () => {
      if (matchMedia.removeEventListener) {
        matchMedia.removeEventListener("change", handleChange);
      } else {
        matchMedia.removeEventListener("change", handleChange);
      }
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [query]);
  return matches;
}
````

### useOutsideClick

바깥을 눌렀을 때 창을 닫을 수 있게 하는 훅이다. ref와 callback을 파라미터로 받는다.

````ts
"use client";

import { useEffect } from "react";

/**
 * @param 사용 예시
 *
 * ```
 * const kebabRef = useRef<HTMLDivElement | null>(null);
 *
 * useOutsideClick(kebabRef, handleClickCloseKebab);
 * ```
 *
 * @param ref 닫고 싶은 대상 ref
 * @param callback 닫는 동작 핸들러
 */
const useOutsideClick = (
  ref: React.MutableRefObject<HTMLDivElement | null>,
  callback: () => void
) => {
  const handleClick = (e: React.BaseSyntheticEvent | MouseEvent) => {
    if (ref.current && !ref.current.contains(e.target)) {
      callback();
    }
  };
  useEffect(() => {
    setTimeout(() => {
      document.addEventListener("click", handleClick);
    }, 0);
    return () => {
      document.removeEventListener("click", handleClick);
    };
  });
};

export default useOutsideClick;
````

```ts
import { useEffect, useRef } from "react";

export const useOutsideClick = (callback: () => void) => {
  const ref = useRef<HTMLDivElement>(null);
  useEffect(() => {
    const handleClickOutside = (event: MouseEvent) => {
      if (ref.current && !ref.current.contains(event.target as Node)) {
        callback();
      }
    };

    document.addEventListener("mousedown", handleClickOutside);
    return () => {
      document.removeEventListener("mousedown", handleClickOutside);
    };
  }, [callback]);

  return ref;
};
```

### PortalWrapper

React Portal을 사용할 때, 자주 쓰는 wrapper

```ts
"use client";

import { createPortal } from "react-dom";
interface IPortalWrapperProps {
  children: React.ReactNode;
  id: string;
}
/**
 *
 * @example 사용 예시
 * <PortalWrapper id="toast-portal">
 * <Toast onShow={onShow}>{children}</Toast>
 * </PortalWrapper>
 */
export default function PortalWrapper({ children, id }: IPortalWrapperProps) {
  if (typeof window !== "object") {
    // Check if document is finally loaded
    return;
  }
  const el = document.getElementById(id) as HTMLElement;
  return createPortal(children, el);
}
```

### useAsync

어떻게 보면 제일 기본적인 api 훅. react query를 사용하기 시작하면서 요즘엔 잘 안 쓰는 것 같다.

```ts
import { useCallback, useState } from "react";

/**
 * 비동기 함수를 처리하고 해당 함수의 실행 결과와 상태를 관리하는 커스텀 훅
 *
 * @param {function} asyncFunction - 비동기 함수
 * @returns {object} - 데이터, 로딩 상태, 에러, 비동기 함수 호출을 관리하는 객체
 * @example
 * const { data, loading, error, callAsyncFunction: fetchGlobalData} = useAsync(AxiosSearch);
 * @expmple fetch 함수 사용: (async () => {await fetchGlobalData();})();
 */
const useAsync = (asyncFunction) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const callAsyncFunction = useCallback(
    async (...args) => {
      setLoading(true);
      setError(null);
      try {
        const result = await asyncFunction(...args);
        setData(result);
      } catch (error) {
        setError(error);
      } finally {
        setLoading(false);
      }
    },
    [asyncFunction]
  );
  return { data, loading, error, callAsyncFunction };
};

export default useAsync;
```

### useViewObserver

IntersectionObserver로 특정 노드를 관찰해야할 때 쓰는 훅이다. 노드가 여러개여도 대응 가능하도록 ref는 배열 형태를 갖습니다. 이 refs를 파라미터로 분리해서 쓸 수도 있다.

```ts
import { useEffect, useRef, useState } from "react";

const useViewObserver = () => {
  const observerTargetRefs = useRef<HTMLDivElement[]>([]);
  const [inView, setInView] = useState<boolean | null>(null);
  useEffect(() => {
    const io = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          setInView(entry.isIntersecting);
        });
      },
      {
        threshold: 0.8,
      }
    );
    observerTargetRefs.current.forEach((ref) => io.observe(ref));
    return () => {
      io.disconnect();
    };
  }, []);

  return { inView, observerTargetRefs };
};

export default useViewObserver;
```

### usePathnameWithHash

GNB에서 hash를 이용해서 list를 구성해야 할 때,
서버 컴포넌트에서의 렌더링과 충돌하지 않으면서 hash가 포함된 pathname을 만들어주는 커스텀 훅

중요한 점은, `typeof window === 'undefined'` 처리 로직을 단순히 변수에 처리하지 않고, useEffect에서 처리해야 한다는 점이다. (SSR 렌더링과 다르지 않게 하려면 useEffect를 타는 게 편하다.)

```tsx
"use client";

import { usePathname, useSearchParams } from "next/navigation";
import { useEffect, useState } from "react";

const usePathnameWithHash = () => {
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const [currentHash, setCurrentHash] = useState<string | undefined>(undefined);

  useEffect(() => {
    if (typeof window === "undefined") return;
    setCurrentHash(window.location.hash);
  }, [searchParams]);

  return {
    pathname,
    searchParams,
    currentHash,
    currentPathnameWithHash: pathname + currentHash,
  };
};

export default usePathnameWithHash;
```
