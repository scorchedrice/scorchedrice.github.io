---
title: "[Algorithm, JavaScript] 전개연산자 런타임 에러는 왜 발생하나 (PCCP 기출 - 퍼즐게임 챌린지)"
author: scorchedrice
date: 2024-12-2 16:00:00 +0900
categories: [Algorithm, JavaScript]
tags: [이분탐색, 프로그래머스, 전개연산자, 런타임에러]
render_with_liquid: false
---

# [PCCP 기출] 퍼즐게임 챌린지

https://school.programmers.co.kr/learn/courses/30/lessons/340212

구현 자체는 어렵지 않았으나, 레벨을 선택하는 과정에서 시간을 단축해야 통과할 수 있던 문제다.

단순 이분탐색이지만, `Math.max` 와 같은 전개연산자를 사용할 때 런타임에러가 발생하는 것을 확인하여 포스팅한다.

## 풀이과정

문제를 풀 때 나는 다음의 과정을 통해 문제풀이를 진행했다.

1. 누적합을 담은 리스트를 생성한다. 첫 문제는 무조건 난이도 1짜리 문제가 생기므로 (무조건 푸니까) 해당 시간을 담은 리스트를 담는다.
2. 이후 누적합 리스트를 쌓는 식으로 진행한다. 문제를 한번에 풀 수 있는 경우 해당 문제풀이 시간과 prev 시간값을 더해 리스트의 마지막 값이 현재까지 사용한 시간 값이 되도록 했다.
3. 만약 한번에 풀 수 없는 경우 반복해야 하는 횟수를 구하고 prev + current 를 한 cycle로 정의해 둘을 곱한값과 current값을 더한 값을 누적합 리스트에 쌓는다.
4. 이와 같은 기능을 levelTest 함수를 따로 생성하고, solution function에서는 level을 바꾸며 해당 함수를 진행한다.

```ts
function levelTest(diffs, times, level, limit) {
  // 현재 레벨을 입력하면, 해당 레벨에 맞춰 누적 합 연산을 진행해본다.
  // 연산이 완료된다? 그러면 숫자를 제공한다. (solution function에서 이를 통해 유효성을 판단한다.)

  // 시간을 누적하면서 리스트를 형성하려 한다.
  // 첫번째는 무조건 푸니까 일단 넣는다.
  const timeList = [times[0]]
  // 이렇게 누적하던 도중 누적합이 limit을 벗어나면 더이상 진행할 필요 없음.
  for (let i = 1; i < diffs.length; i++) {
    const pivot = level - diffs[i];
    if (pivot >= 0) {
      const latestTime = timeList[i-1] + times[i]
      if (latestTime > limit) {
        // console.log(timeList)
        return false
      }
      timeList.push(latestTime)
    } else {
      // pivot이 음수인 경우 한번에 풀이가 불가능함.
      const cycle = times[i-1] + times[i]
      // 반복한 사이클 + 현재 시간 + 이전 누적 합
      const latestTime = Math.abs(cycle * pivot) + times[i] + timeList[i-1] 
      if (latestTime > limit) {
        return false
      }
      timeList.push(latestTime)
    }
  }
  // 위에 있는 반복문을 통과했다? 그렇다면 유효한 값이다.
  return level
}

function solution(diffs, times, limit) {
  let minLev = 1;
  // 런타임 에러 발생할 수 있음. Math.max에 들어가는 값이 크다면
  // 배열의 크기가 크다면 배열 연산자의 사용을 조심해야 한다.
  let maxLev = -1;
  // maxLev = Math.max(...diffs) => error
  for (diff of diffs) {
    if (maxLev < diff) {
      maxLev = diff
    }
  }
  let ans = maxLev

  while (maxLev >= minLev) {
    const level = Math.floor((minLev + maxLev) / 2)
    const returnValue = levelTest(diffs, times, level, limit)
    if (typeof returnValue === 'number') {
      // 이 경우 연산이 되었단 말이니, 레벨을 낮출 필요성이 있다.
      ans = level
      maxLev = level - 1
    }
    else {
      minLev = level + 1
    }
  }
  return ans
}
```

처음엔 단순 level + 1을 하며 하나하나 계산하다보니 시간초과가 발생했다.

이후 이분탐색 방법을 생각했고, 진행했으나 런타임에러가 발생했다.

사실 이유를 찾지 못해 이것저것 찾아보았고 다음의 내용을 알 수 있었다.

## 전개연산자 (Spread Operator)

전개연산자(...)는 ES6에서 도입된 기능이다. 배열이나 객체를 펼쳐 개별요소로 분리해주는 문법이다.

```js
const a = [1,2];
const b = [3,4];

console.log([a,b]) // [[1,2], [3,4]]
console.log([...a, ...b]) // [1,2,3,4]

console.log(Math.max(a)) // error!
console.log(Math.max(...a)) // 2

const copyA = [...a] // a값이 복사
```

이처럼 배열을 조작하고 함수의 인자로 사용하는 등 활용처가 많다. 하지만 전개연산자는 사용 시 주의해야 하는 사항들이 있다.

### JavaScript의 동작 방식과 전개연산자의 상관관계

<img src="/assets/img/Algorithm/programmers/241209/node_eventloop.webp" alt="node event loop">

출처 : https://smit90.medium.com/deep-dive-into-the-event-loop-understanding-node-js-internals-f9263ef91233

JavaScript V8엔진은 함수를 호출할 때 인자를 스택에 푸시한다. 싱글스레드(하나의 콜스택)로 콜스택에 명령을 쌓고, 이를 빼는 과정으로 연산을 진행한다. 

근데, 전개연산자는 모든 배열 요소를 개별 인자로 반환하기에 함수에 전개연산자를 활용한다면 스택오버플로우 및 메모리 부족 현상이 발생할 수 있다.

따라서 전개연산자가 가독성이 좋다는 장점이 있지만 큰 배열의 가능성이 있는 경우, 이와 같은 코드는 자제해야 한다. 다음과 같은 코드로 오류를 방지하자.

```js
// 작은 배열이 확실한 경우
const smallArray = [1, 2, 3, 4, 5];
const max = Math.max(...smallArray);

// 배열이 매우 큰 경우
const hugeArray = new Array(100000).fill(1);
const max = Math.max(...hugeArray);  // 위험!

// 대신 reduce 사용. 혹은 반복문 사용
const max = hugeArray.reduce((a, b) => Math.max(a, b));
```

## 결론

JavaScript의 동작원리에 대해선 원래도 공부를 해둬서 알고있던 내용이긴하다.
근데 알고리즘 문제를 풀면서 이와 같은 내용을 다시 보다니 .. 빠른 시일내에 관련 내용 한번 더 정리해봐야겠다. 
