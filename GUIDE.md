# SEF-2026 플러그인 상세 가이드

> 작성일: 2026-03-27 | 버전: v1.0.0

---

## 1. 개요

SEF-2026 플러그인은 GX 사업본부 공공/민간 SI 프로젝트 개발팀을 위한 Claude Code 자동화 플러그인입니다. 자연어 한 마디로 PRD 작성부터 설계, 구현, 코드 리뷰, PR 생성까지 에이전트 팀이 전 과정을 자동으로 처리합니다.

### 핵심 구성

| 구성 요소 | 수량 | 설명 |
|-----------|------|------|
| 스킬 | 12개 | 사용자가 `/명령어`로 호출하는 진입점 |
| 에이전트 | 11개 | 파이프라인 내부에서 자동 투입되는 전문가 |
| Reference 문서 | 46개 | `§번호`로 참조되는 코딩 표준 문서 |

### 지원 기술 스택

| 항목 | 공공 | 민간 |
|------|------|------|
| 프레임워크 | Spring Boot 2.7 + eGovFrame 4.1 | Spring Boot 3.x |
| ORM | MyBatis | JPA/Hibernate |
| Java | 8 | 17+ |
| HTTP 메서드 | GET/POST only | GET/POST/PUT/DELETE |
| 캐싱 | 없음 | Redis @Cacheable |
| 배포 | WAR | Docker/K8s |

---

## 2. 퀵스타트 (3분 가이드)

### Step 1. 설치

```bash
claude plugin add sqi-energy/sqisoft-sef-2026-plugin
```

### Step 2. 초기 설정

```
/sef-setup
```

setup은 다음 7단계를 순차적으로 실행합니다.

1. VCS 감지 (git/svn)
2. Sector 감지 (public/private) + conventions 자동 설정
3. 필수 도구 확인 (gh CLI, JDK)
4. 인증 (GitHub device flow 또는 SVN 자격증명)
5. 빌드/테스트 명령 감지
6. `context/` 디렉토리 안내
7. Google Chat 알림 연동 (선택)

### Step 3. 첫 번째 개발 명령

```
/sef-dev 공지사항 관리 기능 개발해줘
```

이 한 문장으로 PRD 작성 → 기술 설계 → 코드 구현 → 리뷰 → 인수검증 → 커밋/PR까지 자동으로 진행됩니다.

---

## 3. 아키텍처

### 3.1 전체 구조

```
사용자
  └─ /스킬 명령어 (12개)
       └─ 에이전트 팀 (11개)
            └─ Reference 문서 (46개, §번호 참조)
```

- **스킬**: 사용자가 `/명령어`로 호출하는 진입점
- **에이전트**: `/sef-dev` 내부에서 Phase별로 자동 투입되는 전문가
- **Reference**: `§번호`로 참조되는 코딩 표준 문서 (sector별 분리)

### 3.2 sef-dev 6단계 파이프라인

```
Phase 0: setup        — 작업환경 준비 (브랜치 생성, sector 감지)
Phase 1: requirements — PRD 작성 (product-owner, Q&A 루프)
Phase 2: design       — 기술 설계 (architect + design-critic)
Phase 3: implement    — 코드 구현 (coder + qa-manager 자기점검)
Phase 4: review       — 코드 리뷰 (qa-manager + security-auditor 병렬)
Phase 5: complete     — 인수검증 + 커밋/PR (product-owner)
```

#### 3가지 실행 모드

| 모드 | 실행 Phase | 트리거 자연어 |
|------|-----------|--------------|
| NORMAL | 전체 6단계 | (기본값) |
| HOTFIX | setup → requirements(경량) → implement → complete | "긴급", "핫픽스", "빨리 고쳐" |
| IMPLEMENT | setup → implement → complete | "구현만", "설계 없이 바로" |

#### 파이프라인 상태 관리 (.dev/ 디렉토리)

`/sef-dev` 실행 시 프로젝트 루트에 `.dev/{브랜치슬러그}/` 디렉토리가 생성됩니다. 이 디렉토리에는 파이프라인의 중간 산출물이 저장됩니다.

```
.dev/{branch-slug}/
├── state.md       — 현재 Phase, 모드, 시작 시각 등 상태 정보
├── prd.md         — Phase 1(requirements)에서 생성된 PRD 문서
├── design.md      — Phase 2(design)에서 생성된 설계서
└── (기타 Phase별 산출물)
```

`.dev/` 디렉토리는 **파이프라인 상태 관리 전용**이며, `.gitignore`에 포함되어 **git으로 추적하지 않습니다.** 커밋이나 PR에 포함되지 않으므로 안심하고 사용할 수 있습니다. 파이프라인 완료 후에도 로컬에 남아 있어 `/sef-dev 이어서 해줘`로 재개할 때 활용됩니다.

> **주의**: `.dev/` 하위 파일을 직접 편집하거나 git add 하지 마세요. 파이프라인이 상태를 직접 관리합니다.

