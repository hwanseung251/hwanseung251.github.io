---
layout: post
title: '최소 환승 경로에서 순환선'
date: 2025-11-07 17:34 +0800
logs: [WishEasy]
toc: True
---
## 하차역으로 이동할 때 왜 방면을 알아야 할까?
구현한 로직에서 **어디 방면 승강장에서 탑승해야 하는지 안내를 하기 위해서** 현재역 기준으로 어디 방면(바로 앞 혹은 바로 뒤 역)으로 가야할지 알아야 한다.

## 문제 상황
- 기존 최소 환승 경로를 다익스트라로 구현한 기능에서 기본 그래프 구조가 역 간 직선 연결로 되어있었다.
- 그러면 2호선 같은 순환선에서 끊키는 구간이 생기게 된다.
- 그럼 시청에서 아현으로 가려면 충정로 방면으로 탐색해야하지만 반대의 방면으로 값이 나오는 문제가 있다.

## 해결 방안
### 순환선 연결구조로 이어보자
- 직선 연결로 되어있던 2호선의 첫번째 역과 마지막 역을 직접 연결하여 순환선 그래프 구조를 만들었다.

```python
if line_name in RING_LINES and len(stations) > 1:
    first_node = f"{stations[0]}-{line_name}"
    last_node = f"{stations[-1]}-{line_name}"
    G.add_edge(first_node, last_node, weight=0)

```

```
이렇게 하면 그래프 탐색 시 2호선이 원형 루프로 인식됨!
```

### build_graph 전체 구조 
```python
def build_graph():
    """DB 데이터를 기반으로 최소환승 그래프 구성"""
    G = nx.Graph()

    # 1. DB에서 노선별 역 순서 dict 생성
    qs = Lines.objects.values("line", "station", "order_in_line")
    rows = sorted(list(qs), key=lambda r: (r["line"], r["order_in_line"]))

    line_data = {}  # {"2호선": ["시청", "을지로입구", ...], ...}
    for r in rows:
        line_data.setdefault(r["line"], []).append(r["station"])

    # 2. 호선별 인접역 연결 + 순환선 처리
    for line_name, stations in line_data.items():
        # 인접역
        for i, station in enumerate(stations):
            node = f"{station}-{line_name}"
            G.add_node(node)
            if i > 0:
                prev_node = f"{stations[i-1]}-{line_name}"
                G.add_edge(prev_node, node, weight=0)  # 인접역

        # 순환선이면 첫 역 ↔ 마지막 역도 연결
        if line_name in RING_LINES and len(stations) > 1:
            first_node = f"{stations[0]}-{line_name}"
            last_node = f"{stations[-1]}-{line_name}"
            G.add_edge(first_node, last_node, weight=0)

    # 3. 환승 연결 (역 이름 동일, 호선 다름)
    all_nodes = list(G.nodes)
    for i, n1 in enumerate(all_nodes):
        name1, line1 = n1.split("-")
        for j in range(i + 1, len(all_nodes)):
            n2 = all_nodes[j]
            name2, line2 = n2.split("-")
            if name1 == name2 and line1 != line2:
                G.add_edge(n1, n2, weight=1)  # 환승 비용 1

    print("✅ Subway graph built successfully.")
    return G
```

### 순환선으로 방면 계산하려면??
먼저 호출되는 `get_legs(path)`가 legs를 반환한다.
```python
legs = [
    ("2호선", ["신촌", "이대", "아현"], [idx0, idx1, idx2]),
    # 다음 leg...
]
```

### compute_leg_orientation 
```python
def compute_leg_orientation(
    line_name: str,
    stations: List[str],
    line_data: dict[str, list[str]],
) -> int:
```
반환된 legs는 `compute_leg_orientation`의 인자로 들어온다.
- 튜플의 첫 번째 원소 = `line_name`
- 두 번째 원소 = `stations`
- 세 번째 원소는 해당 호선 전체 라인 순서 dict

```python
    line_list = line_data[line_name]
    n = len(line_list)
    if len(stations) >= 2:
        a, b = stations[0], stations[1]
        ia, ib = line_list.index(a), line_list.index(b)
        if line_name in RING_LINES:
            # 순환선: 모듈러 연산으로 한 칸 차이 확인
            if (ib - ia) % n == 1:
                return 1
            elif (ia - ib) % n == 1:
                return -1
        else:
            # 비순환선: 단순 index 차이
            if ib - ia == 1:
                return 1
            elif ib - ia == -1:
                return -1
    # 기본값: 정방향
    return 1
```
- station의 첫 번째 두 번쨰역의 인덱스를 비교해서 방향을 구한다.
- 이 때, 
    ```python
        if (ib - ia) % n == 1:
            return 1
        elif (ia - ib) % n == 1:
            return -1
    ```
    순환 연결고리를 고려해서 끝 -> 시작의 경우 한 칸 이동으로 방향을 계산할 수 있도록 한다.

때문에 올바른 방면으로 안내가 가능해진다!

끝
