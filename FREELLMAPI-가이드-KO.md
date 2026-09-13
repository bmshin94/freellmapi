# FreeLLMAPI 완전 정리 (한국어)

> 무료 LLM 제공사 34곳의 무료 티어를 한 곳에 모아, **OpenAI 호환 엔드포인트 하나**로 쓰게 해주는 로컬 게이트웨이.
> 이 문서는 리포지토리를 직접 뜯어보며 정리한 한국어 요약본입니다.

## 🔗 링크 모음

| 항목 | 주소 |
| --- | --- |
| 📦 **내 리포지토리 (fork)** | https://github.com/bmshin94/freellmapi |
| ⭐ **원본 리포지토리** | https://github.com/tashfeenahmed/freellmapi |
| 🌐 공식 사이트 | https://freellmapi.co |
| 📋 모델 카탈로그 | https://freellmapi.co/models.html |
| 💾 릴리스 (데스크톱 앱) | https://github.com/tashfeenahmed/freellmapi/releases/latest |
| 📖 설치 문서 | [docs/en/install/01-install.md](docs/en/install/01-install.md) |
| 📖 API 문서 | [docs/en/api/01-rest-api.md](docs/en/api/01-rest-api.md) |
| 📖 아키텍처 문서 | [docs/en/architecture/00-high-level-index.md](docs/en/architecture/00-high-level-index.md) |
| 📖 에이전트 연동 | [docs/en/clients/01-agent-clients.md](docs/en/clients/01-agent-clients.md) |

**리포 통계 (조회 시점 기준)**: ⭐ 25,793 · 🍴 3,519 forks · 언어 TypeScript · 라이선스 MIT · 최초 생성 2026-04

---

## 1. 이게 뭔데?

한 줄 요약: **"AI 무료 쿠폰 34장을 대신 관리해주는 비서"**

- 무료 제공사 **34개**, 무료 모델 엔드포인트 **635개** (chat 584 / embeddings 41 / transcription 7 / video 3)
- 모델 패밀리 **474개**, 합계 **월 약 74억 토큰** 규모의 무료 추론 용량
- 전부 **`http://localhost:3001/v1`** 하나로 통일 (OpenAI 호환)

### 동작 흐름

```
오빠: "AI야, 이거 해줘"
   ↓
🤖 라우터: (34곳 상태 확인) "지금은 구글이 제일 한가하네!"
   ↓ 구글 호출 → 429(한도초과) 응답
🤖 라우터: 해당 키 쿨다운 → 다음 순위 모델로 자동 폴백
   ↓
✅ 응답 반환 (호출자는 무슨 일이 있었는지 모름)
```

### 비유

> 통신사 34개의 무료 데이터를 한 유심에 몰아넣은 느낌.
> SKT 다 쓰면 KT로, 그것도 다 쓰면 LG로 — 근데 폰에는 유심 하나만 꽂혀 있음.

---

## 2. 폴더 구조 (직접 확인한 내용)

npm workspaces 기반 모노레포입니다.

| 폴더 | 정체 | 핵심 내용 |
| --- | --- | --- |
| `server/` | 🧠 핵심 두뇌 | Express 5 + TypeScript + SQLite |
| `client/` | 🎨 관리자 대시보드 | React 19 + Tailwind 4 + shadcn |
| `cli/` | ⚡ 자동 세팅 도구 | `npx freellmapi setup-*` |
| `desktop/` | 💻 데스크톱 앱 | Electron (Win/macOS 인스톨러) |
| `shared/` | 공용 타입 | `types.ts` |
| `docs/` | 📚 문서 | 영어 + 중국어 풀세트 |
| `docker/`, `Dockerfile`, `docker-compose.yml` | 🐳 배포 | |
| `examples/` | 예제 | fetch-relay-worker |

### `server/src` 내부

**`providers/` — 제공사 어댑터 15개**
`google.ts`, `cohere.ts`, `cloudflare.ts`, `zhipu.ts`, `modelscope.ts`, `pollinations.ts`,
`aihorde.ts`, `electronhub.ts`, `experiential.ts`, `router9.ts`, `sail.ts`, `septor.ts`,
`openai-compat.ts`(범용), `base.ts`, `index.ts`