---

### 3.3 Reference 시스템 — 동작 원리 상세

이 섹션은 SEF-2026 플러그인의 핵심 동작 원리를 설명합니다. Reference 시스템을 이해하면 에이전트가 왜 그 코드를 그 방식으로 작성하는지, 그리고 어떻게 46개의 표준 문서가 실제 코드 한 줄 한 줄에 반영되는지 알 수 있습니다.

#### Reference란?

Reference는 프로젝트에서 따라야 할 코딩 표준, 패턴, 설정 가이드를 문서화한 것입니다. 총 46개 파일이 `reference/` 디렉토리에 있으며, sector(공공/민간)에 따라 다른 문서가 적용됩니다.

```
reference/
├── MANIFEST-public.md      ← 공공 sector 파일 목록 + CHECKLIST
├── MANIFEST-private.md     ← 민간 sector 파일 목록 + CHECKLIST
├── shared/ (22개)          ← 공통 문서 (프론트엔드, 데이터, 품질 등)
│   ├── frontend/ (10개)
│   ├── data/ (2개)
│   ├── config/ (2개)
│   ├── quality/ (3개)
│   ├── tools/ (1개)
│   └── templates/ (4개)
├── public/ (17개)          ← 공공 전용 (백엔드, 배포)
│   ├── backend/ (14개)
│   └── deployment/ (3개)
├── private/ (3개)          ← 민간 전용 (백엔드, 배포)
│   ├── backend/ (2개)
│   └── deployment/ (1개)
└── workflows/ (2개)        ← 워크플로우 가이드
```

---

#### 왜 3계층인가? — 토큰 효율 87% 절약

모든 reference 문서를 한 번에 로드하면 수만 줄에 달합니다. LLM의 컨텍스트 윈도우를 낭비하지 않기 위해, **필요한 것만 필요한 시점에** 단계적으로 로드합니다.

```
┌──────────────────────────────────────────────────────────┐
│  1계층: MANIFEST (1회 로드, 가벼움)                        │
│  ┌──────────────────────────────────────────────────┐    │
│  │  각 파일의 이름, 경로, 적용 여부 (✓ / △ / -)     │    │
│  │  → architect가 Phase 2(design)에서 단 1회 Read   │    │
│  └──────────────────────────────────────────────────┘    │
│                         ↓                                 │
│  2계층: CHECKLIST (파일별, 선택적 로드)                    │
│  ┌──────────────────────────────────────────────────┐    │
│  │  각 파일 안의 핵심 체크 항목 (5~15줄)              │    │
│  │  → architect가 ✓/△ 파일의 CHECKLIST를 Read       │    │
│  │  → 설계서에 "§P-B07.6 준수" 같은 §번호 삽입       │    │
│  └──────────────────────────────────────────────────┘    │
│                         ↓                                 │
│  3계층: 상세 문서 (항목별, 최소 로드)                      │
│  ┌──────────────────────────────────────────────────┐    │
│  │  코드 예시, 설정값, 상세 설명 (수십~수백 줄)        │    │
│  │  → coder가 설계서의 §번호를 보고 해당 섹션만        │    │
│  │    선택적으로 Read                                 │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

각 계층의 역할을 정리하면 다음과 같습니다.

| 계층 | 문서 | 크기 | 로드 시점 | 로드 주체 |
|------|------|------|----------|----------|
| 1계층 | MANIFEST-{sector}.md | 수백 줄 | Phase 0 setup에서 1회 | (자동) |
| 2계층 | 각 reference 파일 내 CHECKLIST 섹션 | 5~15줄 | Phase 2 design에서 선택적 | architect |
| 3계층 | 각 reference 파일의 §번호 섹션 | 수십~수백 줄 | Phase 3 implement에서 선택적 | coder |

---

#### MANIFEST — 1계층의 핵심

MANIFEST는 46개 reference 파일 전체의 색인입니다. architect가 설계를 시작하기 전에 딱 한 번 읽는 문서로, 어떤 파일이 이번 작업에 적용되는지 한눈에 파악할 수 있도록 구성되어 있습니다.

MANIFEST의 구조 예시:

```markdown
## Backend

| 파일 | §번호 접두사 | 적용 여부 | 설명 |
|------|------------|----------|------|
| mybatis-query-patterns.md   | §P-B03 | ✓  | MyBatis XML 쿼리 패턴 |
| security-setup.md           | §P-B07 | ✓  | 보안 설정 (XSS, CSRF, SQL Injection) |
| audit-logging.md            | §P-B08 | △  | 감사 로깅 (@Auditable) — CRUD 시 적용 |
| batch-processing.md         | §P-B12 | -  | 배치 처리 — 배치 작업 시에만 적용 |

## Frontend (Shared)

