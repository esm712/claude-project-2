# SPEC — 백엔드 개발자 포트폴리오 사이트 기술 명세

> 상태: **초안 v2 (검토 대기)** · 작성일: 2026-10-23
> 요구사항은 [`prd.md`](./prd.md)를 따른다. 기능 ID(F-xx)는 PRD와 동일하다.
> `[가정]` 표시는 확인이 필요한 항목입니다.

## 1. 기술 스택

버전은 2026-10-23 기준 최신 안정(GA) 버전이며, 개발 착수 시 패치 버전만 갱신한다.

### 1.1 백엔드

| 영역 | 선택 | 비고 |
|---|---|---|
| 언어 | Java 25 (LTS) | record, pattern matching, virtual threads 활용 |
| 프레임워크 | Spring Boot 4.1.x (현재 4.1.2) | Spring Framework 7 기반 |
| 웹 | Spring Web MVC + Virtual Threads | `spring.threads.virtual.enabled=true` |
| 영속성 | Spring Data JPA (Hibernate 7) | 조회 성능을 위해 fetch join / `@EntityGraph` 사용 |
| DB | MySQL 8.4 LTS | utf8mb4 |
| 마이그레이션 | Flyway | 스키마·시드 데이터 버전 관리 |
| 보일러플레이트 | Lombok | 엔티티는 `@Getter` + `@NoArgsConstructor(access = PROTECTED)` 중심, `@Data`·`@Setter` 지양 |
| 검증 | Jakarta Bean Validation | 요청 DTO 검증 |
| 보안 | Spring Security + JWT | 관리자 1인 인증 |
| 요청 제한 | Bucket4j | 연락 폼 스팸 방지 |
| 캐시 | Spring Cache + Caffeine | 공개 조회 API |
| API 문서 | springdoc-openapi (Swagger UI) | |
| 운영 | Spring Boot Actuator | `health`, `info`, `metrics` |
| 빌드 | Gradle (Kotlin DSL) | |
| 테스트 | JUnit 5, AssertJ, Mockito, Spring Boot Test, **Testcontainers(MySQL)**, JaCoCo | |

### 1.2 프론트엔드

| 영역 | 선택 | 비고 |
|---|---|---|
| 프레임워크 | Next.js 16.x (현재 16.4.2), App Router | SSG + ISR, On-demand Revalidation |
| 언어 | TypeScript (strict) | |
| 스타일 | Tailwind CSS v4 | |
| 마크다운 렌더링 | `react-markdown` + `remark-gfm` | 프로젝트 본문은 DB에 Markdown으로 저장 |
| API 타입 | `openapi-typescript` | 백엔드 OpenAPI 스펙에서 TS 타입 자동 생성 |
| 테스트 | Vitest + React Testing Library, Playwright, `@axe-core/playwright` | |
| 품질 | ESLint, Prettier, Lighthouse CI | |
| 패키지 매니저 | pnpm | |

## 2. 시스템 아키텍처

```
                 ┌──────────────────────────┐
  Browser ──────▶│ Next.js (Vercel)         │  https://<domain>
                 │  - SSG/ISR 페이지          │
                 │  - /api/revalidate (비공개) │◀──┐ 콘텐츠 변경 시 재검증 요청
                 └────────────┬─────────────┘   │  (공유 시크릿)
                              │ 빌드·재검증 시 조회     │
                              ▼                  │
  Browser ─(연락 폼 POST)─▶┌──────────────────────┴───┐
  Admin  ─(JWT)─────────▶│ Spring Boot API (Docker)  │  https://api.<domain>
                          │  Nginx(TLS) → App         │
                          └────────────┬─────────────┘
                                       ▼
                          ┌──────────────────────────┐
                          │ MySQL 8.4                │
                          └──────────────────────────┘
```

- 공개 페이지는 Next.js가 **빌드·재검증 시점에만** API를 호출하므로, 방문자 트래픽이 API·DB에 직접 가지 않는다.
- 브라우저가 API를 직접 호출하는 것은 연락 폼 제출(F-11)뿐이다.
- 관리자가 콘텐츠를 수정하면 API가 트랜잭션 커밋 후(`@TransactionalEventListener(AFTER_COMMIT)`) Next.js `/api/revalidate`에 해당 경로(태그) 재검증을 요청한다 (F-13).

## 3. 저장소 구조 (모노레포)

