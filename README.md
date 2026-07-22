# PDF to PPTX Converter (v2)

## 서비스 정보

> 최종 확인: 2026-07-22

| 항목 | 값 |
|---|---|
| 서비스 URL | https://pdf2pptx-v2.pages.dev/ |
| 상태 | ⚠️ **기능 불능** — 번들에 박힌 Gemini 키가 이미 폐기되어 AI 호출이 실패한다 |
| 위치 | ⚗️ **실험 버전** — AI Studio 기반으로 만든 시험작이며 상시 사용하지 않는다 |
| 호스팅 | Cloudflare Pages — 프로젝트명 `pdf2pptx-v2`, GitHub `ahnbu/pdf2pptx_v2` 연결 |
| DB | ❌ 없음 (브라우저 내 변환) |
| 인증 | ❌ 없음 |
| 외부 API | Google Gemini (`@google/genai`) |
| 배포 방식 | CF Pages 자동 빌드 (레포에 설정 파일 없음 — 대시보드 연결) |
| 필요 환경변수 | `GEMINI_API_KEY`, `VITE_GOOGLE_API_KEY` |

> [!WARNING]
> **번들에 박힌 Gemini 키가 이미 폐기되어 AI 기능이 동작하지 않는다.** (확인: 2026-07-22)
>
> `https://pdf2pptx-v2.pages.dev/assets/index-*.js`에 키(식별자 `AIzaSyD8i0`로 시작, `vpBs`로 끝남)가 평문으로 들어 있었으나, Google Generative Language API에 해당 키로 실호출한 결과 `HTTP 400 · "API key not valid" · API_KEY_INVALID`가 반환되었다.
>
> 즉 **유출 리스크는 이미 해소된 상태**다(키가 무효라 악용 불가). 대신 서비스가 동작하지 않는다.
>
> 로컬 `.env`·`color_prompt/.env`·`link-cleaner/.env.local`에도 같은 폐기된 키가 남아 있다. 현행 유효 키는 `~/.env`에 있다.
>
> **구조적 문제는 남아 있다.** `VITE_GOOGLE_API_KEY`를 클라이언트에서 직접 쓰는 한, 유효한 키로 재배포하면 그 키가 다시 노출된다. 복구할 경우 **서버 경유 호출(CF Pages Functions)로 전환**하거나, 최소한 이 프로젝트 전용 키를 별도 발급해 영향을 격리할 것.
>
> 실험 버전이므로 **복구하지 않고 두는 것도 합리적 선택**이다.

### ⚠️ 활성 서비스로 전환 시 선결 과제

이 프로젝트를 상시 사용 서비스로 승격하려면 아래를 먼저 해결해야 한다. **키만 교체하고 배포하면 그 키가 즉시 공개된다.**

| # | 과제 | 현재 상태 | 조치 |
|---|---|---|---|
| 1 | **Gemini 키 클라이언트 노출** | `VITE_GOOGLE_API_KEY`가 번들에 평문 삽입됨 | CF Pages Functions로 서버 경유 호출 전환. `color_prompt`의 `functions/api/generate.js`가 참고 구현 |
| 2 | 유효 키 재주입 | 폐기된 키가 박혀 있어 AI 호출 실패 | 1번 완료 후 CF 대시보드 환경변수로 주입 (레포에 두지 말 것) |
| 3 | 키 격리 | 4개 프로젝트가 동일 키 공유 | 이 프로젝트 전용 키 발급 |
| 4 | 테스트 부재 | 테스트 코드 0건 | 최소 배포 후 스모크 테스트. 이번 키 폐기도 사전에 잡히지 않았다 |

> 1번을 건너뛰면 나머지는 의미가 없다. **순서를 지킬 것.**

> ℹ️ **혼동 주의**: `pdf2pptx.pages.dev`(v2 없음)는 **별개 서비스**다. 그쪽은 `notebooklm_logo_cleaner` 레포를 서빙하며, 기능이 비슷하지만 다른 프로젝트다. 둘은 의도적으로 병존한다.

---

