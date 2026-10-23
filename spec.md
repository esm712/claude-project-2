# SPEC — 개발자 포트폴리오 사이트 기술 명세

> 상태: **초안 (검토 대기)** · 작성일: 2026-10-23
> 요구사항은 [`prd.md`](./prd.md)를 따른다. 기능 ID(F-xx)는 PRD와 동일하다.

## 1. 기술 스택

| 영역 | 선택 | 선택 이유 |
|---|---|---|
| 프레임워크 | Next.js 15 (App Router) | SSG로 빠른 초기 로딩, SEO 메타데이터 API, 서비스 기업에서 널리 사용 |
| 언어 | TypeScript (strict) | 타입 안정성, 콘텐츠 스키마 검증 |
| 스타일 | Tailwind CSS v4 | 디자인 토큰 일관성, 작은 CSS 번들 |
| 콘텐츠 | MDX + frontmatter (`gray-matter`, `next-mdx-remote`) | 코드 수정 없이 콘텐츠 추가 |
| 스키마 검증 | Zod | 빌드 시 frontmatter 누락·오타 검출 |
| 단위 테스트 | Vitest + React Testing Library | 빠른 실행, Jest 호환 API |
| E2E 테스트 | Playwright | 실제 브라우저 기반 시나리오 검증, 모바일 뷰포트 |
| 품질 측정 | Lighthouse CI | PRD 성능·접근성 목표 자동 검증 |
| 린트/포맷 | ESLint, Prettier | 코드 일관성 |
| 패키지 매니저 | pnpm | 빠른 설치, 엄격한 의존성 |
| 배포 | Vercel | Next.js 최적화, PR 프리뷰 배포 |
| CI | GitHub Actions | lint → typecheck → test → build → E2E → Lighthouse |

## 2. 디렉터리 구조

```
.
├── content/
│   ├── profile.ts            # 이름, 소개, 연락처 등 (F-01, F-02, F-07)
│   ├── experience.ts         # 경력 (F-03)
│   ├── skills.ts             # 기술 스택 (F-06)
│   └── projects/
│       └── <slug>.mdx        # 프로젝트 상세 (F-04, F-05)
├── public/
│   ├── resume.pdf            # 이력서 (F-08)
│   └── images/
├── src/
│   ├── app/
│   │   ├── layout.tsx        # 공통 레이아웃, 메타데이터, 폰트
│   │   ├── page.tsx          # 홈 (Hero, About, Experience, Projects, Skills, Contact)
│   │   ├── projects/
│   │   │   └── [slug]/page.tsx   # 프로젝트 상세 (generateStaticParams)
│   │   ├── not-found.tsx
│   │   ├── opengraph-image.tsx
│   │   ├── sitemap.ts
│   │   └── robots.ts
│   ├── components/
│   │   ├── sections/         # Hero, About, Experience, ProjectList, Skills, Contact
│   │   ├── ui/               # Button, Tag, Card, Section
│   │   └── layout/           # Header, Footer, MobileNav
│   ├── lib/
│   │   ├── content.ts        # MDX 로딩/파싱
│   │   └── schema.ts         # Zod 스키마
│   └── types/
├── tests/
│   ├── unit/
│   └── e2e/
└── .github/workflows/ci.yml
```

## 3. 페이지 및 라우팅

| 경로 | 렌더링 | 내용 |
|---|---|---|
| `/` | SSG | 단일 페이지 섹션: Hero → About → Experience → Projects → Skills → Contact |
| `/projects/[slug]` | SSG (`generateStaticParams`) | 프로젝트 상세 |
| `/resume.pdf` | 정적 파일 | 이력서 |
| `/sitemap.xml`, `/robots.txt` | 빌드 생성 | SEO |
| 기타 | `not-found.tsx` | 404 페이지 (홈으로 이동 링크) |

- 헤더 내비게이션: 섹션 앵커 링크(`/#about`, `/#projects` …), 현재 섹션 하이라이트
- 모바일(< 768px): 햄버거 메뉴

## 4. 데이터 모델

