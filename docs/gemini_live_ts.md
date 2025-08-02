# Gemini Live API TypeScript 구현 가이드

## 개요

이 문서는 TypeScript를 사용하여 Gemini Live API를 연동하는 실시간 음성 에이전트를 구축하는 방법을 코드 레벨에서 상세히 설명합니다.

## 1. 환경 설정

### 1.1 필수 요구사항
- Node.js 20 이상
- TypeScript
- @google/genai SDK (최신 공식 SDK)

### 1.2 프로젝트 초기화

```bash
# 프로젝트 디렉토리 생성
mkdir gemini-live-agent && cd gemini-live-agent

# Node.js 프로젝트 초기화
npm init -y

# TypeScript 설치
npm install -D typescript @types/node @types/express @types/ws

# 핵심 의존성 설치
npm install @google/genai express ws dotenv

# TypeScript 설정 파일 생성
tsc --init
```

### 1.3 환경 변수 설정

`.env` 파일 생성:
```env
GEMINI_API_KEY=your_api_key_here
PORT=8080
```

`.gitignore` 파일에 추가:
```
.env
node_modules/
dist/
```

## 2. 아키텍처 설계

### 2.1 서버-서버 아키텍처 (프로덕션 권장)

```
클라이언트 (브라우저) → 자체 백엔드 서버 → Gemini Live API
```

**장점:**
- 최대 보안 (API 키 노출 없음)
- 완전한 제어권
- 프로덕션 환경에 적합

**단점:**
- 구현 복잡성 증가
- 추가 네트워크 홉으로 인한 지연 가능성

## 3. 서버 측 구현

### 3.1 Gemini Live Service 모듈

`src/gemini-live-service.ts`:
```typescript
import { GoogleGenAI } from '@google/genai';

const apiKey = process.env.GEMINI_API_KEY;
if (!apiKey) {
  throw new Error("GEMINI_API_KEY environment variable not set.");
}

const ai = new GoogleGenAI({ apiKey });

export async function startLiveSession() {
  try {
    const session = await ai.live.connect({
      model: 'gemini-live-2.5-flash', // Live API 전용 모델
      config: {
        response_modalities: ['audio', 'text'],
        input_audio_transcription: {}, // 실시간 음성-텍스트 변환
        output_audio_transcription: {}, // 출력 음성-텍스트 변환
        generation_config: {
          temperature: 0.7,
          max_output_tokens: 2048,
        },
        safety_settings: [
          {
            category: 'HARM_CATEGORY_HARASSMENT',
            threshold: 'BLOCK_MEDIUM_AND_ABOVE'
          },
          {
            category: 'HARM_CATEGORY_HATE_SPEECH',
            threshold: 'BLOCK_MEDIUM_AND_ABOVE'
          }
        ]
      },
    });
    
    console.log("Gemini Live API session started.");
    return session;
  } catch (error) {
    console.error("Failed to start Gemini Live session:", error);
    throw error;
  }
}

// Function Calling 지원을 위한 도구 정의 예시
export const weatherTool = {
  name: 'get_weather',
  description: '특정 도시의 현재 날씨 정보를 가져옵니다',
  parameters: {
    type: 'object',
    properties: {
      city: {
        type: 'string',
        description: '날씨를 조회할 도시명'
      }
    },
    required: ['city']
  }
};

// 실제 날씨 API 호출 함수
export async function getWeather(city: string): Promise<string> {
  // 실제 구현에서는 외부 날씨 API 호출
  return `${city}의 현재 날씨는 맑음, 기온 22도입니다.`;
}
```

### 3.2 Express 서버 구현

