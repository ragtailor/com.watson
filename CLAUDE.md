# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## 저장소 구조

이 저장소는 세 개의 git 서브모듈로 이루어진 모노레포다.

```
com.ragwatson/
├── backend/    # FastAPI Python 백엔드 (서브모듈: com.ragwatson.api)
├── frontend/   # Next.js TypeScript 프론트엔드 (서브모듈: com.ragwatson.www)
└── docs/       # Obsidian 문서 (서브모듈: com.ragwatson.docs)
```

서브모듈을 처음 받을 때: `git submodule update --init --recursive`

---

## Backend

### 실행

```bash
cd backend
# PYTHONPATH에 backend/와 backend/apps/ 모두 포함 필요
$env:PYTHONPATH = ".;apps"   # PowerShell
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

### 환경 변수

`backend/.env`를 `.env.example` 기반으로 생성한다.

```
DATABASE_URL=postgresql+psycopg://user:password@localhost:5432/dbname
GEMINI_API_KEY=...
GEMINI_MODEL=gemini-2.5-flash   # 선택
```

### 의존성

```bash
cd backend
pip install -r requirements.txt
```

### DB 마이그레이션

테이블은 앱 시작 시 `create_all_tables()`로 자동 생성된다. Alembic 마이그레이션이 필요할 때:

```bash
cd backend
alembic revision --autogenerate -m "description"
alembic upgrade head
```

### 아키텍처

**헥사고날(Ports & Adapters)** 아키텍처를 사용한다. 각 앱(`apps/<name>/`)은 다음 레이어로 구성된다.

```
apps/<앱명>/
├── domain/             # 엔티티·값 객체 (순수 비즈니스 로직)
├── app/
│   ├── ports/input/    # 유스케이스 인터페이스 (입력 포트)
│   ├── ports/output/   # 레포지터리 인터페이스 (출력 포트)
│   └── use_cases/      # 유스케이스 구현체
└── adapter/
    ├── inbound/api/    # FastAPI 라우터 및 Pydantic 스키마
    └── outbound/       # DB 구현체 (ORM 모델, pg 레포지터리)