```ts
// src/lib/schema.ts
export const ProfileSchema = z.object({
  name: z.string(),
  role: z.string(),                 // 예: "Frontend Engineer"
  headline: z.string(),             // 한 줄 소개
  keywords: z.array(z.string()).length(3),
  about: z.string(),                // Markdown
  email: z.string().email(),
  links: z.object({
    github: z.string().url(),
    linkedin: z.string().url().optional(),
    blog: z.string().url().optional(),
  }),
  resumePath: z.string().default("/resume.pdf"),
});

export const ExperienceSchema = z.object({
  company: z.string(),
  role: z.string(),
  startDate: z.string(),            // "YYYY-MM"
  endDate: z.string().nullable(),   // null = 재직 중
  summary: z.string(),
  achievements: z.array(z.string()), // 수치 포함 성과
  techStack: z.array(z.string()),
});

export const SkillGroupSchema = z.object({
  category: z.enum(["Frontend", "Backend", "Infra", "Collaboration"]),
  items: z.array(z.object({
    name: z.string(),
    description: z.string(),        // 사용 경험 서술 (숙련도 % 바 사용 안 함)
  })),
});

// content/projects/<slug>.mdx frontmatter
export const ProjectFrontmatterSchema = z.object({
  title: z.string(),
  summary: z.string(),              // 카드용 1~2문장
  period: z.string(),               // "2024.03 – 2024.08"
  role: z.string(),                 // 예: "프론트엔드 리드 (3인 팀)"
  techStack: z.array(z.string()),
  thumbnail: z.string(),
  metrics: z.array(z.object({       // 결과 지표 하이라이트
    label: z.string(),              // 예: "LCP"
    before: z.string().optional(),
    after: z.string(),
  })).default([]),
  links: z.object({
    github: z.string().url().optional(),
    demo: z.string().url().optional(),
  }).default({}),
  featured: z.boolean().default(false),
  order: z.number(),
});
```

프로젝트 MDX 본문은 다음 섹션(H2)을 권장 템플릿으로 한다:
`개요` / `문제 정의` / `나의 역할` / `기술적 의사결정` / `트러블슈팅` / `결과` / `회고`

- 빌드 시 Zod 검증 실패 → **빌드 실패** 처리 (잘못된 콘텐츠의 배포 방지)

## 5. 컴포넌트 명세 (주요)

| 컴포넌트 | 기능 ID | 동작 |
|---|---|---|
| `Hero` | F-01 | 이름/역할/헤드라인/키워드 표시, CTA 버튼 2개(이력서 다운로드 `download` 속성, 연락하기 `#contact`) |
| `About` | F-02 | Markdown 렌더링 |
| `ExperienceTimeline` | F-03 | 최신순 정렬, 재직 중이면 "현재" 표시, 기간(n년 m개월) 자동 계산 |
| `ProjectList` | F-04 | `order` 오름차순 정렬, 기술 태그 필터(다중 선택 = OR 조건, "전체" 초기화), 필터 상태 URL 쿼리(`?tag=react`) 동기화 |
| `ProjectCard` | F-04 | 썸네일, 제목, 요약, 태그, 상세 링크 |
| `ProjectDetail` | F-05 | frontmatter 헤더 + 지표 하이라이트 + MDX 본문 + 이전/다음 프로젝트 링크 |
| `Skills` | F-06 | 카테고리별 그룹 |
| `Contact` | F-07 | mailto 링크, 외부 링크는 `target="_blank" rel="noopener noreferrer"` |
| `Header` / `MobileNav` | F-09 | sticky 헤더, 모바일 메뉴 열림 시 포커스 트랩 + ESC 닫기 |

## 6. 디자인 가이드

- 레이아웃: 최대 폭 960px 중앙 정렬, 섹션 간 여백 일관(디자인 토큰)
- 브레이크포인트: `sm 640` / `md 768` / `lg 1024` / `xl 1280`
- 폰트: Pretendard (한글), 서브셋 + `next/font/local`
- 컬러: 중립 배경 + 포인트 컬러 1개, 본문 대비 4.5:1 이상
- 모션: `prefers-reduced-motion` 존중, 스크롤 페이드인 정도로 최소화

## 7. SEO / 메타데이터 (F-10)

