## 프로젝트 개요

카페 운영에 필요한 주문, 장바구니 관리, 고객 서비스 기능을 AI 에이전트들이 자동으로 처리하는 멀티 에이전트 기반의 카페 고객 서비스 시스템입니다. OpenAI GPT 모델과 스트리밍 응답을 활용하여 자연스러운 실시간 대화형 서비스를 제공하며, **음성 입력(STT) 및 음성 출력(TTS)** 기능을 통해 더욱 자연스러운 대화 경험을 제공합니다.

## 시스템 아키텍처

### 멀티 에이전트 구조

```
사용자 입력
    |
Triage Agent (분류 에이전트)
    |-- Order Agent (주문 에이전트) -> 장바구니/멤버십 도구
    |-- Service Agent (서비스 에이전트)
    |-- Conversation Agent (대화 에이전트) -> 날씨 도구

```

### 핵심 아키텍처 특징

- **비동기 스트리밍 처리**: 실시간 응답 생성 및 표시
- **도구 통합 (Function Calling)**: OpenAI Function Calling을 활용한 도구 실행
- **상태 관리**: 세션 기반 대화 기록 및 장바구니 상태 유지
- **모듈화 설계**: 에이전트, 매니저, UI, LLM 클라이언트 분리
- **음성 처리**: OpenAI Transcribe STT + Silero VAD, OpenAI TTS 통합
- **Always-Listening 모드**: 연속적인 음성 대화 지원

### 에이전트 상세 구현

### 1. **Triage Agent (분류 에이전트)**

- **모델**: gpt-4.1-mini
- **구현**: `cafe_agents/core/agents/triage.py`

### 정체성 및 핵심 목표

- 고객의 핵심 의도를 정확하게 파악하여 가장 적합한 전문 에이전트에게 작업을 위임
- 고객 요청을 가장 효율적이고 안전한 경로로 안내하여 빠르고 원활한 서비스 경험 제공

### 의도 분류 카테고리

1. **"order"**: 메뉴 문의, 주문, 장바구니 관련 등 구매와 관련된 모든 발화
2. **"service"**: 매장 시설, 영업시간, 와이파이, 화장실 위치 등 매장 서비스 관련 문의
3. **"conversation"**: 인사, 날씨, 농담 등 뚜렷한 목적이 없는 일상 대화

### 예외 처리 로직

- **복합적 의도**: 여러 의도가 섞인 경우 즉시 해결이 필요한 구체적 요청을 우선 처리
- **불분명한 의도**: 분류 불가능한 경우 "어떤 도움이 필요하신가요?" 같은 명료한 질문으로 의도 파악
- **정체성 질문**: "너는 누구니?" 같은 질문에는 직접 답변하지 않고 서비스로 유도

### 2. **Order Agent (주문 에이전트)**

- **모델**: gpt-4.1
- **구현**: `cafe_agents/core/agents/order.py`
- **도구 (Tools)**: 총 7개
  - 장바구니 도구 (3개):
    - `add_to_cart(menu_item, quantity, options)`: 장바구니 추가
    - `remove_from_cart(menu_item, options, quantity)`: 장바구니 제거
    - `view_cart()`: 장바구니 조회
  - 멤버십 도구 (4개):
    - `search_membership(query)`: 멤버십 정보 조회
    - `use_points(number, name, points)`: 포인트 사용
    - `add_points(number, name, points)`: 포인트 추가
    - `create_membership(number, name)`: 멤버십 생성

### 정체성 및 핵심 목표

- 전문적이고 숙련된 캐셔 역할
- 빠르고, 정확하며, 친절한 주문 경험 제공
- 모든 주문 단계에서 내용 재확인으로 실수 방지
- 고객이 망설일 경우 인기 메뉴 추천
- 필수 옵션 누락 시 반드시 먼저 질문

### 표준 주문 절차 (Standard Ordering Procedure)

### 1단계: 시작 (Initiate)

- "주문을 도와드릴게요"와 같이 즉시 역할 명확화
- 고객이 메뉴를 고민하는 경우 요청에 맞는 메뉴 2~3가지 추천
- 카테고리가 없는 경우 커피와 디저트 각각 추천

### 2단계: 주문 접수 및 옵션 확인

