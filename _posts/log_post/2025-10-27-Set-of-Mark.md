---
layout: post
title: 'API로 데이터를 불러와보자'
date: 2025-10-27 16:32 +0800
logs: [AI Challenge]
toc: True
---
# VLM 성능을 높이는 트릭: Set-of-Mark
모델을 바꿔보고, 파라미터를 건들여보면서 어느정도 성능 향상의 정체기가 왔다.\
Valid를 예측하고 어떤 문제를 틀리는가를 확인해보면,,, **사실 사람이 풀어도 틀릴 가능성이 있는 문제**가 대부분이었다.

그런데 아래 예시는 좀 아쉬웠다.\
![틀린예시]({{ '/assets/images/valid_wrong.png' | relative_url }})

**이 정돈 맞춰줘야하는거 아닌가?** 라는 생각이 들었고, 모델에 어떤 `힌트`를 줄 필요가 있겠다라는 생각이 들었다.


그러다가 AI 강의에서 VLM 성능을 향상시키는 트릭 중 하나로 Set-of-Mark라는 전처리 과정을 배웠던 것이 기억이나서 적용해보았다.

## What is SOM(Set of Mark)?
별도의 파운데이션 모델을 이용해 이미지 내 각 객체를 분리하고,
그 객체 ID를 이미지 위에 오버레이하는 전처리 과정\
참고 리포지토리: [Microsoft SoM](https://github.com/microsoft/SoM)
- 비슷한 객체가 여러 개 존재하거나
- 객체 크기가 작거나
- 복잡한 배경으로 인해 인식이 어려운 경우
  
VLM이 해당 객체를 정확히 구분하지 못하는 문제를 완화하고 결과적으로 **모델의 시각적 인식 성능을 높여주는 기법**으로 알려져있다.

### 대표적인 Model
- `SEEM`
- `Semantic-SAM`
- `SAM`
- `MaskDINO` 
이 중에서 `SAM2` 모델을 활용하여 Set-of-Mark 를 진행해보았다.


### 모델 import
```python
!pip install -U "ultralytics>=8.2.70" opencv-python pillow tqdm --quiet
import ultralytics, torch, sys, platform
from ultralytics import SAM
...

model = SAM("sam2_t.pt")
```

## Output
<p align="center">
  <img src="{{ '/assets/images/som예시2.png' | relative_url }}" width="45%" />
  <img src="{{ '/assets/images/som예시1.png' | relative_url }}" width="45%" />
</p>

- Public 점수를 확인한 결과, 성능 향상은 아주 미세한 수준에 그쳤다.
- 그 원인으로는, 픽셀 단위의 정교한 마스크 생성이 핵심인데... 현재 결과는 단순히 숫자만 오버레이된 상태임을 들 수 있겠다
  
>이를 개선해보고자 했지만, 해당 데이터를 구축하는데만 5시간 이상이 소요되었고, 컴퓨팅 자원의 한계로 더 디벨롭하지 못한 점이 아쉽게 남는다.\
세그멘테이션을 추가해주었다면 실제로 더 큰 향상이 있지 않을까 생각이 든다.

## `+@`세부 세그멘테이션 까지 5개만 추가 적용해보았다.
- 프로젝트는 끝이 났지만,, 아쉬움으로 남아 일부 데이터로만 적용해보았다\
![1]({{ '/assets/images/som보완.png' | relative_url }})
![2]({{ '/assets/images/som보완2.png' | relative_url }})
![3]({{ '/assets/images/som보완3.png' | relative_url }})
![4]({{ '/assets/images/som보완4.png' | relative_url }})
![5]({{ '/assets/images/som보완5.png' | relative_url }})

조금 아쉽지만 확실히 모델에게 힌트를 줄 수 있을 것 같다는 느낌이 든다.

끝.