`src/index.ts`:
```typescript
import express from 'express';
import http from 'http';
import { WebSocketServer, WebSocket } from 'ws';
import dotenv from 'dotenv';
import { startLiveSession, weatherTool, getWeather } from './gemini-live-service';
import path from 'path';

dotenv.config();

const app = express();
const server = http.createServer(app);
const wss = new WebSocketServer({ server });

const PORT = process.env.PORT || 8080;

// 정적 파일 서빙 (클라이언트 HTML/JS)
app.use(express.static(path.join(__dirname, '../public')));

// WebSocket 연결 처리
wss.on('connection', async (ws: WebSocket) => {
  console.log('Client connected');
  
  let geminiSession: any = null;
  
  try {
    // 각 클라이언트 연결에 대해 새로운 Gemini Live 세션 시작
    geminiSession = await startLiveSession();
    
    // Gemini로부터 메시지 수신 및 클라이언트로 전달
    (async () => {
      try {
        for await (const response of geminiSession.receive()) {
          if (response.server_content) {
            // 도구 호출 처리
            if (response.server_content.tool_call) {
              const toolCall = response.server_content.tool_call;
              if (toolCall.function_calls) {
                for (const funcCall of toolCall.function_calls) {
                  if (funcCall.name === 'get_weather') {
                    const city = funcCall.args.city;
                    const weatherResult = await getWeather(city);
                    
                    // 도구 실행 결과를 Gemini에 전송
                    await geminiSession.send({
                      tool_response: {
                        function_responses: [{
                          name: 'get_weather',
                          response: { result: weatherResult }
                        }]
                      }
                    });
                  }
                }
              }
            }
            
            // 응답을 클라이언트로 전송
            ws.send(JSON.stringify({
              type: 'gemini_response',
              data: response.server_content
            }));
          }
        }
      } catch (error) {
        console.error('Error receiving from Gemini:', error);
        ws.send(JSON.stringify({
          type: 'error',
          message: 'Gemini 연결 오류가 발생했습니다.'
        }));
      }
    })();
    
    // 클라이언트로부터 메시지 수신 및 Gemini로 전달
    ws.on('message', async (message: Buffer) => {
      try {
        const data = JSON.parse(message.toString());
        
        if (data.type === 'audio') {
          // 오디오 데이터를 Gemini로 전송
          await geminiSession.send({
            realtime_input: {
              media_chunks: [{
                mime_type: 'audio/pcm',
                data: data.audio_data
              }]
            }
          });
        } else if (data.type === 'text') {
          // 텍스트 메시지를 Gemini로 전송
          await geminiSession.send({
            client_content: {
              turns: [{
                role: 'user',
                parts: [{ text: data.text }]
              }]
            }
          });
        }
      } catch (error) {
        console.error('Error processing client message:', error);
        ws.send(JSON.stringify({
          type: 'error',
          message: '메시지 처리 중 오류가 발생했습니다.'
        }));
      }
    });
    
    ws.on('close', () => {
      console.log('Client disconnected');
      // 세션 정리
      if (geminiSession) {
        geminiSession.close?.();
      }
    });
    
    ws.on('error', (error) => {
      console.error('WebSocket error:', error);
    });
    
  } catch (error) {
    console.error('Failed to initialize connection with Gemini:', error);
    ws.close(1011, 'Failed to connect to AI service');
  }
});

server.listen(PORT, () => {
  console.log(`Server is listening on port ${PORT}`);
});

// 우아한 종료 처리
process.on('SIGINT', () => {
  console.log('Shutting down server...');
  server.close(() => {
    process.exit(0);
  });
});
```

## 4. React 클라이언트 구현

### 4.1 React 프로젝트 설정

```bash
# React 프로젝트 생성
npx create-react-app gemini-live-client --template typescript
cd gemini-live-client

# 추가 의존성 설치
npm install @types/ws
```

### 4.2 WebSocket Hook 구현