- 메뉴 선택 시 필수 옵션(Small/Regular/Large, HOT/ICED) 확인
- 옵션 미언급 시 "사이즈는 어떻게 해드릴까요?", "HOT으로 드릴까요, ICED로 드릴까요?" 명확히 질문
- 모든 메뉴에 대해 모든 옵션 확인 필수
- 필수 옵션 확정 후 `add_to_cart()` 호출

### 3단계: 추가 주문 확인

- "더 필요하신 것 있으신가요?" 물어 추가 주문 확인
- 추가 주문 시 2단계 반복
- 주문 완료 시 다음 단계 진행

### 4단계: 최종 주문 확인 (절대 생략 불가)

- `view_cart()` 호출하여 전체 주문 내역 확인
- "주문 확인해 드릴게요. [메뉴명, 옵션, 수량] 총 N개 맞으신가요?" 형식으로 확인

### 5단계: 멤버십 및 할인 적용

- "멤버십 있으시면 포인트 적립이나 사용을 도와드릴까요?"
- 전화번호 뒷자리 4자리로 멤버십 조회
- 포인트 사용/적립 처리

### 6단계: 주문 완료

- "최종 금액은 OOO원입니다. 주문번호는 OOO번입니다. 맛있게 준비해 드릴게요!"

### 예외 처리 규칙

- **주문 변경/취소**: `remove_from_cart`로 기존 메뉴 제거 후 `add_to_cart`로 새 메뉴 추가
- **멤버십 오류**: 조회 실패 시 번호 재확인 요청, 포인트 부족 시 잔액 안내
- **모호한 요청**: "알아서 해주세요" 같은 요청에는 구체적 질문으로 명확화
- **시스템 오류**: 도구 실패 시 "시스템 오류 발생, 직원에게 문의" 안내

### 3. **Service Agent (서비스 에이전트)**

- **모델**: gpt-4.1-mini
- **구현**: `cafe_agents/core/agents/service.py`

### 정체성 및 핵심 역할

- 메뉴 주문과 일상 대화 외 매장 관련 다양한 요청사항 처리
- 매장 정보 안내 전문가 역할

### 처리 가능한 문의

- 와이파이 비밀번호
- 화장실 위치
- 영업시간
- 매장 시설 정보
- 기타 서비스 관련 문의

### 정보 제공 방식

- `cafe_info.json` 파일의 매장 정보 참조하여 정확한 답변 제공
- 파일에 없는 정보는 "죄송합니다. 알지 못하는 정보입니다" 응답

### 4. **Conversation Agent (대화 에이전트)**

- **모델**: gpt-4.1-mini
- **구현**: `cafe_agents/core/agents/conversation.py`
- **도구 (Tools)**:
  - `weather_search(date, city)`: 날씨 정보 조회

### 정체성 및 핵심 목표

- 주문/매장과 무관한 일상 대화로 친근한 경험 제공
- 카페 직원 정체성 유지하며 대화
- 모든 대화의 결론은 카페 주문으로 자연스럽게 유도
- 긍정적이고 친절하며 약간의 위트 포함
- 간결한 답변 유지, 고객을 가르치려 하지 않음

### 대화 지침

### 권장 대화 주제

- **날씨 대화**: `weather_search()` 활용
  - date: "today", "tomorrow", "YYYY-MM-DD" 형식
  - city: 도시명 (기본값: "Gwangju")
- **가벼운 칭찬**: "오늘 날씨랑 잘 어울리는 옷이네요!"
- **긍정적 농담**: 간단하고 밝은 분위기의 농담

### 금지 대화 주제 (매우 중요)

- 정치, 종교, 사회적 갈등 등 민감한 주제 절대 금지
- 금융, 의료 등 전문적 조언 제공 금지
- 고객 개인정보 질문 금지
- AI 정체성에 대한 장황한 설명 금지

## 프로젝트 구조

