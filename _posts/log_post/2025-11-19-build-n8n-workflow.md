# 깡통전세 위험여부 판단해주고 Apache로 시각화해주는 n8n workflow 만들기

## 전체적인 구조
사용자의 요청 -> api등의 처리 -> 결과 반환 

## 1. 사용자 입력
- https://nocodehackerthon.netlify.app/ 이 앱에서
- 전체 주소
- 주택 유형 
- 전세 보증금
- (전월세라면) 보증금 + 월세
- 사용자선택으로 등기부등록본도 업로드
- 위 정보 입력하고 document.querySelector("#radix-\\:ri\\: > div.text-card-foreground.flex.flex-col.gap-6.rounded-xl.border.p-6.sm\\:p-8.bg-gray-50.border-gray-200 > div > button") 이거 누르면 workflow 시작(트리거)
## 2. 국토교통부_실거래가 정보
- 사용자가 조회를 원하는 집주소를 바탕으로 실거래가격을 알아냄
- 이 데이터는 api는 아니고 csv로 다운받는 구조인거같음
## 3. 등기부등록본 업로드했다면
- OCR을 활용해 근저당권, 압류/가압류/ 전세권 설정/ 임차권 등기 파악해서 깡통전세 위험계산에 포함
## 4. 깡통전세 위험도 계산 로직
- 위 정보를 기반으로 깡통주택 위험도 0~100%로 나누고 안전/주의/위험 등급까지 나눔
## 5. 아파치 시각화
- 게이지 시각화와 시세와 보증금 바차트 등 사용자가 입력한 정보와 계산한 정보를 시각화까지 해서 보여줌


### Webhook 트리거 
1. Post 요청
2. Path 설정(나중에 요청보낼 주소)
3. data