- `layout.tsx`에서 `metadata` 기본값(title template, description, `metadataBase`)
- 프로젝트 상세는 `generateMetadata`로 개별 title/description/OG
- `opengraph-image.tsx`로 OG 이미지 동적 생성 (1200×630)
- JSON-LD `Person` 스키마 삽입
- `sitemap.ts`: 홈 + 모든 프로젝트 상세

## 8. 성능 설계

- 전 페이지 SSG, 클라이언트 컴포넌트는 필터·모바일 메뉴 등 상호작용 부분으로 한정
- 이미지: `next/image`, WebP/AVIF, 명시적 width/height (CLS 방지), Hero 이미지만 `priority`
- 서드파티 스크립트 최소화 (분석 도구는 P1에서 `afterInteractive`로 로드)
- 번들 예산: 홈 First Load JS < 120KB (gzip)

## 9. 테스트 전략

### 9.1 단위 테스트 (Vitest + RTL)
| 대상 | 검증 내용 |
|---|---|
| `lib/schema.ts` | 유효/무효 frontmatter 파싱 결과 |
| `lib/content.ts` | 프로젝트 목록 정렬, slug 조회, 없는 slug 처리 |
| 기간 계산 유틸 | 재직 중/종료, 월 경계 계산 |
| `ProjectList` | 태그 필터 선택/해제/초기화, 결과 0건 메시지 |
| `MobileNav` | 열기/닫기, ESC, 포커스 트랩 |

### 9.2 E2E 테스트 (Playwright — Desktop Chrome, Mobile Safari 뷰포트)
1. 홈 로드 → 모든 섹션 표시, 헤더 내비게이션 앵커 이동
2. 이력서 다운로드 링크가 200 응답의 PDF를 가리킴
3. 프로젝트 태그 필터 → 카드 수 변화 → URL 쿼리 반영 → 새로고침 후 유지
4. 프로젝트 카드 클릭 → 상세 페이지 → 이전/다음 이동
5. 존재하지 않는 경로 → 404 페이지
6. 모바일 뷰포트에서 햄버거 메뉴 동작, 가로 스크롤 없음
7. `@axe-core/playwright`로 주요 페이지 접근성 위반 0건

### 9.3 품질 게이트
- Lighthouse CI: 4개 카테고리 각 95점 미만이면 CI 실패
- 커버리지 목표: `lib/` 90% 이상, 컴포넌트 70% 이상

## 10. CI/CD 및 배포

```
PR 생성/갱신
  └─ GitHub Actions: install → lint → typecheck → unit test → build → E2E → Lighthouse CI
  └─ Vercel: Preview 배포 (PR 코멘트로 URL)
main 머지
  └─ Vercel: Production 배포
```

- 브랜치 전략: `main`(배포) + 기능 브랜치 → PR → 머지
- 환경 변수: MVP에서는 불필요 (`NEXT_PUBLIC_SITE_URL`만 사용)
- 도메인: 기본 `*.vercel.app`, 커스텀 도메인은 확정 시 연결 `[가정]`

## 11. 개발 작업 순서 (개발 단계 계획)

1. 프로젝트 초기화 (Next.js, TS, Tailwind, ESLint/Prettier, Vitest, Playwright)
2. 스키마·콘텐츠 로더 + 더미 콘텐츠
3. 레이아웃(Header/Footer/MobileNav) + 디자인 토큰
4. 홈 섹션 (Hero → About → Experience → Skills → Contact)
5. 프로젝트 목록·필터·상세
6. SEO(메타데이터, OG, sitemap, JSON-LD), 404
7. 테스트 작성 및 Lighthouse 최적화
8. CI 구성 → Vercel 연결 → 배포

## 12. 확인이 필요한 사항

1. 기술 스택(Next.js / Tailwind / Vercel) 확정 여부
2. 홈을 단일 페이지(섹션 스크롤)로 구성하는 방식에 동의하는지
3. 기술 스택 숙련도를 % 바 대신 "사용 경험 서술"로 표현하는 방식에 동의하는지
4. 프로젝트 상세 본문 템플릿(7개 섹션) 수정 여부