```
cafe_agents/
├── app.py                  # Streamlit 메인 애플리케이션 (진입점)
├── cafe_agents/
│   ├── __init__.py        # 패키지 초기화
│   ├── core/              # 핵심 비즈니스 로직
│   │   ├── __init__.py
│   │   ├── cafe_system.py # CafeSystem 핵심 클래스 (TTS 통합)
│   │   ├── agents/        # AI 에이전트 구현
│   │   │   ├── __init__.py
│   │   │   ├── base.py    # BaseAgent 추상 클래스
│   │   │   ├── triage.py  # 분류 에이전트
│   │   │   ├── order.py   # 주문 에이전트
│   │   │   ├── service.py # 서비스 에이전트
│   │   │   └── conversation.py # 대화 에이전트
│   │   ├── llm/           # LLM 클라이언트
│   │   │   ├── __init__.py
│   │   │   └── client.py  # OpenAI API 클라이언트
│   │   └── tools/         # 도구 정의 및 실행
│   │       ├── __init__.py
│   │       ├── schemas.py # OpenAI 도구 스키마
│   │       └── weather.py # 날씨 도구 구현
│   ├── managers/          # 데이터 관리 모듈
│   │   ├── __init__.py
│   │   ├── cart_manager.py        # 장바구니 관리
│   │   ├── membership_manager.py  # 멤버십 CRUD 관리
│   │   └── menu_manager.py        # 메뉴 데이터 관리
│   ├── stt/               # 음성 인식 (Speech-to-Text)
│   │   ├── __init__.py
│   │   ├── base.py        # STT 엔진 추상 클래스
│   │   ├── config.py      # STT 설정 상수
│   │   ├── openai_transcribe.py  # OpenAI Transcribe STT
│   │   └── silero_vad.py  # Silero VAD 음성 감지
│   ├── tts/               # 음성 합성 (Text-to-Speech)
│   │   ├── __init__.py
│   │   ├── base.py        # TTS 엔진 추상 클래스
│   │   ├── config.py      # TTS 설정 상수
│   │   └── openai_tts.py  # OpenAI TTS 구현
│   ├── ui/                # UI 컴포넌트
│   │   ├── __init__.py
│   │   ├── styles.py      # CSS 스타일 정의
│   │   ├── sidebar.py     # 사이드바 컴포넌트 (음성 설정)
│   │   └── chat.py        # 채팅 인터페이스 (STT/TTS 통합)
│   ├── config/            # 설정 및 프롬프트
│   │   ├── __init__.py
│   │   └── agents_prompts.py  # 에이전트 지침 설정
│   └── data/              # 데이터 파일
│       ├── menu.json      # 메뉴 데이터베이스
│       ├── membership.json # 멤버십 데이터베이스
│       └── cafe_info.json # 카페 정보 데이터
├── pyproject.toml         # 프로젝트 의존성 정의
└── .env                   # 환경 변수 (API 키)

```

## 주요 기능

### 1. 실시간 스트리밍 AI 서비스

- **비동기 스트리밍**: OpenAI Streaming API를 통한 실시간 응답 생성
- **Function Calling**: OpenAI Function Calling을 활용한 도구 실행
- **멀티 에이전트**: 트리아지 → 전문 에이전트 라우팅
- **컨텍스트 관리**: 장바구니 도구 사용 시 LLM 메시지 초기화로 효율적 메모리 관리

### 2. 장바구니 관리

- **아이템 추가/제거**: 메뉴 유효성 검증 및 옵션 확인
- **필수 옵션 검증**: 온도(HOT/ICED), 사이즈(Small/Regular/Large) 자동 확인
- **옵션 정규화**: 동일 메뉴+옵션 자동 그룹핑
- **수량 관리**: 기존 아이템 수량 증가/감소 처리
- **요약 제공**: 장바구니 내용 및 총액 실시간 계산

### 3. 메뉴 관리

- **JSON 기반 관리**: 메뉴 데이터 동적 로드
- **메뉴 검증**: 필수 필드 및 데이터 타입 검증
- **카테고리별 조회**: 커피, 논커피, 디저트 등
- **옵션 정보**: 메뉴별 필수 옵션 정의

### 4. 멤버십 관리

- **CRUD 작업**: 생성, 조회, 수정, 삭제
- **포인트 시스템**: 적립/사용 (10% 자동 적립)
- **JSON 저장**: 파일 기반 영구 저장
- **검색 기능**: 전화번호 또는 이름으로 검색

### 5. 음성 입력 (STT - Speech-to-Text)

- **OpenAI Transcribe API (gpt-4o-mini-transcribe)**: 높은 정확도의 한국어 음성 인식
- **Silero VAD**: 실시간 음성 활동 감지 (Voice Activity Detection)
- **Always-Listening 모드**: 연속적인 음성 대화 지원
- **설정 가능한 파라미터**:
  - VAD 임계값 (0.0~1.0)
  - 최소 음성 길이 (250ms)
  - 침묵 감지 시간 (1000ms)
  - 최대 녹음 시간 (30초)

