# 리액트에서 deps가 1(또는 2)인 객체의 속성을 상태로 관리하기

객체는 참조타입이다.<br>
참조타입을 리액트에서 상태로 관리하려면, 메모리 상에서 새롭게 선언해줘야 한다.<br>
deps가 깊으면 별도의 라이브러리를 사용하거나, 커스텀 훅이나 유틸함수를 제작해 관리해야 한다.

객체의 deps를 통일하고(예를 들어 1로 통일), 객체의 속성 별로 상태를 관리하는 방식이 있다.

### 유틸함수

상태 별로 타입은 전부 다르다. 객체의 개별 key 값을 관리하는 상태를 주입하면, setter 훅을 만들어주는 유틸함수다.

````ts
/**
 * @param input 사용할 state
 * @param setInput 사용할 setState
 * @description setByKey를 생성해주는 함수입니다.
 * @example
 * ```jsx
 * const setPartByKey = setObjectByKey(partInput, setPartInput)
 *
 * return (
 * onChange={(event) => setPartByKey('manager', event.target.value)}
 * )
 * ```
 */

export const setObjectByKey =
  <T extends Record<string, any>>(input: T, setInput: (input: T) => void) =>
  (key: keyof T, value: T[keyof T]) => {
    setInput({
      ...input,
      [key]: value,
    });
  };
````

### 사용 방법

사용할 때는 아래와 같이 사용한다.

setObjectByKey에 상태 값을 넣어서 만든 setInputByKey라는 함수에는 이제 key 값과, 해당 key의 변경될 value를 넣어주기만 하면, 객체의 상태를 쉽게 관리할 수 있다.

```tsx
const Component = () => {
  const [input, setInput] = useState({
    color: "red",
    backgroundColor: "blue",
  });

  const setInputByKey = setObjectByKey(input, setInput);

  return (
    <Ui
      changeColor={(color) => setInputByKey("color", color.name)}
      changeBackgroundColor={(background) =>
        setInputByKey("backgroundColor", background.colorName)
      }
    />
  );
};
```

### 객체의 깊이가 2라면?

깊이가 2보다 더 크다면, 재귀 함수를 사용해야 한다.<br>
깊이가 단순히 2라면 아래와 같이 유틸함수를 변경할 수 있다.

```ts
const setNestedInputByKey = <K extends keyof OriginalType>(
  key: K,
  value: keyof NestedType extends keyof OriginalType[K]
    ? NestedType[keyof NestedType]
    : OriginalType[K],
  nestedKey?: keyof NestedType
) => {
  setInput((original) => {
    // 한번 더 중첩되는 객체 상태 관리를 위해 nestedKey가 있는 경우와 없는 경우를 구분
    if (nestedKey) {
      return {
        ...original,
        [key]: {
          ...(original[key] as object),
          [nestedKey]: value,
        },
      };
    }
    return {
      ...original,
      [key]: value,
    };
  });
};
```

3번째 인자로 nestedKey를 받는다. 만약 nestedKey가 존재하면, 한번 더 깊게 객체를 분해해 반환한다.<br>
2번째 인자의 경우, 만약 NestedType이 들어오면 그 값으로 하고, 아니라면 OriginalType으로 지정한다.

사용법은 아래와 같다.

```ts
const exampleObject = {
  color: "red",
  backgroundColor: {
    second: "blue",
  },
};

setNestedInputByKey("color", "blue");
setNestedInputByKey("backgroundColor", "red", "second");
```
