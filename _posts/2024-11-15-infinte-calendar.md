---
title: [React] Infinite Calendar을 만들어보자 - 1
author: Seungwon
date: 2024-11-15
category: front, next, calendar
layout: post
---

## 세로 스크롤이 가능한 캘린더

이번에 모바일 화면에 맞는 캘린더를 제작하려고 합니다. 필요한 기능으로는 다음과 같이 있습니다.

1. 세로로 스크롤이 가능해야한다.
2. 기간 선택이 가능해야한다.
3. 스크롤말고 년, 월 선택으로 이동 가능해야한다.

## 라이브러리

먼저 어떠한 라이브러리가 있는지 확인해봤습니다.

### react-infinite-calendar [#](https://github.com/clauderic/react-infinite-calendar)

원하는 기능은 어느정도 들어가 있지만 마지막 커밋이 7년전입니다.. 그리고 원하는 디자인이랑 기능 추가하려면 배보다 배꼽이 더 커질 것 같아서 새로 만들기로 했습니다.

## @tanstack/react-virtual [#](https://tanstack.com/virtual/latest)

단순히 생각해도, 5년간의 달력을 보여주려면 못해도 12달 \* 5 = 60달 만큼이 필요합니다. 이걸 전부 다 그려놓고 스크롤하는 것은 올바른 접근 방법이 아닐 것이라고 생각해서 react-infinite-calendar에서는 어떻게 구현했는지 찾아봤습니다.

### react-tiny-virtual-list [#](https://github.com/clauderic/react-tiny-virtual-list)

react-infinite-calendar는 위의 virtual list를 통해서 구현했다고 깃허브에서 확인할 수 있었습니다. 모든 리스트를 그리는 것이 아니라, 아이템별 높이를 예측하여 스크롤 박스에서 보이는 개수를 유추하고 그만큼만 그리는 방식으로 성능 이슈를 해결한 것 같았습니다.

하지만 위의 라이브러리 또한 마지막 커밋이 7년전이었고, 저는 아직도 관리가 되고 있는 라이브러리를 사용하고 싶었습니다.

그렇게 찾게된 것이 리액트 쿼리로 유명한 tanstack의 @tanstack/react-virtual 라이브러리였습니다.

태그한 공식 홈페이지에서 예제를 확인하시면, 화면에 보이는 부분만 dom에 띄워지는 것을 확인하실 수 있습니다.

## Coding

먼저 react-virtual 세팅이 필요합니다. 저는 다음과 같이 세팅했습니다.

```javascript
const scrollRef = useRef < HTMLDivElement > null;
// vars
const yearList = [...years, years[years.length - 1] + 1]; // add one more year for the last month(December)

const virtualizer = useVirtualizer({
  count: years.length * 12 + 1, // 12 months * years + 1 for the last month and index is the number of months
  getScrollElement: () => scrollRef.current,
  estimateSize: (index) => {
    const year = yearList[index > 0 ? Math.floor(index / 12) : 0];
    const month = (index % 12) + 1;
    const weeks = Math.ceil(getDayList(`${year}-${month}-01`).length / 7);
    const gaps = weeks - 1;
    return weeks * 32 + gaps * 4;
  },
  overscan: 0, // to reduce calculation error
  gap: 4,
});

const items = virtualizer.getVirtualItems(); // virtual items
```

저는 아이템을 '월'로 설정했고 마지막 년도의 마지막달 이후의 1달이 더 필요했기에 count에 년도 \* 12 + 1 을 했습니다. estimateSize는 프로젝트에 맞게 설정해주시면 됩니다.

화면은 TailwindCSS를 같이 사용했습니다.

```html

<div
  className="scrollbar-hide flex h-52 w-full overflow-y-scroll"
  ref={scrollRef}
>
  <div
    className="relative flex w-full flex-col"
    style={{
      height: `${virtualizer.getTotalSize()}px`,
    }}
  >
    {items.map((item) => (
      <div
        className="absolute grid w-full grid-cols-7 gap-y-1"
        style={{
          height: `${item.size}px`,
          transform: `translateY(${item.start}px)`,
        }}
        ref={virtualizer.measureElement}
        key={item.key}
      >
        <Month
          date={`${yearList[Math.floor(item.index / 12)]}-${
            (item.index % 12) + 1
          }-01`}
          startDate={startDate}
          endDate={endDate}
          nowDate={curr.format("YYYY-MM-DD")}
          onClick={onClick}
        />
      </div>
    ))}
  </div>
</div>
```

여기서는 스크롤 박스의 높이와 너비를 설정해주고, virtual items에 대해서 index로 '달'을 정하고 있다 까지만 봐주시면 됩니다. 주의하실 점으로, gap이나 margin, padding을 설정하셨다면 이를 고려해서 estimateSize 값을 설정하셔야 합니다. (참고로 날짜 라이브러리는 dayjs를 사용했습니다.)

@tanstack/react-virtual 공식문서를 참조하여, virtual items에 대해 미리 구현된 다양한 옵션을 활용해보시면 더욱 편하게 기능 구현하실 수 있습니다.

## 끝

간단란 라이브러리 소개와 사용방법까지만 알려드리고 상세 구현은 다음 글에서 확인하시면 됩니다.