`src/hooks/useWebSocket.ts`:
```typescript
import { useEffect, useRef, useState, useCallback } from 'react';

interface WebSocketMessage {
  type: string;
  data?: any;
  message?: string;
}

interface UseWebSocketReturn {
  ws: WebSocket | null;
  connectionStatus: 'connecting' | 'connected' | 'disconnected' | 'error';
  sendMessage: (message: any) => void;
  lastMessage: WebSocketMessage | null;
}

export const useWebSocket = (url: string): UseWebSocketReturn => {
  const [connectionStatus, setConnectionStatus] = useState<'connecting' | 'connected' | 'disconnected' | 'error'>('connecting');
  const [lastMessage, setLastMessage] = useState<WebSocketMessage | null>(null);
  const wsRef = useRef<WebSocket | null>(null);

  const sendMessage = useCallback((message: any) => {
    if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(message));
    }
  }, []);

  useEffect(() => {
    const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    const wsUrl = `${protocol}//${window.location.host}`;
    
    const ws = new WebSocket(wsUrl);
    wsRef.current = ws;

    ws.onopen = () => {
      console.log('WebSocket connected');
      setConnectionStatus('connected');
    };

    ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data);
        setLastMessage(message);
      } catch (error) {
        console.error('Error parsing server message:', error);
      }
    };

    ws.onclose = () => {
      console.log('WebSocket disconnected');
      setConnectionStatus('disconnected');
    };

    ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      setConnectionStatus('error');
    };

    return () => {
      ws.close();
    };
  }, [url]);

  return {
    ws: wsRef.current,
    connectionStatus,
    sendMessage,
    lastMessage
  };
};
```

### 4.3 오디오 녹음 Hook 구현

`src/hooks/useAudioRecorder.ts`:
```typescript
import { useState, useRef, useCallback } from 'react';

interface UseAudioRecorderReturn {
  isRecording: boolean;
  startRecording: () => Promise<void>;
  stopRecording: () => void;
  audioChunks: Blob[];
}

export const useAudioRecorder = (onAudioChunk?: (chunk: Blob) => void): UseAudioRecorderReturn => {
  const [isRecording, setIsRecording] = useState(false);
  const [audioChunks, setAudioChunks] = useState<Blob[]>([]);
  const mediaRecorderRef = useRef<MediaRecorder | null>(null);
  const streamRef = useRef<MediaStream | null>(null);

  const startRecording = useCallback(async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        audio: {
          sampleRate: 16000,
          channelCount: 1,
          echoCancellation: true,
          noiseSuppression: true
        }
      });

      streamRef.current = stream;
      const mediaRecorder = new MediaRecorder(stream, {
        mimeType: 'audio/webm;codecs=opus'
      });
      mediaRecorderRef.current = mediaRecorder;

      const chunks: Blob[] = [];
      setAudioChunks([]);

      mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
          chunks.push(event.data);
          setAudioChunks(prev => [...prev, event.data]);
          if (onAudioChunk) {
            onAudioChunk(event.data);
          }
        }
      };

      mediaRecorder.onstop = () => {
        stream.getTracks().forEach(track => track.stop());
      };

      mediaRecorder.start(100); // 100ms 간격으로 데이터 수집
      setIsRecording(true);
    } catch (error) {
      console.error('Error starting recording:', error);
      throw error;
    }
  }, [onAudioChunk]);

  const stopRecording = useCallback(() => {
    if (mediaRecorderRef.current && isRecording) {
      mediaRecorderRef.current.stop();
      setIsRecording(false);
    }
  }, [isRecording]);

  return {
    isRecording,
    startRecording,
    stopRecording,
    audioChunks
  };
};
```

### 4.4 메인 컴포넌트 구현

`src/components/GeminiLiveAgent.tsx`:
```typescript
import React, { useState, useEffect, useCallback } from 'react';
import { useWebSocket } from '../hooks/useWebSocket';
import { useAudioRecorder } from '../hooks/useAudioRecorder';
import { TranscriptDisplay } from './TranscriptDisplay';
import { ControlPanel } from './ControlPanel';
import { StatusIndicator } from './StatusIndicator';
import { AudioPlayer } from './AudioPlayer';
import './GeminiLiveAgent.css';

interface Message {
  id: string;
  timestamp: Date;
  type: 'user' | 'assistant' | 'error';
  content: string;
}