```
.
├── backend/
│   ├── build.gradle.kts
│   ├── Dockerfile
│   └── src/
│       ├── main/java/<base-package>/
│       │   ├── PortfolioApplication.java
│       │   ├── global/              # config, security, exception, cache, revalidation
│       │   └── domain/
│       │       ├── profile/         # controller / service / repository / entity / dto
│       │       ├── experience/
│       │       ├── project/
│       │       ├── skill/
│       │       ├── contact/
│       │       └── auth/
│       ├── main/resources/
│       │   ├── application.yml
│       │   └── db/migration/        # V1__init.sql, V2__seed.sql ...
│       └── test/java/...
├── frontend/
│   ├── src/
│   │   ├── app/                     # layout, page, projects/[slug], about-this-site, api/revalidate
│   │   ├── components/              # sections/, ui/, layout/
│   │   ├── lib/                     # api client, utils
│   │   └── types/api.d.ts           # openapi-typescript 생성물
│   ├── public/resume.pdf
│   └── tests/{unit,e2e}/
├── docker-compose.yml               # 로컬: mysql + backend
├── .github/workflows/
│   ├── backend-ci.yml
│   ├── frontend-ci.yml
│   └── backend-deploy.yml
├── prd.md
└── spec.md
```

- 패키지 구조: 도메인형(`domain/<기능>/...`) — 기능 단위 응집도를 보여 주기 위함
- base package: `dev.<name>.portfolio` `[가정: 도메인 확정 후 결정]`

## 4. 데이터 모델 (ERD)

```
profile (1행)
  id PK, name, role, headline, about(TEXT, Markdown), email,
  github_url, linkedin_url, blog_url, resume_url, updated_at

keyword                       ← Hero 핵심 키워드 3개
  id PK, profile_id FK, name, sort_order

experience
  id PK, company, role, start_date(DATE), end_date(DATE NULL=재직 중),
  summary, sort_order, created_at, updated_at
experience_achievement
  id PK, experience_id FK, content, sort_order

project
  id PK, slug UNIQUE, title, summary, period_start, period_end, role,
  thumbnail_url, content(MEDIUMTEXT, Markdown), github_url, demo_url,
  featured, sort_order, created_at, updated_at
project_metric
  id PK, project_id FK, label, before_value NULL, after_value, sort_order

skill_category
  id PK, name, sort_order               ← Language / Backend / Database / Infra·DevOps / Frontend / Collaboration
skill
  id PK, category_id FK, name UNIQUE, level TINYINT(1~5), description, sort_order

project_skill                            ← 프로젝트 기술 태그 (필터 기준)
  project_id FK, skill_id FK, PK(project_id, skill_id)

contact_message
  id PK, name, email, message(TEXT), ip_hash, user_agent,
  status(NEW/READ/ARCHIVED), created_at

admin_user
  id PK, username UNIQUE, password_hash(BCrypt), created_at
```

- 프로젝트 기술 태그는 `skill`과 연결해 "스킬 → 사용한 프로젝트" 역방향 조회도 가능하게 한다.
- `ip_hash`: 원본 IP 대신 SHA-256(IP + salt) 저장 (개인정보 최소화)
- 엔티티 연관관계는 기본 `LAZY`, 목록 조회는 fetch join / `@EntityGraph` + `default_batch_fetch_size`로 N+1 방지
- 관리자 계정은 최초 기동 시 환경 변수(`ADMIN_USERNAME`, `ADMIN_PASSWORD`)로 생성

## 5. API 명세

### 5.1 공통

- Base URL: `https://api.<domain>/api/v1`
- 응답: JSON, 날짜 `YYYY-MM` 또는 ISO-8601
- 에러 응답: RFC 9457 Problem Details (`ProblemDetail`)

```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "입력값이 올바르지 않습니다.",
  "instance": "/api/v1/contacts",
  "errors": [{ "field": "email", "message": "이메일 형식이 아닙니다." }]
}
```

### 5.2 공개 API (인증 불필요, 캐시 적용)

| Method | Path | 설명 | 기능 |
|---|---|---|---|
| GET | `/profile` | 프로필 + 키워드 | F-01, F-02, F-07 |
| GET | `/experiences` | 경력 목록 (최신순, 성과 포함) | F-03 |
| GET | `/projects?tag={skillName}` | 프로젝트 목록 (sort_order 순, 태그 필터 다중 = OR) | F-04 |
| GET | `/projects/{slug}` | 프로젝트 상세 (본문·지표·태그) | F-05 |
| GET | `/skills` | 카테고리별 스킬 (level 포함) | F-06 |
| POST | `/contacts` | 연락 메시지 접수 → `201 Created` | F-11 |