**`routes/` — 엔드포인트 27개 (하이라이트)**

| 파일 | 역할 |
| --- | --- |
| `anthropic.ts` | `/v1/messages` — **Claude Code가 그대로 붙음** |
| `gemini.ts` | `/v1beta` — Gemini CLI 네이티브 |
| `responses.ts` | `/v1/responses` — Codex CLI |
| `ollama.ts` | Ollama 에뮬레이션 (Zed, JetBrains) |
| `media.ts` | 이미지 / 영상 / 음성 생성 |
| `embeddings.ts` | 임베딩 (RAG용) |
| `mcp.ts` | **MCP 서버** (에이전트 내성 조회) |
| `analytics.ts`, `logs.ts`, `backups.ts` | 운영 도구 |
| `premium.ts` | 라이선스 키 활성화 |

**`services/` — 40개+ (핵심만)**

| 파일 | 역할 |
| --- | --- |
| `router.ts` + `scoring.ts` | 속도·성능·신뢰도 점수로 모델 순위 결정 |
| `ratelimit.ts` + `provider-quota.ts` | `(제공사, 모델, 키)` 단위 RPM/RPD/TPM/TPD 추적 |
| `cooldown-probe.ts` | 한도 초과 키 격리 & 복구 확인 |
| `fusion.ts` | **여러 모델 병렬 호출 → 심판 모델이 답 합성** |
| `compression/` | 프롬프트 압축 (토큰 절감) |
| `cache.ts` | 응답 캐시 (스트리밍 포함) |
| `catalog-sync.ts` | 서명된 모델 카탈로그 자동 동기화 |
| `context-handoff.ts` | 모델 전환 시 맥락 인계 (30분 sticky session) |

**보안**: 제공사 키는 SQLite에 **AES-256-GCM 암호화** 저장, 요청 시에만 메모리에서 복호화.

---

## 3. 설치 및 사용법

### A. 데스크톱 앱 (가장 쉬움 — 추천 ⭐)