본 프로젝트는 PDF 파일을 분석하여 편집 가능한 파워포인트(PPTX) 파일로 변환하는 웹 애플리케이션입니다. [Google AI Studio](https://aistudio.google.com/) 환경에서 개발된 코드를 로컬 및 배포 환경에 최적화하여 구성되었습니다.

## 🛠 시스템 아키텍처 및 상세 로직

본 프로젝트인 **PDF to PPTX 변환기**의 전체 작업 로직과 기술 스택, 그리고 데이터 흐름을 단계별로 정리합니다.

### 🛠 기술 스택 (Tech Stack)

- **Frontend**: React (v19), TypeScript, Tailwind CSS (Styling), Lucide React (Icons)
- **Build/Serve**: Vite, Cloudflare Pages (Deployment)
- **AI Model**: **Google Gemini 3.0 Pro Preview** (Vision-to-JSON 분석 최적화)
- **Libraries**:
  - `pdfjs-dist`: PDF 파싱 및 이미지 렌더링
  - `pptxgenjs`: PPTX 파일 생성
- **Local Backend (Optional)**: Node.js, Express (로컬 파일 저장용)

### 🔄 데이터 흐름 및 단계별 상세 로직 (Workflow)

#### 단계 1: 파일 업로드 및 준비 (Upload & Prep)

사용자가 파일을 선택하고 변환 설정을 조정합니다.

- **Input**: `.pdf` 파일, 동시 처리 개수(`batchSize`) 설정 (1~5장 병렬 처리 지원)
- **작업**: 파일 유효성 검사 및 브랜딩 UI 표시 (WandSparkles 아이콘 및 커스텀 테마)
- **주요 기술**: `React State`, `Dropzone.tsx`

#### 단계 2: PDF 이미지 변환 (PDF to Image)

PDF의 각 페이지를 고해상도 이미지로 변환합니다.

- **Input**: PDF Binary
- **작업**: `pdf.js`를 이용해 각 페이지를 2.0 스케일(고화질)의 Base64 이미지로 렌더링
- **Output**: `string[]` (Base64 이미지 배열)
- **주요 기술**: `pdfjs-dist`, `pdfService.ts`

#### 단계 3: AI 병렬 레이아웃 분석 (AI Analysis - Parallel Batch)

이미지를 분석하여 텍스트와 객체 정보를 추출합니다.

- **Input**: Base64 이미지 조각들
- **작업**:
  - 사용자가 선택한 `batchSize`만큼 이미지를 버킷화하여 동시 요청.
  - **Gemini 3.0 Pro Preview** 모델을 사용하여 이미지 내 배경색, 텍스트(내용, 좌표, 폰트, 색상), 도형/이미지 요소를 추출.
  - `High Resolution` 모드 활성화로 미세한 텍스트 및 로고 인식률 극대화.
- **Output**: `SlideData[]` (구조화된 레이아웃 데이터)
- **주요 기술**: `@google/genai`, `geminiService.ts`

#### 단계 4: 슬라이드 미리보기 및 편집 (Preview & Edit)

변환 전 AI의 분석 결과를 확인하고 수정합니다.

- **Input**: `SlideData[]`
- **작업**: 실시간 오버레이 미리보기 및 텍스트 수정 기능 제공.
- **주요 기술**: `SlidePreview.tsx`, `ElementEditor`

#### 단계 5: PPTX 생성 및 다운로드 (Generation & Delivery)

최종 PPTX 파일을 생성하고 전달합니다.

- **Input**: (수정된) `SlideData[]`
- **작업**:
  - 원본 파일명을 보존한 `{원본파일명}_변환.pptx` 형태로 브라우저 자동 다운로드 실행.
  - `pptxgenjs` 기반의 객체 지향 PPT 생성.
- **Output**: 파워포인트 파일
- **주요 기술**: `pptxgenjs`, `pptxService.ts`, `App.tsx`

#### 단계 6: 로컬 저장 (Local Archive - Optional)

- **작업**: `localhost` 환경(개발 모드)인 경우에만 `server.js`를 통해 `output/` 폴더에 결과물 아카이빙.

### 📊 Input/Output 상세 스펙 요약

| 구분               | 데이터 종류 | 스펙/포맷                                                           |
| :----------------- | :---------- | :------------------------------------------------------------------ |
| **Input**          | 원본 파일   | .pdf (비트맵 이미지가 포함된 슬라이드 형태 최적)                    |
| **Processing**     | 분석 데이터 | JSON (index, backgroundColor, elements[{x, y, w, h, content, ...}]) |
| **Output (Main)**  | 변환 파일   | .pptx (편집 가능한 {파일명}\_변환.pptx)                             |
| **Output (Local)** | 아카이브    | output/ 폴더 (PPTX, JSON, 개별 이미지 포함)                         |

---

### ⏳ 슬라이드 1장 기준 작업 비중 (지연 시간 기준)

| 단계           | 작업 이름          | 시간 비중 | 소요 시간(예상) | 주요 특징                                |
| :------------- | :----------------- | :-------: | :-------------: | :--------------------------------------- |
| **Stage 2**    | PDF -> 이미지 변환 |  약 10%   |    0.5 ~ 1초    | 브라우저 성능에 비례                     |
| **Stage 3**    | Gemini AI 분석     |  약 85%   |    3 ~ 10초     | **병렬 처리(1~5개)로 총 시간 단축 가능** |
| **Stage 5**    | PPTX 파일 생성     |   약 4%   |   0.1 ~ 0.3초   | 순수 로컬 계산                           |
| **Stage 1, 6** | 기타 (업로드/저장) |   약 1%   |   0.1초 미만    | 환경별 조건부 실행                       |

---

## 🤖 AI Prompt Engineering (AI 프롬프트 엔지니어링)

본 프로젝트의 핵심인 Stage 3(AI 분석)에서 **Gemini 3.0 Pro Preview** 모델에 전달되는 프롬프트 설계 구조입니다.

### 1. 시스템 인스트럭션 (System Instruction)

#### [English Version]

You are an expert presentation layout parser. Your goal is to analyze an image of a presentation slide and extract a structured JSON representation.

- **Deconstruct Layers**: Identify the background color first, then shapes as containers, and finally text elements.
- **Visual Asset Preservation**: Identify photos, icons, and logos as `image` type to preserve visual fidelity.
- **Text Granularity**: Do not group distinct text blocks; separate titles, subtitles, and body text.
- **Properties & Estimation**:
  - Coordinates: Use percentage (0-100) for `x, y, w, h`.
  - Font Size: Estimate in points (pt) (e.g., Title: 40-60pt, Body: 14-20pt).
  - Colors: Return 6-digit Hex codes (e.g., #FF0000).
  - Typography: Identify `bold` and `align` (left, center, right).

#### [한국어 번역 버전]

당신은 전문적인 프레젠테이션 레이아웃 파서입니다. 슬라이드 이미지를 분석하여 구조화된 JSON 표현을 추출하는 것이 목표입니다.

- **레이어 분해**: 배경색을 먼저 식별한 다음, 컨테이너 역할을 하는 도형을 찾고, 마지막으로 그 위의 텍스트 요소를 찾습니다.
- **시각 자산 보존**: 시각적 완성도를 유지하기 위해 사진, 아이콘, 로고를 `image` 타입으로 식별합니다.
- **텍스트 세분화**: 별개의 텍스트 블록을 하나로 묶지 마십시오. 제목, 부제목, 본문을 각각 분리합니다.
- **속성 및 추정**:
  - 좌표: `x, y, w, h`에 대해 백분율(0-100) 단위를 사용합니다.
  - 폰트 크기: 시각적 규모에 따라 포인트(pt) 단위로 추정합니다 (예: 제목 40-60pt, 본문 14-20pt).
  - 색상: 6자리 Hex 코드(예: #FF0000)를 반환합니다.
  - 타이포그래피: 굵게(`bold`) 처리 및 정렬(`align`: left, center, right) 상태를 식별합니다.

### 2. 사용자 요청 프롬프트 (User Request Prompt)

#### [English Version]

> "Analyze this slide image. Extract layout, text, shapes, and IMAGES into a precise JSON structure. Prioritize preserving diagrams and photos as 'image' type."

#### [한국어 번역 버전]

> "이 슬라이드 이미지를 분석하십시오. 레이아웃, 텍스트, 도형 및 이미지를 정밀한 JSON 구조로 추출하십시오. 다이어그램과 사진은 'image' 타입으로 보존하는 것을 우선순위로 합니다."

---

## 🚀 Run Locally

**Prerequisites:** Node.js

1. **의존성 설치**:
   ```bash
   npm install
   ```
2. **환경 변수 설정**:
   `.env` 파일의 `VITE_GOOGLE_API_KEY` 항목에 Gemini API 키를 입력합니다.
3. **앱 실행**:
   ```bash
   npm run dev
   ```

---

> **Reference**: 본 프로젝트의 AI 모델 구성 및 테스트는 [Google AI Studio](https://aistudio.google.com/)를 활용하였습니다.