export const GeminiLiveAgent: React.FC = () => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [audioToPlay, setAudioToPlay] = useState<string | null>(null);
  
  const { connectionStatus, sendMessage, lastMessage } = useWebSocket('');
  
  const handleAudioChunk = useCallback(async (chunk: Blob) => {
    try {
      const arrayBuffer = await chunk.arrayBuffer();
      const base64Audio = btoa(String.fromCharCode(...new Uint8Array(arrayBuffer)));
      
      sendMessage({
        type: 'audio',
        audio_data: base64Audio
      });
    } catch (error) {
      console.error('Error sending audio chunk:', error);
    }
  }, [sendMessage]);
  
  const { isRecording, startRecording, stopRecording } = useAudioRecorder(handleAudioChunk);

  const addMessage = useCallback((content: string, type: 'user' | 'assistant' | 'error') => {
    const newMessage: Message = {
      id: Date.now().toString(),
      timestamp: new Date(),
      type,
      content
    };
    setMessages(prev => [...prev, newMessage]);
  }, []);

  const handleToggleRecording = useCallback(async () => {
    if (isRecording) {
      stopRecording();
    } else {
      try {
        await startRecording();
      } catch (error) {
        addMessage('마이크 접근 오류가 발생했습니다.', 'error');
      }
    }
  }, [isRecording, startRecording, stopRecording, addMessage]);

  const handleClearMessages = useCallback(() => {
    setMessages([]);
  }, []);

  // WebSocket 메시지 처리
  useEffect(() => {
    if (!lastMessage) return;

    switch (lastMessage.type) {
      case 'gemini_response':
        const data = lastMessage.data;
        
        // 텍스트 응답 처리
        if (data.model_turn && data.model_turn.parts) {
          for (const part of data.model_turn.parts) {
            if (part.text) {
              addMessage(part.text, 'assistant');
            }
          }
        }
        
        // 오디오 응답 처리
        if (data.model_turn && data.model_turn.parts) {
          for (const part of data.model_turn.parts) {
            if (part.inline_data && part.inline_data.mime_type === 'audio/pcm') {
              setAudioToPlay(part.inline_data.data);
            }
          }
        }
        
        // 음성 전사 처리
        if (data.input_audio_transcription && data.input_audio_transcription.transcript) {
          addMessage(data.input_audio_transcription.transcript, 'user');
        }
        break;
        
      case 'error':
        addMessage(`오류: ${lastMessage.message}`, 'error');
        break;
        
      default:
        console.log('Unknown message type:', lastMessage.type);
    }
  }, [lastMessage, addMessage]);

  return (
    <div className="gemini-live-agent">
      <header className="header">
        <h1>Gemini Live Voice Agent</h1>
        <StatusIndicator status={connectionStatus} />
      </header>
      
      <main className="main-content">
        <ControlPanel
          isRecording={isRecording}
          onToggleRecording={handleToggleRecording}
          onClearMessages={handleClearMessages}
          disabled={connectionStatus !== 'connected'}
        />
        
        <TranscriptDisplay messages={messages} />
        
        {audioToPlay && (
          <AudioPlayer
            audioData={audioToPlay}
            onPlayComplete={() => setAudioToPlay(null)}
          />
        )}
      </main>
    </div>
  );
};
```

### 4.5 하위 컴포넌트들

`src/components/StatusIndicator.tsx`:
```typescript
import React from 'react';

interface StatusIndicatorProps {
  status: 'connecting' | 'connected' | 'disconnected' | 'error';
}

export const StatusIndicator: React.FC<StatusIndicatorProps> = ({ status }) => {
  const getStatusText = () => {
    switch (status) {
      case 'connecting': return '연결 중...';
      case 'connected': return '연결됨';
      case 'disconnected': return '연결 끊김';
      case 'error': return '연결 오류';
      default: return '알 수 없음';
    }
  };

  return (
    <div className={`status-indicator status-${status}`}>
      {getStatusText()}
    </div>
  );
};
```

`src/components/ControlPanel.tsx`:
```typescript
import React from 'react';

interface ControlPanelProps {
  isRecording: boolean;
  onToggleRecording: () => void;
  onClearMessages: () => void;
  disabled: boolean;
}

