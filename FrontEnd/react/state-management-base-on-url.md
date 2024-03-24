# Next.js에서 URL로 상태를 관리할 때 사용하기 좋은 Custom hook 만들기

React에서 UI를 변경할 때, 일반적으로 useState로 상태를 변경한다.<br>
이렇게 상태를 관리할 때의 문제점은 저장되지 않는다는 점이다..<br>
뒤로 가기를 누르거나, 해당 UI를 유지한 상태로 url을 공유하고 싶지만, useState만으로는 어렵다.

그래서 url에 state를 저장하고 그 url로 이동하는 식으로 UI를 구현할 수도 있다.

예를 들어, next.js에서의 예시는 다음과 같다.

```tsx
import { useRouter, usePathname } from "next/navigation";

const Component = () => {
  const colors = {
    red: "red",
    blue: "blue",
  };
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const currentColor = searchParams.get("color");

  const changeButtonColorToBlue = () => {
    router(`${pathname}${colors["blue"]}`);
  };

  return (
    <button
      type="button"
      onClick={changeButtonColorToBlue}
      style={{
        color: currentColor,
      }}
    ></button>
  );
};
```

useSearchParams를 통해 color라는 쿼리스트링 값을 가져오고,<br>
변경할 때는 router.push를 통해 url로 이동해 값을 변경하는 방식이다.

### 위 코드의 문제점은?

‘확장성’에 있다.  
컴포넌트가 복잡해지다보면, router.push에 다른 상태들을 전부 고려하지 않으면 router.push로 다른 상태들을 덮어씌워버리는 문제가 생긴다.

예를 들어 `{pathname}?color={color}&size={size}`가 있을 때,  
단순히 `` router.push(`${pathname}?color={color}`) ``라고만 하면 size에 대한 상태를 잃어버린다.

그렇다고 모든 상태를 고려하려다 보면, 개별 컴포넌트들이 전혀 상관없는 다른 컴포넌트들의 상태들을 전부 관리해야 한다는 단점이 있었다.

이럴 때, 아래와 같은 커스텀 훅을 고려해볼 수 있다.

```tsx
import { usePathname, useRouter, useSearchParams } from "next/navigation";

const useRouteWithAddingParam = () => {
  const pathname = usePathname();

  const router = useRouter();

  const searchParams = useSearchParams();

  /**  
      
    *  
      
    * @param currentParamKey 추가할 querystring key  
      
    * @param currentParamValue 추가할 querystring value  
      
    */

  const routeWithAddingParam = (
    currentParamKey: string,
    currentParamValue: string
  ) => {
    // forEach를 사용해 `key=value` 형태로 paramList에 추가

    const _paramList: string[] = [];

    searchParams.forEach((paramValue, paramKey) => {
      if (paramKey === currentParamKey) return;

      _paramList.push(`${paramKey}=${paramValue}`);
    });

    // prefix로 ?를 추가할지, &를 추가할지 결정

    const params = _paramList

      .map((param, index) => {
        if (index === 0) return `?${param}`;

        return `&${param}`;
      })

      .join("");

    // 추가할 param의 형태 결정

    const paramsToBeAdded = params
      ? `&${currentParamKey}=${currentParamValue}`
      : `?${currentParamKey}=${currentParamValue}`;

    router.push(`${pathname}${params}${paramsToBeAdded}`);
  };

  return { router, pathname, searchParams, routeWithAddingParam };
};

export default useRouteWithAddingParam;
```

next.js의 searchParams 기능을 통해 현재 쿼리스트링 key,value를 전부 가져온다.<br>
그리고 router.push에서는 무조건 현재 쿼리스트링 뒤에 추가할 state들을 추가하게끔 한다.<br>
이렇게 되면, useRouteWithAddingParam을 사용할 때 아래와 같이 단순하게 사용할 수 있다.

```ts
const Component = () => {
  const { routeWithAddingParam } = useRouteWithAddingParam();

  const changeButtonColorToBlue = () => {
    // url로 관리하고 싶은 State의 key와 value만 알고 있으면 이렇게 쉽게 관리할 수 있습니다.
    routeWithAddingParam("color", "blue");
  };
  // 생략
};
```
