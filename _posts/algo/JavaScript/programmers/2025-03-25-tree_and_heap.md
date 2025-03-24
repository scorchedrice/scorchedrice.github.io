---
title: "[Algorithm, JavaScript] 힙과 이중트리 - 길찾기게임, 더맵게"
date: 2025-03-24 22:00:00 +0900
categories: [Algorithm, JavaScript]
tags: [프로그래머스, 트리, 힙]
render_with_liquid: false
---

## 힙과 이중트리

트리구조를 class로 구현하는 것은 할줄알았는데, 우선순위큐 같은 것들을 구현하는 것에 익숙하진 않았다. 개념은 알고있었으나 직접 써보지 않아 프로그래머스의 `더맵게`라는 문제를 풀 때 좀 고생했다.

아무튼, 관련 추후 있을 코딩테스트에도 대비하기 위해 정리한다. PCCP도 도전해야하니까~

### 이중트리

[길찾기게임](https://school.programmers.co.kr/learn/courses/30/lessons/42892)

나는 트리를 다음의 방식으로 구현했다.

1. 노드라는 클래스를 만든다.
2. 노드를 Tree라는 클래스에 넣으면서 구조를 만든다.
   - 자식 노드의 여부를 판단하는 로직이 필요하다.
   - left, right를 선택해서 배치되는 로직이 필요하다.

```js
class Node {
  constructor(node) {
    this.node = node;
    this.left = null;
    this.right = null;
  }
}

// node번호를 기준으로 이중트리를 구성하는 상황
// 우측엔 큰 값, 좌측엔 작은 값이 들어가는 이중트리
class Tree {
  constructor(root) {
    this.root = root
  }
  // root를 시작으로 좌우를 비교하며 노드 배치
  insert(element) {
    this._insert(this.root, element);
  }
  _insert(parent, child) {
    if (parent.node > child.node) {
      if (parent.left === null) {
        parent.left = child;
      } else {
        this._insert(parent.left, child);
      }
    } else {
      if (parent.right === null) {
        parent.right = child;
      } else {
        this._insert(parent.right, child);
      }
    }
  }
  // 전위 순회
  pre() {
    const arr = [];
    const result = _pre(this.root, arr);
    return result;
  }
  _pre(nd, arr) {
    if (nd) {
      arr.push(nd);
      this._pre(nd.left, arr);
      this._pre(nd.right, arr);
    }
    return arr;
  }
  // 후위 순회
  pos() {
    const arr = [];
    const result = _pos(this.root, arr);
    return result;
  }
  _pos(nd, arr) {
    if (nd) {
      this._pos(nd.left, arr);
      this._pos(nd.right, arr);
      arr.push(nd);
    }
  }
}
```

### 힙

사실 이 게시물을 작성하는 가장 큰 이유이다.

트리구조를 사용한다는 것은 알고있었다. 자연스럽게 위처럼 코드를 짜고있었는데, 지속적으로 노드간의 위치를 바꿔야하는 우선순위큐임을 깨달은 이후엔 구현하는게 상당히 까다로웠다.

이를 구현할 땐 다음의 과정으로 진행한다.

1. `null`을 미리 넣어 부모노드가 `Math.floor(currentNode / 2)`를 충족하도록 구현한다.
2. 삽입할 땐 가장 끝 노드에 삽입되므로 부모 노드를 향해 정렬하는 과정이 필요하다.
3. `heappop`을 할 땐 1번 노드에 있는 것을 빼야한다.
   - 마지막에 있는 노드를 1번에 임의 배치시켜 강제 재배열이 되도록 해야한다.
   - 루트노드만 존재하는 경우의 예외를 생각해야한다.

```js
// PQ : Priority Queue
class PQ {
  constructor() {
    this.pq = [null];
  }
  length() {
    return this.pq.length;
  }
  insert(element) {
    this.pq.push(element); // 일단 삽입한다.
    let currentIndex = this.pq.length -1;
    let parentIndex =  Math.floor(currentIndex/ 2) // null을 기본값으로 넣은 이유
    while (
      parentIndex !== 0 &&
        this.pq[currentIndex] > this.pq[parentIndex]
    ) {
      // 부모보다 자식이 큰 값이라면 바꿔야한다.
      [this.pq[parentIndex], this.pq[currentIndex]] = [this.pq[currentIndex], this.pq[parentIndex]]
      // 현재 값이 부모 노드위치로 올라갔으니 이를 반영한다.
      currentIndex = parentIndex;
      parentIndex = Math.floor(currentIndex / 2);
    }
  }
  heappop() {
    // 루트노드만 있는 경우의 예외를 처리한다.
    if (this.pq.length === 2) return this.pq.pop();
    // 최상단 노드 값을 임시 저장한다. 재배치를 완료한 이후에 이를 반환한다.
    const returnValue = this.pq[1];
    this.pq[1] = this.pq.pop(); // 마지막 노드값을 할당하여 강제 재배치를 진행한다.
    let currentIndex = 1;
    let leftIndex = 2;
    let rightIndex = 3;
    // 현 위치보다 자식들의 값이 더 크다면 바꿔야함.
    while (
      this.pq[currentIndex] < this.pq[leftIndex] ||
      this.pq[currentIndex] < this.pq[rightIndex]
    ) {
      // 이동할 때 더 큰 노드쪽으로 이동한다.
      if (this.pq[rightIndex] > this.pq[leftIndex]) {
        [this.pq[currentIndex],this.pq[rightIndex]] = [this.pq[rightIndex],this.pq[currentIndex]]
        currentIndex = rightIndex;
      } else {
        [this.pq[currentIndex],this.pq[leftIndex]] = [this.pq[leftIndex],this.pq[currentIndex]]
        currentIndex = leftIndex;
      }
      leftIndex = currentIndex  * 2;
      rightIndex = leftIndex + 1;
      // left, right, current를 다시 정하고 이를 반복한다.
    }
    return returnValue;
  }
}
```

## 결론

python의 `heapq`가 얼마나 좋은 것인지 알게되었다고 해야할까.

아무튼, 직접 이를 타이핑하면서 익힐 수 있어 좋았다.