[Releases](https://github.com/tashfeenahmed/freellmapi/releases/latest)에서 `.exe`(Windows) / `.dmg`(macOS) 받아서 설치. 트레이 아이콘에서 키 복사까지 가능.

### B. Docker 한 줄

```bash
curl -fsSL https://freellmapi.co/install.sh | bash
```

`~/freellmapi` 생성 + 암호화 키 자동 생성 + 컨테이너 기동. 재실행해도 `.env` 보존되고 `:latest`로 업데이트됨.

### C. Docker Compose

```bash
git clone https://github.com/bmshin94/freellmapi.git
cd freellmapi
ENCRYPTION_KEY="$(openssl rand -hex 32)"
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$ENCRYPTION_KEY" > .env
docker compose up -d
```

### D. 로컬 개발 (소스 직접 실행)

```bash
git clone https://github.com/bmshin94/freellmapi.git
cd freellmapi
npm install
ENCRYPTION_KEY="$(node -e 'console.log(require("crypto").randomBytes(32).toString("hex"))')"
printf "ENCRYPTION_KEY=%s\nPORT=3001\n" "$ENCRYPTION_KEY" > .env
npm run dev        # server + client 동시 실행
```

> Node **20.18 이상** 필요 (`.nvmrc` 참고)

### 설치 후 순서

1. http://localhost:3001 접속
2. **Keys** 페이지 → 제공사 키 입력
3. **Fallback Chain** 페이지 → 드래그로 우선순위 정렬
4. **Keys** 헤더에서 통합 키(`freellmapi-…`) 복사
5. 앱에서 base URL을 `http://localhost:3001/v1` 로 지정

### 코딩 에이전트 연결 (명령어 한 줄)

```bash
npx freellmapi setup-claude --url http://localhost:3001 --api-key <통합키>
```

지원: `setup-claude`, `setup-codex`, `setup-cline`, `setup-continue`, `setup-aider`,
`setup-opencode`, `setup-goose`, `setup-qwen`, `setup-roo`, `setup-kilo`, `setup-crush`,
`setup-dsh`, `setup-mimo`, `setup-cursor`, `setup-generic`

**안전장치**: 기존 설정 병합(덮어쓰기 X) + 타임스탬프 백업 + `--dry-run` 미리보기
**`launch` / `launch-codex`**: 자격증명을 파일에 안 쓰고 자식 프로세스 환경변수로만 주입

### ⚠️ 설치 시 주의

| | |
| --- | --- |
| 🔑 | `ENCRYPTION_KEY` 분실 시 저장된 키 전부 복호화 실패 (키 교체는 재암호화 절차 필요) |
| 🔒 | 기본 바인딩은 `127.0.0.1`. 단일 사용자 설계이므로 인터넷 노출 금지 |
| 🌐 | Docker 안에서 호스트 프록시 쓸 땐 `host.docker.internal` 사용 |

---

## 4. 플러그인? 스킬? MCP?

### 정답: 셋 다 아니고 **"독립 서버(게이트웨이)"**

| 종류 | 비유 | 해당? |
| --- | --- | :---: |
| 플러그인 | 자동차에 끼우는 부품 | ❌ |
| 스킬 | Claude에게 주는 설명서 | ❌ |
| MCP 서버 | Claude가 쓰는 도구 상자 | 🟡 일부 |
| **게이트웨이** | **길 자체를 바꾸는 톨게이트** | ✅ |

```
[평소]      Claude Code ──────────────→ Anthropic 서버 (유료)
[이거 쓰면] Claude Code ──→ FreeLLMAPI ──→ 구글/Groq/Mistral… (무료)
                            (Anthropic 형식으로 응답)
```

### 다만 MCP 서버도 "내장" 되어 있음

`server/src/routes/mcp.ts` — MCP SDK 없이 직접 구현한 stateless JSON-RPC (프로토콜 `2025-06-18`). 툴 7개:

| 툴 | 역할 |
| --- | --- |
| `list_models` | 현재 사용 가능한 모델 목록 (컨텍스트·툴 지원·파라미터 포함) |
| `provider_health` | 제공사별 키 상태 (healthy/rate_limited/invalid/error) |
| `usage_summary` | 요청·토큰 총계, 성공률, 상위 모델 |
| `routing_info` | 활성 라우팅 전략 + 상위 점수 모델 |
| `set_routing_strategy` | **라우팅 전략 변경** (유일한 조작 툴) |
| `cache_stats` | 캐시로 절약한 토큰 |
| `compression_stats` | 압축으로 절약한 토큰 |

> 즉 **"본체는 게이트웨이, MCP는 계기판"**

---

## 5. API 토큰은 어떻게?

토큰이 **2단계**로 분리되어 있습니다.

### 🟢 1단계: 제공사 키 (직접 발급 — 전부 무료)

| 제공사 | 발급처 | 난이도 |
| --- | --- | --- |
| **Google (Gemini)** | aistudio.google.com | 제일 쉬움 |
| **Groq** | console.groq.com | 쉬움 (매우 빠름) |
| **Cerebras** | cloud.cerebras.ai | 보통 |
| Mistral / Cohere / NVIDIA / OpenRouter / Cloudflare / Z.ai | 각 사이트 | 보통 |
| ModelScope | 알리윈 중국 실명인증 필요 | 어려움 |

> 💡 **구글 + Groq 2개만 넣어도 충분히 잘 돌아감.** 최소 1개는 있어야 모델이 잡힙니다.

### 🔵 2단계: 통합 키 (자동 생성)

`server/src/db/index.ts:233`

```js
const key = `freellmapi-${crypto.randomBytes(24).toString('hex')}`;
```

```
내 앱 ──[통합키 1개]──→ FreeLLMAPI ──[진짜 키들]──→ 각 제공사
            ↑ 이것만 노출              ↑ AES-256-GCM 암호화 보관
```

---

## 6. 왜 깃허브에서 유명할까?

```
⭐ Stars   25,793
🍴 Forks    3,519
📅 생성     2026년 4월 (약 5개월 만의 수치)
📝 언어     TypeScript
📜 라이선스 MIT
```

1. **통증이 명확** — "AI 쓰고 싶은데 비용 부담" 은 전 세계 개발자 공통 고민
2. **숫자가 강렬** — "월 74억 토큰 / 34 제공사 / 635 엔드포인트 / 엔드포인트 하나"
3. **Claude Code·Cursor가 무료로 돌아감** — 지금 가장 핫한 지점
4. **완성도** — React 19 대시보드(페이지 16개), Electron 앱, 모바일 앱, 60개 언어, 영·중 문서, CI/테스트/마이그레이션/백업
5. **로컬 우선 + 보안** — "내 키를 남의 서버에 안 준다"
6. **정직함** — README에 "프로덕션용 아님" 명시 + 제공사별 약관 검토 문서화

---

## 7. 로컬 에이전트 구축에 도움될까? → **YES (조건부)**

### ✅ 좋은 점

- **실험 비용 0원** — 에이전트는 1회 실행에 LLM을 10~50번 호출
- **잡일 모델 분리** — 어려운 판단은 유료, 요약·분류·파싱은 무료 → 비용 70~80% 절감 가능
- **에이전트 필수 기능 완비** — Tool calling(텍스트 툴콜을 실제 `tool_calls`로 복구), `response_format`, embeddings, streaming, sticky session(30분), context handoff
- **Fusion** — 여러 모델 병렬 호출 후 합성 (무료라서 가능한 사치)
- **자동화/크론 작업** — 매일 도는 요약 봇, 수집기 등

### ⚠️ 한계

| 리스크 | 설명 |
| --- | --- |
| 🐌 속도 편차 | 호출이 많은 에이전트에서 지연이 누적 |
| 🧠 모델 수준 | 복잡한 다단계 추론은 무료 모델이 헤맴 |
| 🌙 시간대 품질 | 상위 모델 일일 한도 소진 → 하위 모델로 밀림 (UTC 자정 리셋) |
| 🔧 Tool calling 편차 | 제공사별 구현 수준 상이 |

### 권장 조합

```
🔴 메인 에이전트 두뇌 / 배포본  → 유료 API
🟢 서브 작업 / 개발 중 테스트 / 배치 → FreeLLMAPI
```

---

## 8. React / PHP로 만들 수 있어?

### 프론트는 **이미 React 19**

```json
"react": "^19.2.4",  "tailwindcss": "^4.2.2",
"@tanstack/react-query", "recharts", "shadcn", "@base-ui/react",
"react-router-dom": "^7", "@dnd-kit/*"   // 체인 드래그 정렬
```

백엔드: `express@5` + TypeScript + SQLite + `zod` / `helmet` / `undici` / `sharp` / `multer`

### ⚛️ React → 완전 가능

```jsx
import OpenAI from 'openai';

const ai = new OpenAI({
  baseURL: 'http://localhost:3001/v1',
  apiKey: 'freellmapi-...',
  dangerouslyAllowBrowser: true,   // ⚠️ 로컬 실험용만
});

const res = await ai.chat.completions.create({
  model: 'auto',                   // 라우터가 알아서 선택
  messages: [{ role: 'user', content: '안녕!' }],
});
```

### 🐘 PHP → **쓰는 건 쉬움 / 서버를 재구현하는 건 비추**

```php
<?php
$ch = curl_init('http://localhost:3001/v1/chat/completions');
curl_setopt_array($ch, [
  CURLOPT_POST => true,
  CURLOPT_HTTPHEADER => [
    'Content-Type: application/json',
    'Authorization: Bearer freellmapi-여기에통합키',
  ],
  CURLOPT_POSTFIELDS => json_encode([
    'model' => 'auto',
    'messages' => [['role' => 'user', 'content' => '안녕!']],
  ]),
  CURLOPT_RETURNTRANSFER => true,
]);
$res = json_decode(curl_exec($ch), true);
echo $res['choices'][0]['message']['content'];
```

서버 자체를 PHP로 재구현하기 어려운 이유:

| 필요 기능 | PHP 사정 |
| --- | --- |
| SSE 스트리밍 중계 | PHP-FPM 구조와 상성이 나쁨 |
| 백그라운드 작업 (카탈로그 동기화, 키 헬스체크, 쿨다운) | 요청 종료 시 프로세스 소멸 → 크론/워커 별도 필요 |
| 인메모리 rate limit 카운터 | 요청마다 초기화 → Redis 필수 |
| 병렬 호출 (Fusion) | `curl_multi`로 가능하나 복잡 |

> Swoole / ReactPHP / Laravel Octane 이면 가능. 하지만 **Node 서버는 그대로 두고 PHP는 클라이언트로 붙이는 게 훨씬 편함.**

---

## 9. 수익화 아이디어

### 📚 먼저: 원저자의 수익 모델 (교재)

`routes/premium.ts` + `services/catalog-sync.ts` 분석 결과:

```
📦 소프트웨어(라우터 전체) → MIT, 100% 무료, 영원히
💎 "살아있는 모델 카탈로그" → $19/년 또는 $49 평생 (Stripe)
```

- 무료: 월간 스냅샷 (모델이 30일 뒤 도착, 현재 약 303개 뒤처짐)
- 유료: 당일 반영, `fla_` 키 하나로 기기 무제한
- **기능을 잠그지 않고 "늦게 도착"시킴** → 반감이 적음
- Ed25519 서명 + 공개키 고정으로 위조 카탈로그 차단
- 카탈로그 서버는 프롬프트·완성·제공사 키를 절대 안 봄

> **핵심 공식: 오픈소스로 신뢰 확보 → 반복 노동/편의를 구독으로 판매**

### 🚧 리스크 지도

**🔴 절대 금지**

| 금지 사항 | 이유 |
| --- | --- |
| 무료 티어 API 재판매 | 제공사 약관 정면 위반 → 계정 영구 정지 |
| 계정 다중 생성으로 키 확보 | 어뷰징 탐지 |
| "무료 GPT" 공개 웹서비스 | 트래픽 몰리면 즉시 차단 |
| 내 키로 타인 트래픽 처리 | 내 계정이 죽음 |

**🟡 주의** — 회사 업무 사용 (무료 티어는 상업적 이용 제한이 흔함)

**🟢 안전** — 개인 비용 절감 / BYOK 소프트웨어 판매 / 지식·콘텐츠·교육 / 여기서 배운 기술로 만든 유료 서비스

> 원저자도 그래서 "API"가 아니라 "카탈로그 구독"을 판매함.

### 💰 아이디어 8개

| # | 아이디어 | 난이도 | 시작까지 | 예상 월수익(가정) | 리스크 | 추천 |
| --- | --- | :---: | :---: | --- | :---: | :---: |
| 8 | 💸 비용 절감 (= 확실한 순수익) | ⭐ | 오늘 | ₩100,000 절감 | 🟢 | ⭐⭐⭐⭐⭐ |
| 4 | 🔧 설치·세팅 대행 | ⭐ | 1주 | ₩30~100만 | 🟢 | ⭐⭐⭐⭐ |
| 2 | 📺 한국어 콘텐츠·교육 | ⭐⭐ | 2주 | ₩10~500만 | 🟢 | ⭐⭐⭐⭐⭐ |
| 5 | 🎁 오픈소스 기여 → 커리어 | ⭐⭐ | 1주 | 간접 | 🟢 | ⭐⭐⭐⭐⭐ |
| 1 | 🥇 BYOK 앱 판매 | ⭐⭐⭐ | 1개월 | ₩50~300만 | 🟢 | ⭐⭐⭐⭐⭐ |
| 6 | 🤖 니치 자동화 봇 | ⭐⭐⭐ | 2주 | ₩20~100만 | 🟡 | ⭐⭐⭐ |
| 7 | 📡 카탈로그형 구독 | ⭐⭐⭐⭐ | 2개월 | ₩30~200만 | 🟢 | ⭐⭐⭐⭐ |
| 3 | 💼 유료 API용 라우터 SaaS (B2B) | ⭐⭐⭐⭐⭐ | 6개월 | ₩500만+ | 🟢 | ⭐⭐⭐⭐ |

> 수익 수치는 **가정치**이며 보장이 아닙니다.

#### 1. BYOK 앱 판매 (최고 추천)

> "소프트웨어를 팔고, 연료(API 키)는 손님이 넣는다"

- 서버 비용 0원 (전부 로컬 실행), 약관 리스크는 사용자 본인 키에 귀속
- 후보: PDF/논문 요약기(₩19,000 일회성), 블로그 초안 생성기(₩9,900/월), 문서 일괄 번역기(₩29,000), 유튜브 자막→블로그(₩9,900/월), 이메일 답장 도우미, 회의록 정리기
- 재활용할 코드: `desktop/`(Electron 껍데기·빌드 설정), `client/src/`(React+shadcn UI), `cli/src/`(설정 병합·백업 로직), `providers/openai-compat.ts`, `services/ratelimit.ts`

#### 2. 한국어 콘텐츠·교육

- 공식 문서가 영어·중국어뿐 → **한국어 자료 거의 없음 = 블루오션**
- 유튜브("Claude Code 구독료 0원으로 쓰는 법"), 블로그/뉴스레터, 전자책(₩15,000~29,000), 인프런 강의(₩55,000~99,000)

#### 3. 유료 API용 스마트 라우터 SaaS (B2B)

- 같은 기술을 **유료 제공사 대상**으로 → 100% 합법
- 파는 가치: 비용 최적화 라우팅 / 장애 자동 폴백 / 부서별 사용량·비용 대시보드 / 키 중앙 관리
- 배울 코드: `services/router.ts`, `scoring.ts`, `ratelimit.ts`, `cooldown-probe.ts`, `cache.ts`, `routes/analytics.ts`, `services/compression/`
- 가격: 스타터 ₩50,000/월 · 팀 ₩200,000/월 · 기업 ₩500,000+/월 (또는 절감액의 10~20%)
- 경쟁: LiteLLM, Portkey 등

#### 4. 설치·세팅 대행

| 서비스 | 가격 |
| --- | --- |
| 원격 설치 대행 (1:1 화면공유) | ₩30,000~50,000 |
| 라즈베리파이 세팅 완제품 | ₩150,000 (기기 포함) |
| 소규모 기업 AI 도입 컨설팅 | ₩300,000~ |
| 월간 유지보수 | ₩30,000/월 |

판매처: 크몽 / 숨고 / 탈잉 / 오픈채팅
⚠️ **손님 API 키는 반드시 손님이 직접 입력**하게 할 것

#### 5. 오픈소스 기여 → 커리어

- 한국어 번역 (`client/src/i18n/`, `docs/ko/`) — 난이도 ⭐
- 새 제공사 어댑터 추가 (`server/src/providers/` 에 파일 하나) — 난이도 ⭐⭐⭐
- 버그 수정 / 문서 개선 (열린 이슈 51개)
- 효과: ⭐25K 프로젝트 컨트리뷰터 이력 → 프리랜서 단가·이직에 반영

#### 6. 니치 자동화 봇

- 조건: **소규모 유지**, "API 제공"이 아니라 "결과물 제공"으로 포지셔닝, 커지면 즉시 유료 API로 전환
- 예: 업계 뉴스 요약 뉴스레터(₩5,000/월), 상품설명 생성(건당 ₩500), 리뷰 감성분석 리포트(₩100,000/월)

#### 7. 카탈로그형 구독 (원저자 모델 응용)

- 한국 AI 서비스 가격/스펙 DB, AI 규제 업데이트 피드, 개발자 무료 티어 총정리, MCP 서버 큐레이션
- `catalog-sync.ts` 구조(서명 + 무료는 월간/유료는 실시간)를 그대로 참고 가능 (MIT)

#### 8. 비용 절감

```
Claude Pro $20 + Cursor Pro $20 + API 실험 $30 ≈ 월 ₩98,000 → 연 ₩1,176,000
개발·테스트 단계를 FreeLLMAPI로 대체하면 그만큼이 순수익
```

### 🏆 추천 전략: "3단 로켓"

```
1단 (0~1개월)  비용절감 + 설치대행 + 콘텐츠 시작 → 현금흐름 & 시장 반응
2단 (1~3개월)  BYOK 앱 출시 → 콘텐츠로 쌓은 신뢰를 첫 고객으로 전환
3단 (3~12개월) 전자책/강의 + 카탈로그 구독 → 자동 수익 파이프라인
```

```
콘텐츠(무료) → 신뢰 → 앱(유료) 판매 → 후기 → 다시 콘텐츠
     ↑______________피드백 루프______________|
```

### 📅 30일 실행 플랜

**1주차 — 기반**
- [ ] FreeLLMAPI 설치 (데스크톱 앱 또는 Docker)
- [ ] 구글 + Groq 키 발급 후 등록
- [ ] `npx freellmapi setup-claude` 로 Claude Code 연결
- [ ] 일주일 실사용하며 체감 기록
- [ ] 유튜브/블로그 계정 개설

**2주차 — 첫 콘텐츠 & 첫 수익**
- [ ] 블로그 글 1편 작성
- [ ] 유튜브 영상 1개 (설치 과정 화면녹화)
- [ ] 크몽에 "AI 세팅 대행" 등록
- [ ] 반응 보고 수요 파악

**3주차 — 앱 착수**
- [ ] BYOK 앱 아이템 확정 (반응 좋았던 주제)
- [ ] `client/` React 구조 참고해 프로토타입
- [ ] 결제 연동 (Gumroad / Lemon Squeezy)
- [ ] 랜딩페이지 제작

**4주차 — 출시 & 확산**
- [ ] 베타 출시 (초기 50명 반값)
- [ ] 한국어 번역 PR
- [ ] 커뮤니티 공유 (GeekNews, 디스콰이엇, 클리앙, r/LocalLLaMA)
- [ ] 후기 수집 → 다음 콘텐츠 소재

### 💳 실무 팁

**결제 플랫폼**

| 플랫폼 | 장점 | 수수료 |
| --- | --- | --- |
| Lemon Squeezy | 해외 판매 + 세금 자동 처리(MoR) | ~5% |
| Gumroad | 가장 간단, 전자책에 적합 | ~10% |
| Paddle | SaaS 구독에 강함 | ~5% |
| 크몽 / 인프런 | 국내 + 자체 트래픽 | 15~20% |

**가격 원칙**
- 일회성은 $19~49 (₩25,000~65,000) — 고민 없이 결제하는 구간
- 구독은 ₩9,900 (심리적 저항선 아래)
- 평생 라이선스 = 연간 × 2.5배 (원저자: $19/년 vs $49 평생)
- 너무 싸면 오히려 가치를 의심받음

**마케팅 채널 (무료)**
- 국내: GeekNews, 디스콰이엇, 클리앙, 오픈카톡, 커리어리
- 해외: Reddit(r/LocalLLaMA), Hacker News, X, Product Hunt

**법적 체크리스트**
- [ ] 통신판매업 신고 (연매출 기준 확인)
- [ ] 종합소득세 신고 준비
- [ ] 이용약관 / 환불정책 명시
- [ ] **"본인 API 키 필요"** 를 판매 페이지에 명확히 표시
- [ ] 제공사 약관 위반 책임 면책 조항

---

## 10. 한계와 면책 (반드시 읽기)

원본 README `## Disclaimer` 요약:

> **이 프로젝트는 개인 실험·학습용이며 프로덕션용이 아님.**
> 각 제공사와의 관계는 가입 시 동의한 약관의 지배를 받으며, 프록시를 거쳐도 그 약관은 그대로 적용됨. 준수 책임은 사용자에게 있음.

| 한계 | 설명 |
| --- | --- |
| 🍱 프론티어 모델 없음 | 최상위 모델은 무료 티어에 없음 |
| ⏱️ SLA 없음 | 지연시간 변동 큼 |
| 🌙 시간대별 품질 저하 | 상위 모델 일일 한도 소진 → UTC 자정 리셋 |
| 🔒 단일 사용자 설계 | 인터넷 노출 금지, 통합 키 하나로만 보호됨 |
| 💎 Premium | 기능 제한이 아니라 **카탈로그 반영 속도** 차이 |

제공사별 약관 검토(2026년 5월 기준)는
[docs/en/architecture/00-high-level-index.md#terms-of-service-review](docs/en/architecture/00-high-level-index.md#terms-of-service-review) 참고.

---

## 11. 다음 단계 로드맵

```
1주차   🐳 설치 → 구글 + Groq 키 등록
2주차   ⚡ npx freellmapi setup-claude 로 Claude Code 연결
3주차   🔍 server/src/services/router.ts, scoring.ts 정독
4주차   ⚛️ React 앱 하나 만들어 연결
5주차+  💰 BYOK 앱 제작 → 수익화 도전
```

---

*이 문서는 리포지토리 코드와 문서를 직접 확인해 정리한 한국어 요약본입니다.*
*원본: https://github.com/tashfeenahmed/freellmapi (MIT License)*