### 6. 음성 출력 (TTS - Text-to-Speech)

- **OpenAI TTS API**: 자연스러운 한국어 음성 합성
- **다양한 음성 선택**: alloy, echo, fable, onyx, nova, shimmer
- **음성 속도 조절**: 0.25~4.0배속
- **자동 재생**: AI 응답 자동 음성 출력
- **스트리밍 음성**: 실시간 음성 스트리밍 지원

### 7. 세션 관리

- **UUID 기반 ID**: 충돌 방지 세션 식별자
- **대화 기록 이원화**: UI용 전체 기록 + LLM용 최적화 기록
- **상태 추적**: 장바구니, 멤버십, 에이전트 상태

## 설치 및 실행

### 요구사항

- Python 3.12 이상
- OpenAI API 키
- 마이크 (음성 입력 사용 시)
- 스피커/헤드폰 (음성 출력 사용 시)

### 설치 방법

1.  저장소 클론

    ```bash
    git clone <https://github.com/SOSLAB-IVC/cafe_agents>
    cd cafe_agents

    ```

2.  가상환경 생성 및 활성화

    ```bash
    # 가상환경 생성
    python -m venv .venv

    # Windows
    .venv\\Scripts\\activate

    # Linux/Mac
    source .venv/bin/activate

    ```

3.  의존성 설치

    ```bash
    pip install -e .

    ```

4.  환경 변수 설정
    `.env` 파일을 생성하고 OpenAI API 키 추가:
        ```
        OPENAI_API_KEY=your-api-key-here

        ```
5.  실행

    ```bash
    streamlit run app.py

    ```

## 사용법

### 기본 대화 흐름

1. **세션 시작**: 좌측 사이드바에서 "세션 시작" 클릭
2. **대화 시작**:
   - **텍스트 입력**: 채팅창에 메시지 입력
   - **음성 입력**: 사이드바에서 "음성 입력 활성화" 후 마이크에 대고 말하기
3. **주문 예시**:
   - "아메리카노 한 잔 주세요"
   - "메뉴 추천해주세요"
   - "치즈케이크도 하나 추가할게요"

### 음성 기능 사용

### 음성 입력 (STT)

1. **활성화**: 사이드바 → "🎤 음성 입력" 체크
2. **자동 모드**: 활성화 시 자동으로 음성 감지 시작
3. **대화 방식**:
   - 말하기 시작하면 자동으로 녹음 시작
   - 1초 이상 침묵 시 자동으로 녹음 종료
   - 인식된 텍스트가 자동으로 전송됨

### 음성 출력 (TTS)

1. **활성화**: 사이드바 → "🔊 음성 출력" 체크
2. **음성 선택**: 6가지 음성 중 선택 가능
3. **속도 조절**: 0.25~4.0배속 조절 가능
4. **자동 재생**: AI 응답이 자동으로 음성 출력됨

### 멤버십 관리

1. **멤버십 생성**:
   - 사이드바 → 멤버십 생성 → 4자리 번호와 이름 입력
2. **멤버십 조회**:
   - 사이드바 → 멤버십 검색 → 번호 또는 이름으로 검색
3. **포인트 적립**:
   - 주문 시 멤버십 번호 제공하면 자동 10% 적립

### 메뉴 확인

- 사이드바 "메뉴 보기" 버튼 클릭
- 카테고리별 메뉴와 가격 확인 가능

## 핵심 클래스 설명

### CafeSystem (`cafe_agents/core/cafe_system.py`)

```python
class CafeSystem:
    """카페 에이전트 시스템 메인 클래스"""
    - __init__(): 매니저 초기화 및 도구 설정
    - start_session(): 새 세션 시작 (UUID 생성)
    - process_message_stream(): 비동기 스트리밍 메시지 처리
    - process_message_stream_sync(): 동기 스트리밍 (Streamlit용)
    - get_tts_engine(): TTS 엔진 지연 초기화
    - end_session(): 세션 종료 및 상태 초기화

    # 상태 관리
    - messages: UI 표시용 전체 대화 기록
    - llm_messages: LLM 전달용 최적화된 대화 기록
    - tts_engine: TTS 엔진 인스턴스 (지연 초기화)

```

### BaseAgent (`cafe_agents/core/agents/base.py`)

