---
layout: post
title: '최소 환승 경로에서 순환선'
date: 2025-11-07 17:34 +0800
logs: [WishEasy]
toc: True
---
## 문제 상황
- 기존 최소 환승 경로를 다익스트라로 구현한 기능에서 기본 그래프 구조가 역 간 직선 연결로 되어있었다.
- 그러면 2호선 같은 순환선에서 끊키는 구간이 생기게 된다.
- 그럼 시청에서 아현으로 가려면 충정로 방면으로 탐색해야하지만 반대의 방면이 출력되는 문제가 있었다.

## 해결 원리
### 순환선 연결
- 직선 연결로 되어있던 2호선의 첫번째 역과 마지막 역을 직접 연결했다

```python
if line_name == "2호선":
    first_node = f"{stations[0]}-{line_name}"
    last_node  = f"{stations[-1]}-{line_name}"
    G.add_edge(first_node, last_node, weight=0)
```

```
이렇게 하면 그래프 탐색 시 2호선이 원형 루프로 인식됨!
```

끝
