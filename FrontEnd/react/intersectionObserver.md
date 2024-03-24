# intersectionObserver가 여러 개의 ref를 바라봐야 할 때

intersectionObserver를 사용해서 어떤 요소가 화면에 보이는지 체크할 수 있다. (entry.isIntersecting)

react에서는 useRef를 사용해서 요소를 체크할 수 있다.<br>
이때 ref가 여러 개면, 매번 new IntersectionObserver를 통해 observer를 갱신해줘야 한다.

```ts
let observer: IntersectionObserver;

targetRefs.forEach((targetRef) => {
  if (targetRef?.current) {
    observer = new IntersectionObserver(onIntersect, options);
    observer.observe(targetRef.current);
  }
});

return () => observer?.disconnect();
```

커스텀 훅의 전체 코드를 보면, ref들을 인자로 받는다.

```tsx
// useIntersectionObserver.ts

import { DependencyList, RefObject, useEffect } from "react";

const useIntersectionObserver = (
  targetRefs: RefObject<Element>[],
  onIntersect: IntersectionObserverCallback,
  deps?: DependencyList, // sideEffect 필요한 변수
  options?: IntersectionObserverInit
) => {
  useEffect(() => {
    let observer: IntersectionObserver;

    targetRefs.forEach((targetRef) => {
      if (targetRef?.current) {
        observer = new IntersectionObserver(onIntersect, options);
        observer.observe(targetRef.current);
      }
    });

    return () => observer?.disconnect();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [onIntersect, ...targetRefs, ...(deps ?? [])]);
};

export default useIntersectionObserver;
```

컴포넌트에서는 ref들을 선언해주고,<br>
IntersectionObserverCallback 함수를 선언해준다.

```tsx
// Component.tsx
'use client'

import useIntersectionObserver from '@/libs/shared/hooks/useIntersectionObserver'


const Component = () => {
	const ctaHeroRef = useRef<HTMLDivElement | null>(null)
	const ctaContractRef = useRef<HTMLDivElement | null>(null)
	const [showHero, setShowHero] = useState<boolean | undefined>(undefined)
	const [showContract, setShowContract] = useState<boolean | undefined>(undefined)

	const onIntersectCharacter = ([entry]: IntersectionObserverEntry[]) => {
		if (isDesktop || typeof window === undefined) {
	      return
		}

		if (entry.target === ctaContractRef.current) {
		  setShowContract(entry.isIntersecting)
		}
		if (entry.target === ctaHeroRef.current) {
		  setShowHero(entry.isIntersecting)
		}
	  }

    useIntersectionObserver(
	  [ctaHeroRef, ctaContractRef],
      onIntersectCharacter,
	  [
		isDesktop,
		showHero,
		showContract,
	  ],
	  {
	    threshold: 0,
	  },
  )

  return (
    <div/>
      <div ref={ctaHeroRef}>{hero}</div>
      <div ref={ctaContractRef}>{contact}</div>
      <ViewButton isView={showHero === false && showContract === false} />
    </div>
  )
}

export default Component
```

이때, 단순히 onIntersectCharacter 함수에서 isIntersecting을 체크해서 요소가 화면에 보이는지 체크하면 문제가 생긴다. 한 페이지에 여러 요소가 같이 존재하다보면, entry가 생성되는 시점과 순서에 따라 isIntersecting이 꼬이는 순간들이 온다.

그래서 useState로 각 요소에 대한 별도 state를 선언해주고,<br>
entry의 target이 ref.current일 때만 state를 별도로 관리해주고,<br>
그렇게 관리한 각각의 state를 isView라는 조건에서 한번에 묶어서 관리해주면 된다.

그러면 요소가 많아지더라도 intersectionObserver로 intersecting을 꼬임없이 체크할 수 있다.