export const ControlPanel: React.FC<ControlPanelProps> = ({
  isRecording,
  onToggleRecording,
  onClearMessages,
  disabled
}) => {
  return (
    <div className="control-panel">
      <button
        className={`record-button ${isRecording ? 'recording' : ''}`}
        onClick={onToggleRecording}
        disabled={disabled}
      >
        {isRecording ? '🛑 녹음 중지' : '🎤 녹음 시작'}
      </button>
      
      <button
        className="clear-button"
        onClick={onClearMessages}
      >
        대화 내용 지우기
      </button>
    </div>
  );
};
```

`src/components/TranscriptDisplay.tsx`:
```typescript
import React, { useEffect, useRef } from 'react';

interface Message {
  id: string;
  timestamp: Date;
  type: 'user' | 'assistant' | 'error';
  content: string;
}

interface TranscriptDisplayProps {
  messages: Message[];
}

export const TranscriptDisplay: React.FC<TranscriptDisplayProps> = ({ messages }) => {
  const transcriptRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (transcriptRef.current) {
      transcriptRef.current.scrollTop = transcriptRef.current.scrollHeight;
    }
  }, [messages]);

  const getMessageIcon = (type: string) => {
    switch (type) {
      case 'user': return '👤';
      case 'assistant': return '🤖';
      case 'error': return '❌';
      default: return '💬';
    }
  };

  const getMessageLabel = (type: string) => {
    switch (type) {
      case 'user': return '사용자';
      case 'assistant': return 'Gemini';
      case 'error': return '오류';
      default: return '시스템';
    }
  };

  return (
    <div className="transcript-display" ref={transcriptRef}>
      {messages.length === 0 ? (
        <p className="empty-message">
          <em>여기에 대화 내용이 표시됩니다...</em>
        </p>
      ) : (
        messages.map((message) => (
          <div key={message.id} className={`message message-${message.type}`}>
            <div className="message-header">
              <span className="message-icon">{getMessageIcon(message.type)}</span>
              <span className="message-label">{getMessageLabel(message.type)}</span>
              <span className="message-timestamp">
                {message.timestamp.toLocaleTimeString()}
              </span>
            </div>
            <div className="message-content">{message.content}</div>
          </div>
        ))
      )}
    </div>
  );
};
```

`src/components/AudioPlayer.tsx`:
```typescript
import React, { useEffect, useRef } from 'react';

interface AudioPlayerProps {
  audioData: string;
  onPlayComplete: () => void;
}

export const AudioPlayer: React.FC<AudioPlayerProps> = ({ audioData, onPlayComplete }) => {
  const audioContextRef = useRef<AudioContext | null>(null);

  useEffect(() => {
    const playAudio = async () => {
      try {
        if (!audioContextRef.current) {
          audioContextRef.current = new (window.AudioContext || (window as any).webkitAudioContext)({
            sampleRate: 24000
          });
        }

        const audioContext = audioContextRef.current;
        const decodedData = atob(audioData);
        const audioBuffer = new Uint8Array(decodedData.length);
        
        for (let i = 0; i < decodedData.length; i++) {
          audioBuffer[i] = decodedData.charCodeAt(i);
        }

        const decodedBuffer = await audioContext.decodeAudioData(audioBuffer.buffer);
        const source = audioContext.createBufferSource();
        source.buffer = decodedBuffer;
        source.connect(audioContext.destination);
        
        source.onended = onPlayComplete;
        source.start();
      } catch (error) {
        console.error('Error playing audio:', error);
        onPlayComplete();
      }
    };

    if (audioData) {
      playAudio();
    }
  }, [audioData, onPlayComplete]);

  return null; // 이 컴포넌트는 UI를 렌더링하지 않음
};
```

## 5. 고급 기능 구현

### 5.1 Function Calling 확장

```typescript
// src/tools.ts
export const availableTools = {
  get_weather: {
    definition: {
      name: 'get_weather',
      description: '특정 도시의 현재 날씨 정보를 가져옵니다',
      parameters: {
        type: 'object',
        properties: {
          city: { type: 'string', description: '날씨를 조회할 도시명' }
        },
        required: ['city']
      }
    },
    handler: async (args: { city: string }) => {
      // 실제 날씨 API 호출
      return `${args.city}의 현재 날씨는 맑음, 기온 22도입니다.`;
    }
  },
  
  get_time: {
    definition: {
      name: 'get_time',
      description: '현재 시간을 가져옵니다',
      parameters: { type: 'object', properties: {} }
    },
    handler: async () => {
      return new Date().toLocaleString('ko-KR');
    }
  }
};
```

### 5.2 오류 처리 및 재연결 로직

```typescript
// src/error-handler.ts
export class ErrorHandler {
  static handleApiError(error: any): string {
    if (error.status) {
      switch (error.status) {
        case 400:
          return '잘못된 요청입니다. 입력을 확인해주세요.';
        case 429:
          return '요청이 너무 많습니다. 잠시 후 다시 시도해주세요.';
        case 500:
          return '서버 오류가 발생했습니다. 잠시 후 다시 시도해주세요.';
        case 503:
          return '서비스가 일시적으로 사용할 수 없습니다.';
        default:
          return '알 수 없는 오류가 발생했습니다.';
      }
    }
    return '네트워크 오류가 발생했습니다.';
  }
  