| 파일 | §번호 접두사 | 적용 여부 | 설명 |
|------|------------|----------|------|
| page-templates.md           | §S-F03 | ✓  | 공통 페이지 템플릿 |
| form-validation.md          | §S-F05 | ✓  | 폼 유효성 검증 패턴 |
```

적용 여부 기호의 의미:

| 기호 | 의미 |
|------|------|
| ✓ | 이번 작업에 필수 적용 |
| △ | 조건부 적용 (조건은 설명 컬럼 참조) |
| - | 이번 작업과 무관, 로드 불필요 |

architect는 MANIFEST를 보고 ✓와 △ 항목만 추려서 2계층(CHECKLIST)을 읽습니다. - 항목은 완전히 무시합니다. 이것이 토큰을 절약하는 첫 번째 관문입니다.

---

#### §번호 체계

모든 reference 항목에는 고유한 §번호가 붙습니다. 이 번호가 설계서와 코드를 연결하는 핵심 식별자입니다.

```
§ P - B 07 . 6
│ │   │  │    └── 항목 번호 (해당 문서 내 6번째 체크 항목)
│ │   │  └─────── 문서 순번 (07번째 문서)
│ │   └────────── 카테고리 (B = Backend)
│ └────────────── Sector (P = Public)
└──────────────── §번호 시작 기호
```

**Sector 접두사**

| 접두사 | 의미 | 예시 |
|--------|------|------|
| P | 공공 (Public) | §P-B07.6 |
| R | 민간 (pRivate) | §R-B01.3 |
| S | 공통 (Shared) | §S-F02.1 |

**카테고리**

| 기호 | 카테고리 | 파일 위치 |
|------|----------|----------|
| B | Backend | `reference/{sector}/backend/` |
| F | Frontend | `reference/shared/frontend/` |
| D | Deployment | `reference/{sector}/deployment/` |
| Q | Quality | `reference/shared/quality/` |
| T | Tools | `reference/shared/tools/` |
| C | Config | `reference/shared/config/` |

---

#### §번호 → 파일 경로 변환 규칙

coder가 §번호를 보고 실제 파일을 찾을 때 적용하는 규칙입니다.

| §번호 접두사 | 파일 경로 |
|-------------|----------|
| §P-B | `reference/public/backend/` |
| §R-B | `reference/private/backend/` |
| §S-F | `reference/shared/frontend/` 또는 `reference/shared/templates/` |
| §S-D | `reference/shared/data/` |
| §S-Q | `reference/shared/quality/` |
| §S-T | `reference/shared/tools/` |
| §S-C | `reference/shared/config/` |
| §P-D | `reference/public/deployment/` |
| §R-D | `reference/private/deployment/` |

순번(07)은 해당 디렉토리 내 파일 목록에서 07번째 파일을 가리킵니다. MANIFEST에 파일명과 §번호 접두사가 함께 나와 있으므로 coder는 MANIFEST를 다시 읽지 않아도 됩니다 — 설계서에 적힌 §번호만 보면 파일 경로를 바로 도출할 수 있습니다.

---

#### Reference 참조 흐름 — 에이전트별 역할 분담

각 에이전트가 Reference를 어떻게 사용하는지 단계별로 설명합니다.

**전체 흐름 다이어그램**

```
┌─────────────────────────────────────────────────────────────┐
│  Phase 0: setup                                             │
│                                                             │
│  config.json에서 sector 읽기 ("public" / "private")         │
│  → MANIFEST-{sector}.md를 1회 Read → 메모리에 보관           │
│  → 브랜치 생성, .dev/{branch-slug}/ 디렉토리 초기화           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Phase 1: requirements — product-owner                      │
│                                                             │
│  사용자 요청을 Q&A 루프로 구체화하여 PRD 작성                  │
│  → .dev/{branch-slug}/prd.md 저장                           │
│  (Reference 미사용 — 비즈니스 요건 정의 단계)                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Phase 2: design — architect (+ design-critic)              │
│                                                             │
│  [1] MANIFEST에서 이번 작업과 관련된 파일 선별                 │
│      ✓ = 반드시 읽음  △ = 조건 확인 후 읽음  - = 무시          │
│                                                             │
│  [2] 선별된 파일의 CHECKLIST 섹션만 Read (2계층)              │
│      예) mybatis-query-patterns.md의 CHECKLIST 5줄 읽기      │
│                                                             │
│  [3] 설계서(design.md)에 §번호를 명시하며 작성                 │
│      예) "MyBatis XML 매퍼는 §P-B03.2 동적 WHERE 패턴을        │
│           따른다. 페이지네이션은 §P-B03.5 오프셋 기반           │
│           패턴을 적용한다."                                    │
│                                                             │
│  ※ architect만 MANIFEST를 읽습니다.                          │
│    다른 에이전트는 MANIFEST를 직접 읽지 않습니다.               │
│                                                             │
│  → .dev/{branch-slug}/design.md 저장                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Phase 3: implement — coder (+ qa-manager 자기점검)          │
│                                                             │
│  [coder]                                                    │
│  [1] design.md를 읽고 §번호 목록 추출                         │
│      예) [§P-B03.2, §P-B03.5, §P-B08.3, §S-F03.1]          │
│                                                             │
│  [2] §번호 → 파일 경로 변환 규칙 적용                         │
│      §P-B03 → reference/public/backend/ 의 03번 파일         │
│      §S-F03 → reference/shared/templates/ 의 03번 파일       │
│                                                             │
│  [3] 해당 파일의 §번호 섹션만 선택적으로 Read (3계층)           │
│      전체 문서가 아닌 .2 / .5 항목 등 필요한 부분만 읽음        │
│                                                             │
│  [4] 읽은 패턴대로 코드 구현                                   │
│                                                             │
│  ※ coder는 MANIFEST를 직접 읽지 않습니다.                     │
│    설계서에 적힌 §번호만 따릅니다.                             │
│    이렇게 하면 architect의 설계 의도가 코드에 그대로 반영됩니다.│
│                                                             │
│  [qa-manager — 자기점검]                                     │
│  - design.md의 §번호 + antipatterns.md CHECKLIST를 읽음      │
│  - coder가 작성한 코드와 §번호 기준을 대조                      │
│  - CERTAIN(확실한 문제)만 지적 — 모호하면 지적하지 않음         │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Phase 4: review — qa-manager + security-auditor (병렬)     │
│                                                             │
│  [qa-manager]                                               │
│  - 설계서 §번호와 구현 코드를 대조하여 스펙 충족 여부 검증       │
│  - CERTAIN(확실한 문제) / QUESTION(확인 필요) 으로 분류        │
│                                                             │
│  [security-auditor]                                         │
│  - 보안 관련 §항목 + antipatterns.md §보안 섹션 대조           │
│  - RISK / GAP / POLICY / ASSUMPTION 4가지로 분류             │
│  - OWASP Top 10 기준으로 검증                                 │
│                                                             │
│  두 에이전트가 병렬로 동작하여 리뷰 시간을 단축합니다.           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  Phase 5: complete — product-owner                          │
│                                                             │
│  PRD 기준으로 인수검증 → 통과 시 커밋/PR 생성                  │
└─────────────────────────────────────────────────────────────┘
```

**핵심 원칙: "architect만 MANIFEST를 읽고, coder는 §번호만 따른다"**

이 분리가 중요한 이유:

1. architect가 전체 reference를 파악하고, 이번 작업에 필요한 항목만 선별합니다.
2. coder는 선별된 §번호만 참조하므로 불필요한 문서를 읽지 않습니다.
3. qa-manager와 security-auditor는 §번호를 검증 기준으로 삼아 일관된 리뷰를 수행합니다.
4. 결과적으로 토큰을 87% 절약하면서도 코딩 표준이 빠짐없이 적용됩니다.

---

#### 구체적 예시: 공공 프로젝트 공지사항 CRUD 개발 시 Reference 흐름

아래는 `/sef-dev 공지사항 관리 기능 개발해줘`를 실행했을 때 Reference 시스템이 실제로 동작하는 과정입니다.

```
1. Phase 0 (setup)
   sector = "public" 감지
   → MANIFEST-public.md 로드 (1회)


