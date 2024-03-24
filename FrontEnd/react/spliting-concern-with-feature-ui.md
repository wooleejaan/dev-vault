# 배열로 feature과 ui 관심사 분리하기

컴포넌트가 비대해지고 prop이 많아질 때 + prop들이 하나의 참조 타입 데이터에서 나온다는 공통점이 존재할 때,

- 배열을 사용해서 데이터 정리는 return 문 위에 작성하고
- return 문 아래에는 배열의 속성만 부여할 수 있게 map 메서드를 사용하면
- 데이터를 뿌려주는 쪽과 데이터를 받는 쪽으로 코드를 리팩토링할 수 있다.

```ts
const FeatureComponent = () => {

	const exampleListMap = useMemo(() => {
		switch (suitableShapeType) {
			case Enum.Corporate:
				return {
					hasCorporate: true,
					subNameOnlyForOval: undefined,
					hasRound: false,
					hasOval: false,
					hasSquare: false,
				}
			case Enum.All:
				return {
					hasCorporate: false,
					subNameOnlyForOval: subName,
					hasRound: true,
					hasOval: true,
					hasSquare: true,
				}
			case Enum.OnlySquare:
				return {
					hasCorporate: false,
					subNameOnlyForOval: undefined,
					hasRound: false,
					hasOval: false,
					hasSquare: true,
				}
			case Enum.OnlyOval:
				return {
					hasCorporate: false,
					subNameOnlyForOval: name,
					hasRound: false,
					hasOval: true,
					hasSquare: false,
				}
		}
	}, [suitableShapeType])

return (
	<div>
		{exampleListMap.map((fontType, index) => (
			<UiComponent
				key={index.toString()}
				currentId={currentId}
				fontType={fontType}
				name={name}
				setId={setId}
				subNameOnlyForOval={exampleListMap?.subNameOnlyForOval}
				hasCorporate={exampleListMap?.hasCorporate}
				hasRound={exampleListMap?.hasRound}
				hasOval={exampleListMap?.hasOval}
				hasSquare={exampleListMap?.hasSquare}
			/>
		))}
	<div/>
	)
}

export default FeatureComponent
```