```python
class BaseAgent(ABC):
    """에이전트 베이스 클래스"""
    - prepare_messages(): 메시지 준비 및 시스템 프롬프트 추가
    - create_stream_response(): OpenAI 스트림 응답 생성
    - process_stream_with_tools(): 도구 호출 포함 스트림 처리
    - process(): 추상 메서드 (각 에이전트가 구현)

```

### CartManager (`cafe_agents/managers/cart_manager.py`)

```python
class CartManager:
    """장바구니 관리 클래스"""
    - add_item(): 메뉴 추가 (옵션 검증 포함)
    - remove_item(): 메뉴 제거 (수량 조절 가능)
    - get_cart_summary(): 장바구니 요약 문자열
    - _validate_required_options(): 필수 옵션 검증
    - _normalize_options(): 옵션 정규화 및 정렬

```

### LLMClient (`cafe_agents/core/llm/client.py`)

```python
class LLMClient:
    """OpenAI API 클라이언트 래퍼"""
    - create_completion(): 채팅 완성 생성
    - process_stream(): 스트림 응답 처리
    - collect_stream_response(): 스트림 응답 수집
    - extract_tool_calls(): 도구 호출 추출

```

### OpenAITranscribeSTT (`cafe_agents/stt/openai_transcribe.py`)

```python
class OpenAITranscribeSTT(STTEngine):
    """OpenAI Transcribe 음성 인식 엔진"""
    - __init__(): gpt-4o-mini-transcribe 모델 및 VAD 설정 초기화
    - listen(): Silero VAD로 음성 감지 후 OpenAI Transcribe로 인식
    - transcribe_audio(): 오디오 파일을 텍스트로 변환

    # VAD 설정
    - threshold: 음성 감지 임계값 (0.4)
    - min_speech_duration_ms: 최소 음성 길이 (250ms)
    - min_silence_duration_ms: 종료 판단 침묵 (1000ms)

```

### OpenAITTS (`cafe_agents/tts/openai_tts.py`)

```python
class OpenAITTS(TTSEngine):
    """OpenAI TTS 음성 합성 엔진"""
    - __init__(): TTS 모델 및 음성 설정
    - speak(): 텍스트를 음성으로 변환 및 재생
    - set_voice(): 음성 종류 변경
    - set_speed(): 재생 속도 조절
    - set_instructions(): 음성 효과 지침 설정

    # 지원 음성
    - alloy, echo, fable, onyx, nova, shimmer

```

## 데이터 흐름

### 스트리밍 처리 흐름

1. **사용자 입력**
   - **텍스트**: Streamlit chat_input
   - **음성**: STT → Silero VAD → OpenAI Transcribe
2. **동기 스트림 변환** → `CafeSystem.process_message_stream_sync()`
3. **비동기 스트림 처리** → `CafeSystem.process_message_stream()`
4. **의도 분류** → `TriageAgent.classify_intent()`
5. **에이전트 라우팅** → Order/Service/Conversation Agent
6. **도구 실행** → `BaseAgent.process_stream_with_tools()`
7. **스트림 청크 전송** → `yield {"type": "content", "data": ...}`
8. **UI 실시간 업데이트** → `st.write_stream()`
9. **음성 출력** (TTS 활성화 시) → OpenAI TTS API

### 도구 호출 흐름

1. **Function Calling 감지** → OpenAI API
2. **도구 인자 파싱** → JSON 변환
3. **도구 실행** → `execute_tool()`
4. **결과 반환** → 에이전트 메시지에 추가
5. **최종 응답 생성** → 도구 결과 기반 응답

## 데이터 구조

### menu.json

```json
{
  "커피": [
    {
      "name": "아메리카노",
      "price": 4500,
      "description": "깊고 진한 에스프레소..."
    }
  ]
}
```

### membership.json

```json
{
  "5678": {
    "name": "김철수",
    "points": 3000,
    "created": "2024-01-10"
  }
}
```

### cafe_info.json

```json
{
  "영업시간": {
    "평일": "오전 8시부터 오후 10시까지",
    "주말": "오전 9시부터 오후 11시까지"
  },
  "와이파이": {
    "이름": "cafe",
    "비밀번호": "cafe1234"
  }
}
```

## 기술 스택 및 구현 특징

### 기술 스택

