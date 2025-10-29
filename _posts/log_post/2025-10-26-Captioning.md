---
layout: post
title: 'Captioning'
date: 2025-10-26 23:03 +0800
logs: [AI Challenge]
toc: True
---
## Image Captioning 적용

<br>

### Image Captioning 이란?
이미지 안에 있는 시각적 정보를 기반으로 사람이 이해할 수 있는 자연어 문장을 자동으로 생성하는 기술.\

**예시**

![기차]({{ '/assets/images/captioning/기차.png' | relative_url }})
**'A blue train is passing through a street.'**

<br>

## 목적
현재 모델의 구조는 이미지와 간단한 프롬프트 만 input으로 들어가고 있다.

그래서 Cationing 모델로 추출한 이미지에 대한 정보를 프롬프트에 넣어 추론 정확도를 올리고자 시도했다.

<br>

## 기대효과
| 개선 포인트               | 기대되는 성능 향상 이유            |
| -------------------- | ------------------------ |
| 이미지-텍스트 정합성 증가       | 질문에서 언급된 객체/상황을 더 정확히 식별 |
| 잡음/모호성 감소            | 불완전한 질문 맥락 보완            |
| 소형 VLM의 시각 인식 보조     | 이미지 의미를 언어로 정리해 추론에 도움   |

<br>

## 적용한 모델
`Salesforce BLIP Large`
```python
CAPTION_MODEL_ID = "Salesforce/blip-image-captioning-large"
caption_image(...) 
```
현재 컴퓨팅 자원을 고려했을때, 적당히 가벼우면서 캡션 품질/ 성능이 안정적인 모델을 기준으로 하였다!

<br>

## 캡셔닝 적용 결과와 한계 정리
### 결과
![1]({{ '/assets/images/captioning/캡셔닝1이미지.png' | relative_url }})
**This picture seems like snowy landscape of a city with a bridge and a park.**

![2]({{ '/assets/images/captioning/캡셔닝2이미지.png' | relative_url }})
**This picture seems like there is a field with a building in the background.**

<br>

**하지만,,**

위 결과를 직접 적용하여 결과물을 뽑아보니 예상과 달리 오히려 소폭 감소하는 현상을 보여주었다.. 

<br>

위 결과의 각각의 질문과 선지는 다음과 같다.

<br>

**첫번째Q**: 이 사진 속 운동기구가 설치된 장소는 어디일까요?
- a: 학교 운동장
- b: 공원
- c: 헬스장 내부
- d: 쇼핑몰 내부
  
**두번째Q**: 이 사진에 보이는 전통 한국 건축물은 무엇인가요?
- a: 궁궐
- b: 성
- c: 사찰
- d: 한옥

### 원인 분석

`1. 정보 비정합`\
선지는 폐쇄형 카테고리(공원/사찰/한옥 등) 로, 정답을 가르는 핵심 특징(기둥/지붕 형태, 안내표지, 실내/실외, 시설물 배치 등)이 필요.

캡션은 장면 전반 요약에 치우쳐 결정적 단서가 빠지거나 희석됨 → 추가 토큰만 늘고 신호대잡음비(SNR) 악화.

`2. 앵커링(Anchoring) 유발`

캡션이 “park, bridge” 같은 단어를 던지면 모델이 해당 개념에 과도하게 고정 → 실제 선지 맥락과 충돌 시 오답 유도.

`3. 태스크 조건 미반영`

캡셔닝은 질문 비조건(question-agnostic) 이라, “무엇을 구분해야 하는가”가 반영되지 않음 → 정답에 필요한 관찰 포인트를 못 찍음.

<br>

## 결론
단순히 이미지만 주고 캡셔닝을 진행한 결과 질문과 연관성이 적거나 오히려 맥락을 흐리는 결과가 생긴것으로 판단된다.

때문에 이 방법의 효과를 보기 위해선 "질문 조건부" 캡션으로 적용하여 질문에 맥락을 반영하는 등 Task를 구분하고 다양한 실험이 필요해 보인다!