2. Phase 2 (design) — architect
   MANIFEST에서 관련 파일 선별:

   ✓ mybatis-query-patterns.md   (§P-B03)  — MyBatis 쿼리 패턴
   ✓ security-setup.md           (§P-B07)  — 보안 설정
   ✓ page-templates.md           (§S-F03)  — 페이지 템플릿
   △ audit-logging.md            (§P-B08)  — CRUD이므로 조건 충족 → 적용
   - batch-processing.md         (§P-B12)  — 배치 없음 → 무시

   각 파일의 CHECKLIST를 Read (2계층):
   → mybatis-query-patterns.md CHECKLIST 7줄
   → security-setup.md CHECKLIST 9줄
   → page-templates.md CHECKLIST 5줄
   → audit-logging.md CHECKLIST 6줄

   설계서(design.md)에 §번호를 명시하여 작성:
   "Controller는 §P-B03.1 GET/POST only 패턴을 따른다"
   "Mapper XML은 §P-B03.2 동적 WHERE 패턴을 사용한다"
   "목록 조회는 §P-B03.5 오프셋 기반 페이지네이션을 적용한다"
   "등록/수정/삭제 메서드에는 §P-B08.3 @Auditable을 적용한다"
   "목록 페이지는 §S-F03.1 DataTable 템플릿을 사용한다"
   "등록/수정 폼은 §S-F03.2 FormPanel 템플릿을 사용한다"


