---
layout: post
title: '지하철 안내 DB구조 개편'
date: 2025-11-08 13:50 +0800
logs: [WishEasy]
toc: True
---
## Why?
기존에는 간선(Edge) 과 정점(Node) 테이블 모두에 역명(station)과 호선(line) 컬럼이 포함되어 있었고,
또한 Edge 테이블에는 relation(안내문)을 미리 만들어 저장하는 방식으로 관리하고 있었다.

하지만 이런 구조는 데이터의 **중복성과 확장성 측면에서 비효율적**이다!!

## 1. 기존 구조의 문제점

기존(Before) DB 구조에서는 각 테이블이 “자기 객체 정보 + 다른 객체 정보까지 다 섞여 있는” 형태

- **Nodes** 테이블 안에 `역명(station)`, `호선(line)`까지 같이 들어있고
- **Edge** 테이블 안에도 `역명(station)`이 또 들어가 있고
- 보조 테이블로 역–호선 정보를 따로도 관리함
    
    ![before_db.png]({{ '/assets/images/Relational-table/before_db.png' | relative_url }})
    

이렇게 되면서: 

1. **역/호선 데이터가 중복 저장**
    - 같은 역 이름이 Nodes, Edge, lines등 여러 곳에 반복 저장
    - 역 명을 관리하기 어려워짐(수정할 일은 별로 없겠지만..)
2. **새 역/호선 추가 시 작업이 번거로움**
    - CSV에서 노드/엣지/호선 정보를 먼저 만들고
    - 각 테이블에 일일이 relation을 생성해서 넣어줘야 함
3. **정규화가 안 되어 관계형 DB의 장점을 못 씀**
    - 역이라는 개념, 호선이라는 개념이 명확히 분리되지 않고
    - 참조 무결성 관리가 어려움

그래서 **객체별로 테이블을 분리하고**, 역–호선–노드–엣지의 관계를 명확하게 표현하는 방향으로 구조를 재설계했다.

## 2. 구조 재설계

새 구조는 **객체별 테이블 분리 + 관계 중심 설계**를 목표로 했다.

![after_db.png]({{ '/assets/images/Relational-table/after_db.png' | relative_url }})

### 2.1 Station

```
Station
- id      : 지하철 역의 고유 코드 (PK)
- name    : 실제 지하철 역명 (한글)
```

- 역에 대한 단일 소스
- 다른 테이블들은 모두 `station_id` 로만 역을 참조한다.

### 2.2 Line

```
Line
- id      : 지하철 호선 PK
- name    : 지하철 호선명 (예: 2호선, 7호선)
```

- 호선에 대한 단일 소스

### 2.3 StationLine (역 - 호선 교차 테이블, M:N 관계를 위해)

```
StationLine
- station_id : FK → Station.id
- line_id    : FK → Line.id
(복합 PK)
```

- 한 역에 여러 호선이 지나는 환승역 표현 가능
- 한 호선에 여러 역이 속하는 표현 가능

### 2.4 Node

```
Node
- id         : 노드 고유 id (정점 코드)
- name       : 노드 이름 (예: 2번출구, 2호선 승강장 등)
- floor      : 층 정보 (지하1층, 지상1층 등)
- type       : 타입 (출구, 승강장, 대합실 등)
- station_id : FK → Station.id (1:N 관계)
```

- 각 노드는 반드시 하나의 역에 속함
- 역 이름은 FK로 JOIN으로 가져올 수 있음

### 2.5 Edge

```
Edge
- id          : 간선 고유 id
- source_node : FK → Node.id (시작 노드)
- target_node : FK → Node.id (도착 노드)
- escalator   : 에스컬레이터 유무
```

- 그래프의 연결 관계만 표현
- 어느 역의 간선인지 알고 싶으면 source_node → Node.station_id 를 통해 역 정보에 접근할 수 있음

### 2.6 FastGate

```
FastGate
- boarding_gate : 탑승 출입문 위치 번호
- escalator     : 하차 시 에스컬레이터 이용 가능 여부
- station_id    : FK → Station.id
- line_id       : FK → Line.id
( station_id + line_id 가 복합 PK )
```

- 하차역을 기준으로 어디서 하차해아 빠른지 정보를 저장
- 사용 시점에는:
    - get_subway_route 결과에서 **하차역(station_name)** 과 **해당 호선(line_name)** 을 알고 있으므로
    - `Station.name`, `Line.name`으로 각각 `id`를 찾고
    - `(station_id, line_id)` 로 FastGate 레코드를 조회해서 탑승구 안내 메시지를 만든다

## 3. 개편한 구조의 장점

### 3.1 데이터 추가/수정 시 작업량 감소

- **새 역 추가**
    1. `Station`에 역 한 줄 추가
    2. 해당 역이 속한 호선들을 `StationLine`에 추가
    3. 그 역의 노드/엣지(Graph)는 Node/Edge에만 넣으면 끝
- 이 테이블 구조만 있으면 guide로직에서 relation을 만들어 제공하는 구조.

### 3.2 정규화로 인해 무결성 및 관리 용이

- 역 이름을 바꾸고 싶으면 `Station.name` 한 곳만 수정
    
    → Node, Edge, FastGate는 전부 `station_id`로만 참조하고 있기 때문에 영향 최소화
    
- 호선 이름/색상 등을 추가로 관리할 때도 `Line` 테이블만 확장하면 됨
- 역–호선 매핑이 `StationLine`에만 있으므로
    
    → “이 역을 지나는 호선 리스트” / “이 호선에 포함된 역들”을 쿼리로 쉽게 구할 수 있음
    

### 3.3 그래프 구조와 비즈니스 로직 분리

- **Node/Edge**는 “공간 그래프”만 표현 (계단/에스컬레이터, 통로 등)
- **Station/Line/StationLine**은 “철도 정보(역·호선)”만 표현
- **FastGate**는 “여정 중 하차 최적 탑승구”라는 별도 도메인 로직만 담당

이렇게 역할을 나눔으로써:

- 지하철 구조가 바뀌어도 (예: 엘리베이터 추가) Node/Edge만 수정하면 되고
- 역/호선 리뉴얼은 Station/Line/StationLine 중심으로 수정 가능
- 탑승구 추천 로직은 FastGate 만 보면서 고도화 가능

## 4. 로직에서 변경한 DB 구조 사용 흐름

1. **지상 출발 → 승강장까지 안내**
    - get_subway_route 결과:
        
        `('신촌', '2번출구', '2호선 이대 방면 승강장', ('2호선', '건대입구'))`
        
    - `df_nodes/df_edges` 에서 해당 역(`station='신촌'`)의 Node/Edge만 뽑아 BFS
    - Node.type 이 `'승강장'` 이면 FastGate를 이용해 탑승구 안내 추가
2. **FastGate 검색**
    - 하차역: `'건대입구'`, 호선: `'2호선'`
    - `Station(name='건대입구') → station_id`
    - `Line(name='2호선') → line_id`
    - `FastGate(station_id, line_id)` 조회
    - `boarding_gate` / `escalator` 조건에 맞게 안내문 생성

끝.