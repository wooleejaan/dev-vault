# 애니메이션 성능 기초 — 종료조건 준수

### Confetti 애니메이션

프로젝트에서 Confetti 애니메이션을 구현해야 해서, 아래 처럼 급하게 구글 검색으로 구현한 적이 있었다.

![[1.gif]]

레퍼런스가 충분해서 구현 자체는 어렵지 않았는데, 아주 큼직한 문제가 하나 있었다.

### 무한히 실행되는 requestAnimationFrame 문제

처음 애니메이션을 띄우고 나서, 앱 자체가 확연하게 느려지는 게 문제였다.

처음엔 단순하게 콘솔이 너무 많이 찍히는 건가? 해서 throttle을 적용했는데, 해결되지 않았다.

```js
useEffect(() => {
	const canvas = canvasRef.current as HTMLCanvasElement;
	canvas.width = window.innerWidth;
	canvas.height = window.innerHeight;

	window.addEventListener("resize", throttleHelper(1000, resizeCanvas));

	return () => {
		window.removeEventListener("resize", throttleHelper(1000, resizeCanvas));
	};
}, []);
```

적용해도 해결이 안 되길래 콘솔로 찍어봤더니 아래 gif처럼 함수가 무한하게 실행되고 있었다. 이게 계속 쌓이니 앱이 느려지고 있었던 것이다.

![[2.gif]]

### 해결 — 배열 종료 조건을 부여하자

며칠을 고민하다가 깨닫게 된 건, 빈 배열이 찍히는 데도 계속해서 함수가 실행된다는 점이었다.

Confetti 애니메이션을 구현할 때, 여러 개의 종이 애니메이션을 생성하고 배열에 삽입했었다. 그리고 그 배열을 순회하며 애니메이션에 임의의 방향을 부여하며 Confetii를 구현하는 건데, 배열이 비워지는 순간 함수 실행을 종료하니 문제를 해결할 수 있었다.

```js
const renderConfetti = () => {
  // (애니메이션 구현 부분 생략...)

  /**
   * requestAnimationFrame을 통해 매 프레임마다 renderConfetti 함수가 호출되므로
   * 아래와 같이 renderConfetti 함수 호출을 멈추도록하는 조건을 설정해야 합니다.
   * 그러지 않으면 renderConfetti 함수 호출이 너무 많아져 앱이 느려집니다.
   */

  confettiRef.current.forEach((confetto, index) => {
    if (confetto.position.y >= canvas.height) {
      confettiRef.current.splice(index, 1); // 요소 제거
    }
  });

  sequinsRef.current.forEach((sequin, index) => {
    if (sequin.position.y >= canvas.height) {
      sequinsRef.current.splice(index, 1); // 요소 제거
    }
  });

  // 모든 요소가 제거되면 애니메이션 중지
  if (confettiRef.current.length === 0 && sequinsRef.current.length === 0) {
    return;
  }

  requestAnimationFrame(renderConfetti);
};
```

![[3.gif]]

이제 빈 배열이 되는 순간 함수 실행도 멈추므로 실행량이 확연하게 줄은 걸 확인할 수 있다.

### Confetti 애니메이션 구현 코드

[https://github.com/wooleejaan/react-playground/tree/main/src/pages/ConfettiPage](https://github.com/wooleejaan/react-playground/tree/main/src/pages/ConfettiPage)