`POST /contacts` 규칙
- 검증: `name` 1~50자, `email` 형식, `message` 10~2000자, 허니팟 필드(`website`)가 비어 있어야 함
- 요청 제한: 동일 IP 기준 **10분당 3회**, 초과 시 `429 Too Many Requests` + `Retry-After`
- CORS: 프론트 도메인만 허용

### 5.3 관리자 API (`/admin/**`, JWT 필요)

| Method | Path | 설명 |
|---|---|---|
| POST | `/auth/login` | 아이디/비밀번호 → Access Token(30분) 발급, 로그인 실패 5회 시 15분 잠금 |
| PUT | `/admin/profile` | 프로필 수정 |
| POST / PUT / DELETE | `/admin/experiences[/{id}]` | 경력 CRUD |
| POST / PUT / DELETE | `/admin/projects[/{id}]` | 프로젝트 CRUD |
| POST / PUT / DELETE | `/admin/skills[/{id}]` | 스킬 CRUD |
| GET | `/admin/contacts?status=&page=&size=` | 연락 메시지 목록 (페이지네이션) |
| PATCH | `/admin/contacts/{id}` | 상태 변경 (READ / ARCHIVED) |

- 콘텐츠 변경 성공 시 → 캐시 evict + Next.js 재검증 요청 (실패 시 3회 재시도 후 경고 로그, 관리자 API 응답 자체는 성공 처리)
- Swagger UI: `/swagger-ui` — 공개 API 그룹만 기본 노출, 관리자 API는 운영 환경에서 숨김

### 5.4 운영

| Path | 공개 여부 |
|---|---|
| `/actuator/health` | 공개 (헬스체크용) |
| `/actuator/info`, `/actuator/metrics` | 내부망/관리자만 |

## 6. 프론트엔드 명세

### 6.1 페이지

| 경로 | 렌더링 | 내용 |
|---|---|---|
| `/` | SSG + ISR (태그 재검증) | Hero → About → Experience → Projects → Skills → Contact |
| `/projects/[slug]` | SSG (`generateStaticParams`) + ISR | 프로젝트 상세 |
| `/about-this-site` | SSG | 사이트 아키텍처 다이어그램, 기술 선택 이유, GitHub·Swagger 링크 (F-15) |
| `/api/revalidate` | Route Handler | 백엔드 전용, `x-revalidate-secret` 헤더 검증 후 `revalidateTag` |
| 그 외 | `not-found.tsx` | 404 |

- `fetch(..., { next: { tags: ['projects'] } })`처럼 리소스별 태그 지정 → 관리자 수정 시 해당 태그만 재검증
- 백엔드 장애 시 마지막으로 생성된 페이지를 계속 제공

### 6.2 주요 컴포넌트

| 컴포넌트 | 기능 | 동작 |
|---|---|---|
| `Hero` | F-01 | 이름/역할/헤드라인/키워드, CTA 2개(이력서 `download`, 연락 `#contact`) |
| `ExperienceTimeline` | F-03 | 최신순, 재직 중이면 "현재", 기간(n년 m개월) 자동 계산 |
| `ProjectList` | F-04 | 태그 필터(다중 = OR, "전체" 초기화), URL 쿼리(`?tag=spring`) 동기화, 결과 0건 메시지 |
| `ProjectDetail` | F-05 | 헤더 + 지표 하이라이트(전→후) + Markdown 본문 + 이전/다음 링크 |
| `SkillBar` | F-06 | 아래 6.3 참고 |
| `ContactForm` | F-11 | 클라이언트 검증 → POST, 제출 중 버튼 비활성화, 성공/429/오류 메시지 |
| `Header` / `MobileNav` | F-09 | sticky 헤더, 모바일 메뉴 포커스 트랩 + ESC 닫기 |

### 6.3 기술 숙련도 막대 (F-06)

- 5단계 막대(5칸 세그먼트) + 단계명 + 한 줄 사용 경험
- 단계 기준 (사이트에 범례로 표기):

| 단계 | 이름 | 기준 |
|---|---|---|
| 1 | 학습 | 학습·토이 프로젝트 경험 |
| 2 | 기초 | 도움을 받아 기능 구현 가능 |
| 3 | 실무 | 실무에서 독립적으로 기능 개발·운영 |
| 4 | 숙련 | 설계 결정, 성능·장애 문제 해결 경험 |
| 5 | 전문 | 팀 내 기준 수립, 리뷰·멘토링 가능 |