- **Backend**: Python 3.12
- **AI/LLM**: OpenAI GPT-4.1 / GPT-4.1-mini
- **STT**: OpenAI Transcribe API (gpt-4o-mini-transcribe) + Silero VAD
- **TTS**: OpenAI TTS API (gpt-4o-mini-tts)
- **UI Framework**: Streamlit
- **데이터 저장**: JSON 파일 기반
- **비동기 처리**: Python asyncio
- **오디오 처리**: PyAudio, torch

### 핵심 구현 특징

### 1. **스트리밍 응답 처리**

- OpenAI Streaming API를 활용한 실시간 응답 생성
- 비동기 제너레이터를 통한 청크 단위 전송
- Streamlit의 `st.write_stream()`과 통합

### 2. **Function Calling 통합**

- OpenAI Function Calling API 활용
- 도구 스키마 정의 (`schemas.py`)
- 자동 도구 실행 및 결과 반영

### 3. **대화 컨텍스트 최적화**

- 이원화된 메시지 관리:
  - `messages`: UI 표시용 전체 기록
  - `llm_messages`: LLM 전달용 최적화 기록
- 장바구니 도구 사용 시 자동 컨텍스트 초기화

### 4. **에이전트 아키텍처**

- 추상 베이스 클래스 (`BaseAgent`) 기반 상속 구조
- 공통 스트리밍 및 도구 처리 로직 재사용
- 에이전트별 특화된 온도 설정 및 도구 구성

### 5. **옵션 검증 시스템**

- 메뉴별 필수 옵션 자동 검증
- 옵션 정규화 및 중복 방지
- 누락된 옵션 자동 요청

### 6. **음성 처리 시스템**

- **VAD 기반 음성 감지**: Silero VAD로 실시간 음성 활동 감지
- **자동 녹음 시작/종료**: 음성 감지 시 자동 녹음, 침묵 시 자동 종료
- **Always-Listening**: 연속적인 음성 대화를 위한 자동 재활성화
- **스트리밍 TTS**: OpenAI TTS API를 통한 실시간 음성 스트리밍

## 도구 및 API

### 장바구니 도구 (Order Agent)

| 도구명             | 파라미터                     | 설명                   |
| ------------------ | ---------------------------- | ---------------------- |
| `add_to_cart`      | menu_item, quantity, options | 장바구니에 메뉴 추가   |
| `remove_from_cart` | menu_item, options, quantity | 장바구니에서 메뉴 제거 |
| `view_cart`        | -                            | 장바구니 내용 조회     |

### 멤버십 도구 (Order Agent)

| 도구명              | 파라미터             | 설명             |
| ------------------- | -------------------- | ---------------- |
| `search_membership` | query                | 멤버십 정보 조회 |
| `use_points`        | number, name, points | 포인트 사용      |
| `add_points`        | number, name, points | 포인트 적립      |
| `create_membership` | number, name         | 멤버십 생성      |

### 날씨 도구 (Conversation Agent)

| 도구명           | 파라미터   | 설명                     |
| ---------------- | ---------- | ------------------------ |
| `weather_search` | date, city | 특정 날짜/도시 날씨 조회 |

## UI/UX 특징

- **Streamlit 기반**: 간단하고 직관적인 웹 인터페이스
- **실시간 스트리밍**: 타이핑 효과로 자연스러운 응답 표시
- **사이드바 메뉴**:
  - 세션 관리 (시작/종료)
  - 음성 기능 설정 (STT/TTS 토글, 음성 선택, 속도 조절)
  - 메뉴 조회
  - 멤버십 관리
  - 장바구니 표시
- **음성 인터페이스**:
  - Always-Listening 모드로 연속 대화
  - 실시간 음성 상태 표시
  - 자동 음성 출력
- **반응형 디자인**: 다양한 화면 크기 지원
- **커스텀 CSS**: 깔끔한 디자인과 통일된 색상 테마

## 주요 의존성

```toml
[project]
dependencies = [
    "streamlit>=1.49.0",           # UI 프레임워크
    "openai[voice-helpers]>=1.102.0",  # OpenAI API + 음성 헬퍼
    "python-dotenv>=1.1.1",        # 환경변수 관리
    "requests>=2.32.5",            # HTTP 요청
    # 음성 처리
    "silero-vad>=5.1",             # 음성 활동 감지
    "torch>=2.0.0",                # PyTorch (VAD용)
    "pyaudio>=0.2.14",             # 오디오 입출력
]

```
