# Voice Interaction MVP - Product Requirements Document

## 프로젝트 개요

### 목적
Gemini Live API와 LangGraph를 활용하여 실시간 음성 상호작용이 가능한 웹 애플리케이션 MVP 개발

### 기술 스택
- Frontend: React + TypeScript
- Backend: Express + TypeScript
- AI/ML: Gemini Live API, LangGraph
- 음성 처리: Web Audio API, MediaRecorder API

## 핵심 기능 (MVP)

### 1. 음성 입력 기능
- 기능: 사용자 음성을 실시간으로 캡처
- 구현: 
  - MediaRecorder API를 사용한 음성 녹음
  - 실시간 오디오 스트림 처리
  - 음성 데이터를 서버로 전송

### 2. 음성-텍스트 변환
- 기능: 음성을 텍스트로 변환
- 구현:
  - Gemini Live API의 STT 기능 활용
  - 실시간 음성 인식 처리

### 3. AI 응답 생성
- 기능: 사용자 입력에 대한 지능적 응답 생성
- 구현:
  - LangGraph를 통한 대화 플로우 관리
  - Gemini API를 통한 자연어 처리
  - 컨텍스트 유지 및 대화 히스토리 관리

### 4. 텍스트-음성 변환
- 기능: AI 응답을 음성으로 출력
- 구현:
  - Gemini Live API의 TTS 기능 활용
  - 오디오 스트림 재생

### 5. 실시간 대화 인터페이스
- 기능: 직관적인 음성 대화 UI
- 구현:
  - 녹음 상태 표시
  - 실시간 텍스트 표시
  - 대화 히스토리 표시

## 시스템 아키텍처

### Frontend (React + TypeScript)
```
src/
├── components/
│   ├── VoiceRecorder.tsx      # 음성 녹음 컴포넌트
│   ├── ConversationDisplay.tsx # 대화 표시 컴포넌트
│   └── AudioPlayer.tsx        # 음성 재생 컴포넌트
├── hooks/
│   ├── useVoiceRecording.ts   # 음성 녹음 훅
│   ├── useWebSocket.ts        # WebSocket 연결 훅
│   └── useAudioPlayer.ts      # 오디오 재생 훅
├── services/
│   └── api.ts                 # API 통신 서비스
└── types/
    └── conversation.ts        # 타입 정의
```

### Backend (Express + TypeScript)
```
src/
├── routes/
│   ├── voice.ts              # 음성 처리 라우트
│   └── conversation.ts       # 대화 관리 라우트
├── services/
│   ├── geminiService.ts      # Gemini API 서비스
│   ├── langGraphService.ts   # LangGraph 서비스
│   └── audioService.ts       # 오디오 처리 서비스
├── middleware/
│   └── websocket.ts          # WebSocket 미들웨어
└── types/
    └── api.ts                # API 타입 정의
```

## API 설계

### WebSocket Events

#### Client → Server
- `voice_start`: 음성 녹음 시작
- `voice_chunk`: 음성 데이터 청크
- `voice_end`: 음성 녹음 종료

#### Server → Client
- `transcription`: 실시간 음성 인식 결과
- `ai_response`: AI 응답 텍스트
- `audio_response`: AI 응답 오디오
- `error`: 에러 메시지

### REST API

#### POST /api/conversation
- 목적: 새로운 대화 세션 시작
- 응답: `{ sessionId: string }`

#### GET /api/conversation/:sessionId
- 목적: 대화 히스토리 조회
- 응답: `{ messages: Message[] }`

## 데이터 모델

### Message
```typescript
interface Message {
  id: string;
  sessionId: string;
  type: 'user' | 'assistant';
  content: string;
  audioUrl?: string;
  timestamp: Date;
}
```

### ConversationSession
```typescript
interface ConversationSession {
  id: string;
  userId?: string;
  messages: Message[];
  createdAt: Date;
  updatedAt: Date;
}
```

## 구현 우선순위

### Phase 1: 기본 음성 처리
1. 음성 녹음 및 재생 기능
2. Gemini Live API 연동
3. 기본 STT/TTS 구현

### Phase 2: AI 대화 기능
1. LangGraph 대화 플로우 구현
2. 컨텍스트 관리
3. 실시간 WebSocket 통신

### Phase 3: UI/UX 개선
1. 반응형 대화 인터페이스
2. 음성 시각화
3. 에러 처리 및 사용자 피드백

## 기술적 고려사항

### 성능
- 음성 데이터 스트리밍 최적화
- 실시간 처리를 위한 버퍼링 전략
- 메모리 사용량 관리

### 보안
- API 키 보안 관리
- 음성 데이터 암호화
- 세션 관리 및 인증

### 확장성
- 모듈화된 아키텍처
- 플러그인 가능한 AI 서비스
- 수평 확장 가능한 설계

## 개발 환경 설정

### 필수 환경 변수
```env
GEMINI_API_KEY=your_gemini_api_key
LANGGRAPH_API_KEY=your_langgraph_api_key
PORT=3001
CLIENT_URL=http://localhost:3000
```

### 패키지 의존성

#### Frontend
- react, react-dom
- typescript
- @types/react, @types/react-dom
- socket.io-client
- axios

#### Backend
- express
- typescript
- socket.io
- @google-ai/generativelanguage
- langgraph
- multer (파일 업로드)
- cors
