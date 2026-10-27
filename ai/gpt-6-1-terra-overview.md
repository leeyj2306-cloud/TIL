# GPT-6.1 Terra: 저가·고속 컴퓨터 사용 모델 (OpenAI)

OpenAI가 공개한 GPT-6.1 Terra는 시각 추론과 컴퓨터 사용(computer use)에 강점이 있는 빠르고 저렴한 모델이다. 성능·가격·속도 중 어디에 쓸지 고를 때 기준이 되는 수치를 정리했다. 아래 수치는 모두 OpenAI가 자체 벤치마크로 발표한 값이다.

## 가격과 사용 방법
- API 모델명은 `gpt-6.1-terra`다. 표준 가격은 입력 $1, 캐시 입력 $0.05, 출력 $5 (100만 토큰당).
- 발표에 따르면 GPT-5.6 Terra, GPT-6.1 Sol보다 표준 API 가격이 50% 낮다.
- ChatGPT Work와 Codex에서는 Plus, Pro, Enterprise, Edu 사용자가 쓸 수 있다.

```python
from openai import OpenAI
client = OpenAI()
resp = client.responses.create(model="gpt-6.1-terra", input="...")  # (보충) 호출 형태는 기존 SDK 기준 예시
```

## 컴퓨터 사용·시각 추론
- 전문 업무 벤치마크 Agents' Last Exam에서 중간 추론 강도 기준 GPT-5.6 Terra보다 15%p 높고, 비용은 약 1/3이다.
- 작업 완료 속도는 GPT-6 Astra, GPT-6.1 Sol보다 3배 넘게 빠르다고 한다.
- 송장을 읽어 회계 프로그램에 입력하는 식의 대량 반복 업무가 주요 대상이다.

## 사실성과 코딩
- 오류 유도 프롬프트 내부 벤치마크에서 낮은 추론 강도 기준 오류율이 53.6% → 16.7%로, 최대 강도에서는 4.3%로 낮아졌다.
- Terminal-Bench 4.0(최대 강도)은 50.3%로 GPT-5.6 Terra의 21.5%보다 두 배 이상이다.
- 가장 복잡한 코딩은 여전히 GPT-6.1 Sol, GPT-6 Astra가 낫다고 OpenAI도 밝혔다.

## 내 작업에 적용
- simAI처럼 AI API를 설계할 때, 대량·반복 호출 구간은 저가 모델로 두고 어려운 추론만 상위 모델로 보내는 구성을 비교해 볼 수 있다. 단, 자체 발표 수치이므로 우리 데이터로 직접 평가한 뒤 결정한다.

## 참고
- [Introducing GPT-6.1 Terra](https://openai.com/index/introducing-gpt-6-1-terra/)