3. Phase 3 (implement) — coder
   design.md에서 §번호 추출:
   → [§P-B03.1, §P-B03.2, §P-B03.5, §P-B08.3, §S-F03.1, §S-F03.2]

   §번호별로 해당 파일의 해당 섹션만 Read (3계층):
   → reference/public/backend/mybatis-query-patterns.md
     § P-B03.1 섹션 (GET/POST only Controller 패턴 코드 예시)
     § P-B03.2 섹션 (동적 WHERE XML 예시)
     § P-B03.5 섹션 (오프셋 페이지네이션 XML + Java 예시)
   → reference/public/backend/audit-logging.md
     § P-B08.3 섹션 (@Auditable 애노테이션 적용 예시)
   → reference/shared/templates/page-templates.md
     § S-F03.1 섹션 (DataTable 템플릿 HTML/JS)
     § S-F03.2 섹션 (FormPanel 템플릿 HTML/JS)

   읽은 패턴대로 코드 구현


4. Phase 4 (review) — qa-manager + security-auditor
   qa-manager:
   → §P-B03.1 기준: Controller에 PUT/DELETE 없는지 확인
   → §P-B08.3 기준: create/update/delete에 @Auditable 있는지 확인
   → 위반 시 CERTAIN으로 분류하여 수정 요청

   security-auditor:
   → §P-B07 기준: SQL Injection 방지 패턴 (PreparedStatement, #{} 사용) 확인
   → OWASP Top 10 A03 (Injection) 기준으로 검증
   → 위반 시 RISK로 분류
```

이 흐름을 통해 개발자가 직접 코딩 표준을 찾아보지 않아도, 에이전트가 46개 문서 중 필요한 섹션만 정확히 참조하여 표준에 맞는 코드를 생성합니다.

---

### 3.4 Sector 분기 상세

| 항목 | 공공 (public) | 민간 (private) |
|------|--------------|----------------|
| 프레임워크 | Spring Boot 2.7 + eGovFrame 4.1 | Spring Boot 3.x |
| ORM | MyBatis | JPA/Hibernate |
| Java | 8 | 17+ |
| HTTP 메서드 | GET/POST only | GET/POST/PUT/DELETE |
| API 접두사 | `/adm/v1/`, `/api/v1/` | `/api/` |
| 서비스 구조 | interface + Impl (EgovAbstractServiceImpl) | interface + Impl (권장) |
| 캐싱 | 없음 | Redis @Cacheable |
| 배포 | WAR | Docker/K8s |

---

## 4. 스킬 레퍼런스 (12개)

### 4.1 /sef-dev — 전체 개발 사이클 오케스트레이터

전체 개발 파이프라인을 자동 실행하는 핵심 스킬입니다.

**트리거 자연어**: "개발해줘", "구현해줘", "기능 추가", "만들어줘"

**모드 전환 자연어**

| 자연어 | 동작 |
|--------|------|
| "긴급", "핫픽스", "빨리 고쳐" | HOTFIX 모드 |
| "구현만", "설계 없이 바로" | IMPLEMENT 모드 |
| "설계만", "PRD만" | 해당 Phase만 실행 |
| "어디까지 됐어?", "상태 확인" | 현재 상태 출력 |
| "이어서 해줘", "계속" | 중단된 지점부터 재개 |
| "{branch}에서 작업해줘" | BASE 브랜치 지정 |

**투입 에이전트**: product-owner, architect, design-critic, coder, qa-manager, security-auditor, researcher, hacker, simplifier

**사용 예시**

```
/sef-dev 공지사항 관리 기능 개발해줘
/sef-dev 로그인 버그 긴급 수정해줘
/sef-dev 결제 모듈 구현만 해줘
/sef-dev develop 브랜치에서 작업해줘
```

---

### 4.2 /sef-commit — Git 커밋

변경사항을 분석하여 한국어 커밋 메시지를 자동 생성하고 커밋합니다.

**트리거 자연어**: "커밋", "커밋해줘", "변경사항 저장", "commit"

**동작 흐름**

1. 브랜치명에서 커밋 타입 자동 파싱
2. 빌드 사전 검증 실행
3. 민감 파일 감지 (`.env*`, `*.key`, `*.pem`, `credentials*`, `*secret*`)
4. 한국어 커밋 메시지 자동 생성
5. 커밋 후 context 동기화 제안

**커밋 메시지 형식**

```
{type}: {한국어 요약}

- 변경 내용 bullet 1
- 변경 내용 bullet 2
```

**참고**: SVN 환경에서는 지원하지 않습니다. `svn commit`을 직접 실행하세요.

---

### 4.3 /sef-pull-request — PR 생성

커밋 히스토리를 분석하여 PR 제목과 본문을 자동 생성합니다.

**트리거 자연어**: "PR", "PR 올려", "PR 생성", "풀리퀘"

**동작 흐름**

1. 커밋 히스토리 분석
2. PR 제목/본문 자동 생성
3. `gh pr create` 실행
4. Google Chat 웹훅 알림 전송 (설정된 경우)

**PR 제목 형식**

```
[TYPE] 설명을 ~한다.
```

예시: `[FEATURE] 로그인 기능을 구현한다.`

**PR 본문 구조**: Background → Summary → Changes → Checklist

**참고**: 기존 PR이 있는 경우 업데이트/신규 생성/취소 중 선택합니다.

---

### 4.4 /sef-context — 도메인 컨텍스트 관리

프로젝트의 도메인 지식을 `context/` 디렉토리에 등록하고 관리합니다.

**트리거 자연어**: "도메인 등록", "컨텍스트", "용어 정리", "context"

**5가지 모드 (자동 판단)**

| 모드 | 조건/트리거 | 설명 |
|------|------------|------|
| 스캔 | `context/` 없을 때 | 코드베이스 분석 → 도메인 자동 생성 |
| 신규 | "도메인 등록해줘" | Q&A 기반 새 도메인 생성 (5개 검증 질문) |
| 문서 기반 | "파일 기반으로 만들어줘" | 파일에서 도메인 추출하여 생성/갱신 |
| 갱신 | "도메인 수정해줘" | 기존 도메인 내용 수정 |
| 동기화 | "동기화해줘" | git 히스토리 분석 → `status.md` 갱신 |

**사용 예시**

```
/sef-context 결제 도메인 등록해줘
/sef-context docs/req.md 기반으로 결제 컨텍스트 만들어줘
/sef-context 결제 도메인 동기화해줘
```

---

### 4.5 /sef-lens — 비즈니스 정책 분석 (읽기 전용)

코드에서 비즈니스 정책을 추출하여 비개발자도 이해할 수 있는 보고서로 제공합니다.

**트리거 자연어**: "정책 확인", "영향도", "이거 바꾸면 어디 영향", "정리해줘"

**동작 흐름**: Prepare → Explore → Report → (선택) Impact → Impact-Report

**사용 예시**

```
/sef-lens 할인 정책 정리해줘
/sef-lens 할인 한도를 500원으로 바꾸면 어떻게 돼?
/sef-lens 결제 플로우 상세하게 분석해줘
```

**참고**: 코드를 수정하지 않는 읽기 전용 스킬입니다.

---

### 4.6 /sef-research — 웹 리서치

웹 검색과 문서 분석을 통해 출처가 명확한 기술 리포트를 제공합니다.

**트리거 자연어**: "조사해줘", "찾아봐", "리서치", "비교해줘"

**결과물 형태**

| 형태 | 설명 |
|------|------|
| 종합 리포트 | 여러 출처를 종합한 분석 문서 |
| 비교표 | 기술/라이브러리 비교 |
| 핵심 요약 | 빠른 결론 중심 요약 |

**조사 깊이**

| 깊이 | 키워드 수 | 조회 URL |
|------|----------|---------|
| 꼼꼼하게 | 5개 | 10개 |
| 빠르게 | 3개 | 5개 |

결과는 `.research/` 디렉토리에 저장됩니다.

---

### 4.7 /sef-verify — 패턴 검증 + 문서 동기화

변경된 코드가 SEF-2026 표준을 준수하는지 검증하고, Reference 문서와 코드의 일치 여부를 확인합니다.

**트리거 자연어**: "검증해줘", "패턴 확인", "문서 동기화", "레퍼런스 맞아?"

**2가지 모드**

| 모드 | 트리거 | 설명 |
|------|--------|------|
| 코드 검증 (기본) | "검증해줘" | 변경 코드가 CHECKLIST 준수 여부 확인 → CRITICAL/WARNING/INFO 분류 → 자동 교정 선택 |
| 동기화 | "문서 동기화해줘" | reference 문서의 코드 예시와 실제 소스 일치 여부 확인 → 불일치 자동 수정 |

**사용 예시**

```
/sef-verify 검증해줘
/sef-verify 문서 동기화해줘
```

---

### 4.8 /sef-humanizer — AI 글쓰기 교정

AI가 작성한 문서에서 AI 특유의 표현 패턴을 감지하고 자연스러운 사람 글로 교정합니다.

**트리거 자연어**: "AI 티 빼줘", "자연스럽게 고쳐줘", "사람이 쓴 것처럼"

**감지 패턴**: 40개 이상 (한국어 19개 + 영어 19개 + 공통 6개)

**심각도 분류**

| 등급 | 설명 |
|------|------|
| P1 | 확실한 AI 흔적 |
| P2 | AI 작성 의심 |
| P3 | 스타일 개선 권장 |

**2가지 모드**

| 모드 | 설명 |
|------|------|
| audit | 패턴 감지 및 보고만 |
| rewrite | 감지 후 자동 수정 |

적용 콘텐츠 유형: 블로그, 기술 문서, 마케팅 문서, 학술 자료, 코드 주석, SNS 게시물

---

### 4.9 /sef-backend-builder — 백엔드 모듈 생성

sector에 맞는 백엔드 레이어 전체를 자동 생성합니다.

**트리거 자연어**: "백엔드 만들어줘", "API 만들어줘", "CRUD 만들어줘", "모듈 생성"

**sector별 생성 파일**

| sector | 생성 파일 |
|--------|----------|
| 공공 | Controller → Service(interface+Impl) → Mapper → MyBatis XML → DTO |
| 민간 | Controller → Service → Repository → Entity → DTO |

**sector별 특징**

- 공공: GET/POST only, `@Mapper`, `EgovAbstractServiceImpl` 상속, `@Auditable` 적용
- 민간: 표준 REST, JPA `@Entity`, Redis `@Cacheable` 지원

---

### 4.10 /sef-frontend-builder — 프론트엔드 컴포넌트 생성

Nuxt 4 기반 프론트엔드 컴포넌트를 자동 생성합니다.

**트리거 자연어**: "화면 만들어줘", "마크업", "폼 구현", "컴포넌트 생성", "UI 짜줘"

**기술 스택**: Nuxt 4 + Vue 3 Composition API + shadcn-vue + Tailwind CSS v4

**2가지 모드**

| 모드 | 설명 |
|------|------|
| Markup | 정적 마크업 + 스타일링 (로직 없음) |
| Full | API 연동 + 상태관리 + 폼 유효성(vee-validate + Zod) |

**지원 컴포넌트 유형**: 폼, 데이터 테이블, 검색 폼

---

### 4.11 /sef-db-schema-query — DB 스키마 조회

연결된 DB의 스키마 구조를 단계적으로 조회합니다.

**트리거 자연어**: "테이블 구조", "DB 스키마", "컬럼 확인", "ERD"

**지원 DB**: PostgreSQL, MySQL, Oracle, SQLite

**조회 흐름**: 스키마 목록 → 테이블 목록 → 테이블 상세 (컬럼, PK, FK, 인덱스)

**참고**: MCP 기반으로 동작합니다. `.env` 파일에 DB 연결 정보를 설정해야 합니다.

---

### 4.12 /sef-setup — 초기 설정

플러그인을 처음 사용하거나 설정을 변경할 때 실행합니다.

**트리거 자연어**: "설정", "셋업", "setup", "초기화"

**7단계 순차 실행**

| 단계 | 내용 |
|------|------|
| 1 | VCS 감지 (git/svn) |
| 2 | Sector 감지 (public/private) + conventions 자동 설정 |
| 3 | 필수 도구 확인 (gh CLI, JDK) |
| 4 | 인증 (GitHub device flow 또는 SVN 자격증명) |
| 5 | 빌드/테스트 명령 감지 |
| 6 | `context/` 디렉토리 안내 |
| 7 | Google Chat 알림 연동 (선택) |

---

## 5. 에이전트 레퍼런스 (11개)

### 5.1 Process 에이전트 (9개) — sef-dev 파이프라인 전용

| 에이전트 | 모델 | 투입 Phase | 역할 |
|---------|------|-----------|------|
| product-owner | sonnet | requirements, complete | PRD 작성 + 인수검증. 비즈니스 언어로 소통. |
| architect | opus | design | MANIFEST → CHECKLIST 기반 §번호 포함 설계서 작성 |
| design-critic | opus | design (중형 이상) | 설계의 암묵적 가정 도전, 과잉설계 식별 |
| coder | opus | implement | 설계서 §번호를 따라 reference 참조하여 구현 |
| qa-manager | sonnet | implement, review | 자기점검(CERTAIN만) + 리뷰(CERTAIN/QUESTION 분류) |
| security-auditor | sonnet | review | RISK/GAP/POLICY/ASSUMPTION 분류, OWASP Top 10 기준 |
| researcher | sonnet | 필요 시 | 근본 원인 분석, 코드베이스 탐색 |
| hacker | sonnet | 정체 시 | 제약 우회 (CONSTRAINT/BYPASS/REFRAME) |
| simplifier | sonnet | 정체 시 | 복잡도 제거 (CUT/MINIMUM/DEFER) |

### 5.2 Planning 에이전트

**screen-spec-writer**: 공공 프로젝트 화면정의서 자동 작성

작성 항목: 화면 ID, 입출력 항목, 버튼 동작 정의, API 연동 명세

### 5.3 Operations 에이전트

**incident-manager**: 장애 대응 전담 에이전트

- 심각도: P1 (서비스 중단) ~ P4 (경미한 이슈)
- 기능: 스택트레이스 분석, 영향 범위 파악, 포스트모템 작성
- 자동 투입 트리거: "에러 나는데", "왜 안 돼", "500 에러", "빌드 실패"

---

## 6. 설정 가이드

### 6.1 config.json

위치: `.claude/config.json`

주요 설정 필드:

| 필드 | 설명 |
|------|------|
| `sector` | `public` 또는 `private` |
| `vcs` | `git` 또는 `svn` |
| `conventions.branchTypes` | 허용 브랜치 타입 목록 |
| `conventions.branchFormat` | 브랜치명 형식 패턴 |
| `conventions.commitFormat` | 커밋 메시지 형식 |
| `conventions.httpMethods` | 허용 HTTP 메서드 |
| `conventions.apiPrefix` | API URL 접두사 |
| `projectTypes` | 프로젝트 유형 목록 |
| `sensitiveFilePatterns` | 민감 파일 감지 패턴 |
| `buildArtifactPatterns` | 빌드 아티팩트 패턴 |
| `contextLimits` | context/ 파일 크기 제한 |
| `notifications` | Google Chat 웹훅 설정 |

### 6.2 MCP 서버

플러그인에 포함된 4개의 MCP 서버:

| 서버 | 용도 |
|------|------|
| context7 | 라이브러리 최신 문서 조회 |
| playwright | 브라우저 자동화 테스트 |
| shadcnVue | shadcn-vue 컴포넌트 문서/예시 |
| sequential-thinking | 복잡한 추론 단계 지원 |

### 6.3 보안 규칙

다음 명령은 pre-tool-guard에 의해 자동으로 차단됩니다.

- `git push --force`, `-f`, `--force-with-lease`
- `git reset --hard`
- SVN 직접 커밋 (SVN 환경에서 `/sef-commit` 사용 시)

**민감 파일 자동 감지 패턴**: `.env*`, `*.key`, `*.pem`, `credentials*`, `*secret*`

**빌드 아티팩트 자동 tracking 해제**: `target/`, `build/`, `dist/`, `node_modules/` 등

---

## 7. 워크플로우 시나리오

### 7.1 신규 기능 개발 (전체 사이클)

```
/sef-setup
  → /sef-context 결제 도메인 등록해줘
  → /sef-dev 결제 기능 개발해줘
      (내부: PRD → 기술 설계 → 구현 → 리뷰 → 인수검증)
  → /sef-commit
  → /sef-pull-request
```

### 7.2 긴급 버그 수정 (핫픽스)

```
/sef-dev 로그인 500 에러 긴급 수정해줘
  (내부: 경량 PRD → 구현 → 인수검증 → 커밋/PR 포함)
```

설계 단계와 코드 리뷰를 생략하고 빠르게 수정합니다.

### 7.3 기존 코드 분석 후 개선

```
/sef-lens 할인 정책 정리해줘
  → /sef-research 최신 할인 처리 패턴 조사해줘
  → /sef-context 할인 도메인 갱신해줘
  → /sef-dev 할인 정책 개선 개발해줘
```

### 7.4 프론트엔드 + 백엔드 독립 생성

```
/sef-db-schema-query 주문 테이블 구조 확인해줘
  → /sef-backend-builder 주문 관리 API 만들어줘
  → /sef-frontend-builder 주문 목록 화면 만들어줘
  → /sef-verify 검증해줘
```

---

## 8. 스킬 체이닝

두 스킬을 연속으로 실행하도록 자연어로 요청할 수 있습니다.

```
커밋하고 PR 올려줘
```

위 요청은 `/sef-commit` 완료 후 자동으로 `/sef-pull-request`를 실행합니다.

**주의**: 첫 번째 스킬이 실패하면 두 번째 스킬은 실행되지 않습니다.

---

## 9. 트러블슈팅

### sector가 설정되지 않았다는 메시지가 나타날 때

`/sef-setup`을 실행하여 sector를 설정하세요.

### gh CLI가 없다는 메시지가 나타날 때

`/sef-setup`에서 자동 설치를 시도하거나, https://cli.github.com 에서 수동으로 설치하세요.

### SVN 환경에서 /sef-commit이 동작하지 않을 때

SVN은 커밋/PR 스킬을 지원하지 않습니다. `svn commit`을 직접 실행하세요.

### Reference 문서와 코드가 맞지 않을 때

```
/sef-verify 문서 동기화해줘
```

### 빌드 실패로 커밋이 되지 않을 때

`/sef-commit`은 커밋 전 빌드 사전 검증을 실행합니다. 빌드 오류를 먼저 수정한 후 다시 시도하세요.

### 파이프라인이 중간에 멈췄을 때

```
/sef-dev 이어서 해줘
```

또는

```
/sef-dev 상태 확인해줘
```

로 현재 진행 상태를 확인하고 재개할 수 있습니다.

### .dev/ 디렉토리가 git에 추적되고 있을 때

`.dev/` 디렉토리는 git 추적 대상이 아닙니다. `.gitignore`에 아래 항목이 있는지 확인하세요.

```
.dev/
```

항목이 없다면 추가하고, 이미 추적 중인 경우 아래 명령으로 추적을 해제하세요.

```bash
git rm -r --cached .dev/
```
