# mui의 lineChart에 click 이벤트 부여하기

@mui/x-charts 라이브러리의 LineChart를 사용한 그래프가 있다.<br>
안타깝게도 해당 라이브러리의 LineChart는 onClick과 같은 prop을 전달하지 못하게 되어 있다.

해당 차트 컴포넌트에 onClick 이벤트를 부여하고,<br>
클릭했을 때 해당 tick과 관련된 데이터를 다루는 게 목표다.

어떻게 할까 고민하다가, useRef를 선택했다.

useRef.current에 직접적으로 이벤트 리스너를 부착해줬고,<br>
클릭했을 때의 getBoundingClientRect()를 사용해서 현재 클릭했을 때 가장 가까운 index를 찾는다.<br>
그렇게 찾은 index를 기반으로 해당 차트 막대기와 관련한 데이터들을 자유롭게 다룰 수 있다.

```tsx
const Component = () => {
  // 기타 state 생략 ...
  const lineChartRef = useRef<SVGSVGElement>(null);
  const [openDetailModal, setOpenDetailModal] = useState<
    typeof result | undefined
  >(undefined);

  const openCurrentTickScheduleModal = (event: MouseEvent) => {
    const currentLineChartRect = lineChartRef.current?.getBoundingClientRect();
    if (!currentLineChartRect) {
      return;
    }
    const mouseX = event.clientX - currentLineChartRect.left;
    // xAxis tick 요소 리스트 (HTML Collection)
    const xAxisTicks = lineChartRef.current?.getElementsByClassName(
      "MuiChartsAxis-tickLabel"
    );
    if (!xAxisTicks) {
      return;
    }
    let minDistance = Number.MAX_SAFE_INTEGER;
    let currentIndex: number | undefined;

    for (let i = 0; i < xAxisTicks.length; i++) {
      const tickRect = xAxisTicks[i]?.getBoundingClientRect();
      if (!tickRect) {
        return;
      }
      // distance : 각 xAxisTick과 마우스 클릭 지점 간 수평 거리 (xAixTick의 중심과 마우스 클릭 지점 사이 거리)
      const centerOfXAixTick = tickRect.left + tickRect.width / 2; // xAixTick의 중심
      const pointOfMouseClick = currentLineChartRect.left + mouseX; // 마우스 클릭 지점 사이 거리
      const distance = Math.abs(centerOfXAixTick - pointOfMouseClick);
      if (distance < minDistance) {
        // 클릭했을 때, 현재 지점에서 가장 가까운 xAxisTick 요소 탐색
        minDistance = distance;
        currentIndex = i;
      }
    }
    if (currentIndex === undefined) {
      return;
    }

    setCurrentYear(dateLabels[currentIndex].year);
    setCurrentMonth(dateLabels[currentIndex].month);

    setOpenDetailModal(result);
  };

  useEffect(() => {
    lineChartRef.current?.addEventListener(
      "click",
      openCurrentTickScheduleModal
    );
    return () => {
      lineChartRef.current?.removeEventListener(
        "click",
        openCurrentTickScheduleModal
      );
    };
  }, [lineChartRef.current, year]);

  return (
    <LineChart
      ref={lineChartRef}
      // 기타 prop 생략 ...
    />
  );
};
```
