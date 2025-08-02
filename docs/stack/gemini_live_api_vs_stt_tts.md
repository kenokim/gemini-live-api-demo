### 가격 조사

#### Gemini Live API (일체형 스트리밍)
- 설명: STT·TTS 일체형 스트리밍
- 버전: Gemini 2.5 Flash (Preview)
- 가격:
  - Audio 입력: $3 / 1M token
  - Audio 출력: $12 / 1M token
- 1분 기준 (≈ 1,920 token 입/출 가정 시):
  - 총 비용: $0.029 (약 39원)

---

#### 모듈 조합 (STT + Gemini Text LLM + TTS)

1.  STT (v2 Standard): $0.016/분
2.  LLM (Gemini 2.5 Flash):
    - 입력: $0.30 / 1M token
    - 출력: $2.50 / 1M token
    - *(200 ↔ 200 token 가정)*
3.  TTS 옵션별 조합 비용:

    - Standard voice
      - TTS 가격: $4 / 1M char (→ 960 char)
      - 총 조합 비용: $0.020 (약 28원)
      - 특징: 가장 저렴, 음색 품질 보통

    - WaveNet voice
      - TTS 가격: $16 / 1M char
      - 총 조합 비용: $0.032 (약 41원)
      - 특징: Live API와 거의 동일한 품질 및 가격

    - Chirp-3 HD
      - TTS 가격: $30 / 1M char
      - 총 조합 비용: $0.045 (약 61원)
      - 특징: 고품질 TTS, Live API보다 50% 비쌈