  static async retryWithBackoff<T>(
    fn: () => Promise<T>,
    maxRetries: number = 3,
    baseDelay: number = 1000
  ): Promise<T> {
    for (let i = 0; i < maxRetries; i++) {
      try {
        return await fn();
      } catch (error) {
        if (i === maxRetries - 1) throw error;
        
        const delay = baseDelay * Math.pow(2, i);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
    throw new Error('Max retries exceeded');
  }
}
```

## 6. 빌드 및 배포

### 6.1 TypeScript 컴파일 설정

`tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 6.2 빌드 스크립트

**서버 측 `package.json`:**
```json
{
  "name": "gemini-live-server",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsc --watch & nodemon dist/index.js",
    "clean": "rm -rf dist"
  },
  "dependencies": {
    "@google/genai": "^latest",
    "express": "^4.18.0",
    "ws": "^8.14.0",
    "dotenv": "^16.3.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0",
    "@types/express": "^4.17.0",
    "@types/ws": "^8.5.0",
    "nodemon": "^3.0.0"
  }
}
```

**클라이언트 측 `package.json`:**
```json
{
  "name": "gemini-live-client",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@types/node": "^16.18.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@types/ws": "^8.5.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-scripts": "5.0.1",
    "typescript": "^4.9.0",
    "web-vitals": "^2.1.0"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": [
      "react-app",
      "react-app/jest"
    ]
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  },
  "proxy": "http://localhost:8080"
}
```

### 6.3 CSS 스타일링

`src/components/GeminiLiveAgent.css`:
```css
.gemini-live-agent {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', sans-serif;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 30px;
  padding-bottom: 20px;
  border-bottom: 1px solid #e0e0e0;
}

.header h1 {
  margin: 0;
  color: #333;
  font-size: 2rem;
}

.status-indicator {
  padding: 8px 16px;
  border-radius: 20px;
  font-size: 0.9rem;
  font-weight: 500;
}

.status-connecting {
  background-color: #fff3cd;
  color: #856404;
}

.status-connected {
  background-color: #d4edda;
  color: #155724;
}

.status-disconnected,
.status-error {
  background-color: #f8d7da;
  color: #721c24;
}

.control-panel {
  display: flex;
  gap: 15px;
  margin-bottom: 25px;
  justify-content: center;
}

.record-button,
.clear-button {
  padding: 12px 24px;
  font-size: 16px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-weight: 500;
}

.record-button {
  background-color: #007bff;
  color: white;
}

.record-button:hover:not(:disabled) {
  background-color: #0056b3;
}

.record-button.recording {
  background-color: #dc3545;
  animation: pulse 1.5s infinite;
}

.record-button:disabled {
  background-color: #6c757d;
  cursor: not-allowed;
}

.clear-button {
  background-color: #6c757d;
  color: white;
}

.clear-button:hover {
  background-color: #545b62;
}

@keyframes pulse {
  0% { opacity: 1; }
  50% { opacity: 0.7; }
  100% { opacity: 1; }
}

.transcript-display {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 20px;
  min-height: 400px;
  max-height: 600px;
  overflow-y: auto;
  background-color: #fafafa;
}

.empty-message {
  text-align: center;
  color: #666;
  font-style: italic;
  margin-top: 100px;
}

.message {
  margin-bottom: 20px;
  padding: 15px;
  border-radius: 8px;
  background-color: white;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.message-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  font-size: 0.9rem;
}

.message-icon {
  font-size: 1.2rem;
}

.message-label {
  font-weight: 600;
  color: #333;
}

.message-timestamp {
  color: #666;
  margin-left: auto;
  font-size: 0.8rem;
}

.message-content {
  line-height: 1.5;
  color: #444;
}

.message-user {
  border-left: 4px solid #007bff;
}

.message-assistant {
  border-left: 4px solid #28a745;
}

.message-error {
  border-left: 4px solid #dc3545;
  background-color: #fff5f5;
}

.main-content {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

@media (max-width: 768px) {
  .gemini-live-agent {
    padding: 15px;
  }
  
  .header {
    flex-direction: column;
    gap: 15px;
    text-align: center;
  }
  
  .control-panel {
    flex-direction: column;
    align-items: center;
  }
  
  .record-button,
  .clear-button {
    width: 100%;
    max-width: 300px;
  }
}
```

### 6.4 메인 App 컴포넌트

`src/App.tsx`:
```typescript
import React from 'react';
import { GeminiLiveAgent } from './components/GeminiLiveAgent';
import './App.css';

function App() {
  return (
    <div className="App">
      <GeminiLiveAgent />
    </div>
  );
}

export default App;
```

### 6.5 실행 명령어

```bash
# 서버 실행 (백엔드)
cd server
npm run dev

# 클라이언트 실행 (프론트엔드)
cd client
npm start

# 프로덕션 빌드
# 서버
cd server
npm run build
npm start

# 클라이언트
cd client
npm run build
```

## 7. 보안 고려사항

### 7.1 API 키 보안
- 절대 클라이언트 측에 API 키 노출 금지
- 환경 변수 사용
- .gitignore에 .env 파일 추가

### 7.2 임시 토큰 구현 (고급)

```typescript
// 임시 토큰 생성 엔드포인트
app.post('/api/get-gemini-token', async (req, res) => {
  try {
    // 사용자 인증 확인
    const token = await generateEphemeralToken();
    res.json({ token, expiresIn: 3600 }); // 1시간 유효
  } catch (error) {
    res.status(500).json({ error: 'Token generation failed' });
  }
});
```

## 8. 성능 최적화

### 8.1 오디오 형식 최적화
- 입력: 16-bit PCM, 16kHz, 모노
- 출력: 24kHz
- 실시간 형식 변환 구현

### 8.2 연결 관리
- WebSocket 연결 풀링
- 자동 재연결 로직
- 세션 상태 관리

## 9. 테스트 및 디버깅

### 9.1 로깅 설정

```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
    new winston.transports.Console()
  ]
});
```

### 9.2 모니터링
- WebSocket 연결 상태 모니터링
- API 호출 성공률 추적
- 오디오 품질 메트릭

## 결론

이 가이드는 TypeScript를 사용하여 Gemini Live API를 연동하는 완전한 실시간 음성 에이전트를 구축하는 방법을 제시합니다. 프로덕션 환경에서는 추가적인 보안, 성능 최적화, 모니터링이 필요하며, 실제 오디오 형식 변환 로직의 구현이 중요합니다.

주요 포인트:
1. @google/genai SDK 사용 (최신 공식 SDK)
2. 서버-서버 아키텍처로 보안 확보
3. WebSocket을 통한 실시간 양방향 통신
4. Function Calling으로 기능 확장
5. 포괄적인 오류 처리 및 재연결 로직

이 구현을 기반으로 다양한 실시간 AI 애플리케이션을 개발할 수 있습니다.