```

**Python import 경로**: `backend/apps/`가 PYTHONPATH에 있으므로 `from titanic.xxx import ...` 형태로 임포트한다 (`from apps.titanic.xxx`가 아님).

### 앱 목록 및 역할

| 앱 | 역할 |
|----|------|
| `titanic` | 타이타닉 승객 CSV 업로드·조회 (ML 교육용 데이터셋) |
| `friday13th` | 사용자·관리자 관리 |
| `matrix` | Gemini API 키·외부 클라이언트 싱글톤 관리 (`Keymaker`) |
| `inception` | DB 헬스체크 |
| `imitation_game` | 문서 읽기·분석 |
| `social_network` | 소셜 기능 (스켈레톤) |

### 네이밍 컨벤션

파일명·클래스명·라우터 prefix에 영화 캐릭터 이름을 bounded context 식별자로 사용한다. 예: `james_router`, `rose_router`, `walter_repository`, `jason_command_router`. 새 컴포넌트를 추가할 때도 해당 앱의 기존 캐릭터 체계를 따른다.

---

## Frontend

### 실행

```bash
cd frontend
pnpm dev        # 개발 서버 (localhost:3000)
pnpm build      # 프로덕션 빌드
pnpm lint       # ESLint
```

패키지 매니저는 **pnpm**을 사용한다.

### 환경 변수

`frontend/.env.local`:

```
GEMINI_API_KEY=...
```

### 기술 스택

- **Next.js 16** (App Router) + **React 19** + **TypeScript 5.7**
- **Tailwind CSS v4** (`@tailwindcss/postcss`)
- **shadcn/ui** — `components/ui/`에 Radix UI 기반 컴포넌트. 새 UI 컴포넌트는 `pnpm dlx shadcn@latest add <component>`로 추가한다.
- **Gemini AI** — `app/api/gemini/chat/route.ts`가 Google REST API 직접 호출 (SDK 미사용)

### 페이지 구조

| 경로 | 역할 |
|------|------|
| `/` | 홈 (Hero, Portfolio, Education, Contact) |
| `/lesson` | 교육 레슨 목록 |
| `/lesson/titanic` | 타이타닉 ML 수업 개요 |
| `/lesson/titanic/data-collection` | CSV 업로드·데이터 수집 실습 |
| `/titanic` | 타이타닉 데이터 뷰어 |
| `/login` | 로그인 |
| `/notice` | 공지사항 |

### 컴포넌트 컨벤션

- 페이지별 컴포넌트는 `components/<페이지명>/` 하위에 둔다
- `components/ui/`는 shadcn/ui 자동 생성 파일이므로 직접 수정하지 않는다
- `next.config.mjs`의 `typescript.ignoreBuildErrors: true`는 의도된 설정이다 (빌드 시 타입 에러 무시)

---

# LLM 코딩 행동 지침

일반적인 LLM 코딩 실수를 줄이기 위한 행동 지침이다. [Andrej Karpathy의 관찰](https://x.com/karpathy/status/2015883857489522876)을 바탕으로 정리되었다. 프로젝트별 지침이 있으면 본 문서와 병합하여 사용한다.

**트레이드오프:** 속도보다 신중함에 우선한다. 사소한 작업은 상황에 맞게 판단한다.

---

## 1. Think Before Coding (구현 전 사고)

**가정하지 않는다. 혼란을 숨기지 않는다. 트레이드오프를 드러낸다.**

구현에 들어가기 전에 다음을 지킨다.

- 가정은 명시한다. 불확실하면 질문한다.
- 해석이 여러 가지면 조용히 하나를 고르지 말고, 가능한 해석을 모두 제시한다.
- 더 단순한 방법이 있으면 말한다. 타당하면 사용자 요청에 반대·수정 의견을 낸다.
- 불명확하면 멈춘다. 무엇이 혼란스러운지 이름 붙이고 질문한다.

---

## 2. Simplicity First (단순성 우선)

**문제를 푸는 데 필요한 최소한의 코드만 쓴다. 추측성 내용은 넣지 않는다.**

- 요청받지 않은 기능은 넣지 않는다.
- 일회성 코드를 위한 추상화는 만들지 않는다.
- 요청받지 않은 “유연함”이나 “설정 가능성”은 넣지 않는다.
- 현실적으로 일어날 수 없는 시나리오를 위한 예외 처리는 하지 않는다.
- 200줄로 쓸 수 있는 것을 50줄로 줄일 수 있으면 다시 쓴다.

스스로에게 묻는다: “시니어 엔지니어가 이건 과하게 복잡하다고 할까?” 그렇다면 단순화한다.

---

## 3. Surgical Changes (정밀한 수정)

**꼭 필요한 곳만 손대고, 본인이 만든 잔여만 정리한다.**

기존 코드를 고칠 때:

- 인접한 코드·주석·포맷을 “개선”하지 않는다.
- 망가지지 않은 부분은 리팩터링하지 않는다.
- 본인 스타일과 달라도 기존 스타일을 맞춘다.
- 작업과 무관한 데드 코드를 발견하면 언급만 하고, 임의로 삭제하지 않는다.

본인 변경으로 쓰이지 않게 된 것이 있으면:

- 본인 변경 때문에 불필요해진 import·변수·함수는 제거한다.
- 원래부터 있던 데드 코드는 요청이 없으면 제거하지 않는다.

**검증:** 바뀐 모든 줄이 사용자 요청과 직접적으로 연결되어야 한다.

---

## 4. Goal-Driven Execution (목표 중심 실행)

**성공 기준을 정의한다. 검증될 때까지 반복한다.**

작업을 검증 가능한 목표로 바꾼다.

- “유효성 검사 추가” → “잘못된 입력에 대한 테스트를 쓰고, 통과시킨다”
- “버그 수정” → “재현 테스트를 쓰고, 통과시킨다”
- “X 리팩터링” → “전후로 테스트가 통과함을 확인한다”

다단계 작업이면 짧은 계획을 쓴다.

```text
1. [단계] → 검증: [확인 사항]
2. [단계] → 검증: [확인 사항]
3. [단계] → 검증: [확인 사항]
```

성공 기준이 분명해야 같은 기준으로 반복할 수 있다. “작동만 하게”처럼 약한 기준은 계속 되묻게 만든다.