- 접근성: `role="meter"`, `aria-valuemin=1`, `aria-valuemax=5`, `aria-valuenow`, `aria-label="Spring Boot 숙련도 4/5 (숙련)"` — 색만으로 정보 전달하지 않도록 단계명 텍스트 병기
- 애니메이션: 화면 진입 시 채워지는 효과, `prefers-reduced-motion`이면 생략

### 6.4 디자인 가이드

- 최대 폭 960px 중앙 정렬, 디자인 토큰으로 여백·색 일관 유지
- 브레이크포인트: `sm 640` / `md 768` / `lg 1024` / `xl 1280`
- 폰트: Pretendard(한글) + JetBrains Mono(코드), `next/font/local` 서브셋
- 컬러: 중립 배경 + 포인트 컬러 1개, 본문 대비 4.5:1 이상

### 6.5 SEO (F-10)

- `metadata` 기본값(title template, description, `metadataBase`), 상세는 `generateMetadata`
- `opengraph-image.tsx` (1200×630), JSON-LD `Person`, `sitemap.ts`(홈 + 모든 프로젝트), `robots.ts`

## 7. 보안 설계

| 항목 | 내용 |
|---|---|
| 인증 | `/auth/login` → JWT(HS256, 30분). 서명 키는 환경 변수 `JWT_SECRET` (256bit 이상) |
| 인가 | `/api/v1/admin/**` → `ROLE_ADMIN`, 나머지 공개 GET + `POST /contacts` 허용 |
| 비밀번호 | BCrypt, 로그인 실패 5회 → 15분 잠금 |
| CORS | `https://<domain>`(프로덕션), `http://localhost:3000`(로컬)만 허용 |
| 입력 | Bean Validation, Markdown 렌더링 시 raw HTML 비허용(XSS 방지) |
| 스팸 | Bucket4j IP 요청 제한 + 허니팟 필드 |
| 비밀값 | `.env`·GitHub Secrets·서버 환경 변수, 저장소에 커밋 금지 |
| 전송 | HTTPS 강제 (Nginx + Let's Encrypt) |

## 8. 성능 설계

- **프론트**: 전 페이지 SSG/ISR, 클라이언트 컴포넌트는 필터·메뉴·폼으로 한정, `next/image`, 홈 First Load JS < 120KB(gzip)
- **API**: 공개 조회 결과 Caffeine 캐시(TTL 10분, 관리자 수정 시 evict), N+1 방지, 페이지네이션은 관리자 목록에만 적용
- **DB**: `project.slug` UNIQUE 인덱스, `contact_message(status, created_at)` 복합 인덱스
- **검증**: k6로 공개 API p95 < 200ms 확인 (부하 테스트 결과를 F-15 페이지에 게시)

## 9. 테스트 전략

### 9.1 백엔드

| 종류 | 도구 | 대상 |
|---|---|---|
| 단위 | JUnit 5 + Mockito + AssertJ | Service 로직(정렬, 기간 계산, 상태 변경, 요청 제한 판정) |
| 영속성 | `@DataJpaTest` + Testcontainers(MySQL) | Repository 쿼리, fetch join, 태그 필터 쿼리, Flyway 마이그레이션 적용 |
| 웹 계층 | `@WebMvcTest` | 요청 검증, 에러 응답 포맷(ProblemDetail), 인증·인가(401/403) |
| 통합 | `@SpringBootTest` + Testcontainers | 로그인 → 관리자 CRUD → 공개 API 반영, 연락 폼 429 시나리오, 재검증 이벤트 발행 |
| 커버리지 | JaCoCo | line 80% 미만이면 빌드 실패 |

### 9.2 프론트엔드

| 종류 | 대상 |
|---|---|
| 단위 (Vitest + RTL) | 기간 계산 유틸, `ProjectList` 필터, `SkillBar` aria 속성, `ContactForm` 검증·상태별 메시지, `MobileNav` |
| E2E (Playwright, Desktop Chrome + Mobile Safari 뷰포트) | ① 홈 섹션 표시·앵커 이동 ② 이력서 PDF 200 응답 ③ 태그 필터 → URL 반영 → 새로고침 유지 ④ 상세 이동·이전/다음 ⑤ 연락 폼 성공/검증 오류 ⑥ 404 ⑦ 모바일 메뉴, 가로 스크롤 없음 ⑧ axe 위반 0건 |
| 품질 게이트 | Lighthouse CI 4개 카테고리 각 95점 미만이면 실패 |

- E2E는 Docker Compose로 백엔드+MySQL(시드 데이터)을 띄운 상태에서 실행

## 10. 배포 및 CI/CD

### 10.1 인프라 `[가정: 기본안, 확인 필요]`

| 구성 | 기본안 | 대안 |
|---|---|---|
| 프론트 | Vercel | — |
| 백엔드 | AWS EC2(t4g.small) 1대 — Docker + Nginx(TLS) | AWS Lightsail, Railway, Fly.io |
| DB | AWS RDS for MySQL 8.4 (db.t4g.micro) | 같은 EC2의 MySQL 컨테이너(비용 절감, 백업 직접 관리) |
| 이미지 저장소 | GitHub Container Registry(GHCR) | Amazon ECR |
| DNS | 도메인 등록 업체 또는 Route 53 | |

### 10.2 파이프라인

```
PR
 ├─ backend-ci  : Gradle build → 테스트(Testcontainers) → JaCoCo 검사
 └─ frontend-ci : install → lint → typecheck → unit → build → E2E → Lighthouse CI
                  + Vercel Preview 배포

main 머지
 ├─ backend-deploy : Docker 이미지 빌드 → GHCR push → EC2 SSH 접속
 │                   → 새 컨테이너 기동 → /actuator/health 확인 → 이전 컨테이너 종료
 │                   (헬스체크 실패 시 이전 이미지로 롤백)
 └─ Vercel Production 배포
```

- 브랜치 전략: `main`(배포) + 기능 브랜치 → PR → 머지
- DB 마이그레이션: 애플리케이션 기동 시 Flyway 자동 적용 (하위 호환되는 변경만 허용)

### 10.3 환경 변수

| 이름 | 위치 | 용도 |
|---|---|---|
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | 백엔드 | MySQL 접속 |
| `JWT_SECRET` | 백엔드 | JWT 서명 |
| `ADMIN_USERNAME`, `ADMIN_PASSWORD` | 백엔드 | 최초 관리자 생성 |
| `CORS_ALLOWED_ORIGINS` | 백엔드 | CORS |
| `REVALIDATE_URL`, `REVALIDATE_SECRET` | 백엔드 | Next.js 재검증 요청 |
| `IP_HASH_SALT` | 백엔드 | 연락 폼 IP 해시 |
| `API_BASE_URL` | 프론트(서버) | 빌드·재검증 시 API 호출 |
| `NEXT_PUBLIC_API_BASE_URL` | 프론트(브라우저) | 연락 폼 POST |
| `REVALIDATE_SECRET` | 프론트 | 재검증 요청 검증 |
| `NEXT_PUBLIC_SITE_URL` | 프론트 | 메타데이터 기준 URL |

## 11. 개발 작업 순서

1. 모노레포 초기화, `docker-compose.yml`(MySQL), 백엔드 Spring Boot 프로젝트 생성(Gradle, Java 25)
2. Flyway 스키마(V1) + 더미 시드(V2), JPA 엔티티·Repository
3. 공개 조회 API + 공통 예외 처리(ProblemDetail) + 캐시 + Swagger
4. 연락 폼 API + 요청 제한
5. Spring Security + JWT 로그인 + 관리자 CRUD API + 재검증 이벤트
6. 백엔드 테스트(단위·슬라이스·통합) + JaCoCo
7. Next.js 프로젝트 생성, OpenAPI 타입 생성, 레이아웃·디자인 토큰
8. 홈 섹션(Hero → About → Experience → Skills(막대) → Contact 폼)
9. 프로젝트 목록·필터·상세, `/about-this-site`, SEO, 404, `/api/revalidate`
10. 프론트 테스트(단위·E2E·Lighthouse)
11. CI 구성 → 인프라 구성 → 배포 → 도메인·HTTPS 연결

## 12. 확인이 필요한 사항

1. 배포 인프라 기본안(프론트 Vercel / 백엔드 AWS EC2 + RDS) 동의 여부, 혹은 비용을 줄이기 위해 DB를 EC2 컨테이너로 운영할지
2. 관리자 기능을 MVP에서는 **API + Swagger로만** 제공하고 관리자 화면은 차기(F-20)로 미루는 방식에 동의하는지
3. 숙련도 5단계 기준(6.3절) 문구 수정 여부
4. 빌드 도구 Gradle(Kotlin DSL) 사용 여부 (Maven 선호 시 변경)
