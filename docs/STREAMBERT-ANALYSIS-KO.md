# Streambert 전수조사 분석 & 활용 정리 (한국어)

> 작성일: 2026-09-20
> 대상 저장소: **[bmshin94/streambert](https://github.com/bmshin94/streambert)** (포크)
> 원본 저장소: **[truelockmc/streambert](https://github.com/truelockmc/streambert)**
> 원본 미러(릴리스 배포처): **[codeberg.org/truelockmc/streambert](https://codeberg.org/truelockmc/streambert)**
> AUR 패키지: **[streambert-bin](https://aur.archlinux.org/packages/streambert-bin)**
> 분석 기준 버전: `v2.6.0`

---

## 목차

1. [프로젝트 정체 요약](#1-프로젝트-정체-요약)
2. [기술 스택 & 아키텍처](#2-기술-스택--아키텍처)
3. [폴더별 상세 분석](#3-폴더별-상세-분석)
4. [작동 원리 (쉬운 설명)](#4-작동-원리-쉬운-설명)
5. [설치 및 사용법](#5-설치-및-사용법)
6. [플러그인? 스킬? MCP?](#6-플러그인-스킬-mcp)
7. [API 토큰 정리](#7-api-토큰-정리)
8. [왜 GitHub에서 유명한가](#8-왜-github에서-유명한가)
9. [로컬 AI 에이전트 구축에 도움이 되는가](#9-로컬-ai-에이전트-구축에-도움이-되는가)
10. [React / PHP로 만들 수 있는가](#10-react--php로-만들-수-있는가)
11. [수익화 아이디어](#11-수익화-아이디어)
12. [리스크 & 주의사항](#12-리스크--주의사항)
13. [참고 링크](#13-참고-링크)

---

## 1. 프로젝트 정체 요약

**Streambert = 영화 / TV 시리즈 / 애니를 스트리밍하고 다운로드하는 크로스플랫폼 Electron 데스크톱 앱.**

| 항목 | 값 |
|---|---|
| 종류 | 독립 실행형 데스크톱 애플리케이션 (플러그인/스킬/MCP 아님) |
| 라이선스 | GPL-3.0 (포크도 반드시 오픈소스 유지) |
| 버전 | 2.6.0 |
| 총 코드량 | 약 28,000줄 (React + IPC 모듈) |
| 배포 포맷 | exe / dmg(universal) / deb / rpm / AppImage / pacman |
| 중앙 서버 | 없음 (완전 로컬, 트래킹 0) |
| 핵심 셀링포인트 | 광고·트래커 완전 차단 |

핵심 포인트: **영상 파일을 직접 호스팅하지 않는다.** 메타데이터는 공식 API에서,
영상은 제3자 스트리밍 사이트를 `<webview>`로 임베드해서 가져온다.

---

## 2. 기술 스택 & 아키텍처

| 계층 | 기술 | 주요 파일 |
|---|---|---|
| 데스크톱 셸 | Electron 40 | `index.js` (메인 프로세스) |
| 보안 브릿지 | contextBridge / IPC | `preload.js`, `popout-preload.js` |
| UI | React 18 | `src/App.jsx` (1,441줄) |
| 빌드 | Vite 7 + terser | `vite.config.js` |
| 스타일 | 순수 CSS (프레임워크 없음) | `src/styles/global.css` (3,910줄) |
| 패키징 | electron-builder 26 | `package.json` > `build` |

### 프로세스 구조

```
[메인 프로세스 index.js]  <--- IPC --->  [렌더러 (React)]
  ├ webRequest 가로채기 (광고차단 + m3u8/vtt 스니핑)
  ├ child_process (다운로더 바이너리 / mpv / VLC 실행)
  ├ fs (백업, 자막 파일)
  ├ safeStorage (OS 키체인 암호화)
  └ BrowserWindow (메인 창 + PiP 팝아웃 창)
```

### 성능 최적화 플래그 (`index.js`)

```js
--max-old-space-size=256   // V8 힙 상한
--expose-gc                // 플레이어 종료 시 강제 GC
renderer-process-limit=3   // 렌더러 프로세스 제한
disk-cache-size=80MB       // 디스크 캐시 상한
NetworkServiceInProcess2   // 유틸리티 프로세스 하나 절약
```

---

## 3. 폴더별 상세 분석

### `src/pages/` — 화면 6개

| 파일 | 줄 수 | 역할 |
|---|---|---|
| `SettingsPage.jsx` | 4,508 | 설정 전체 (테마/자막/백업/부모통제/디스코드/업데이트) |
| `TVPage.jsx` | 2,716 | 드라마·애니 상세 + 시즌/에피소드 플레이어 |
| `MoviePage.jsx` | 1,349 | 영화 상세 + 플레이어 |
| `DownloadsPage.jsx` | 1,343 | 다운로드 큐 관리 UI |
| `HomePage.jsx` | 442 | 트렌딩 홈 |
| `LibraryPage.jsx` | 214 | 보관함 (시청기록/저장) |

### `src/ipc/` — 메인 프로세스 기능 모듈 (핵심 기술)

| 파일 | 줄 수 | 내용 |
|---|---|---|
| `allmanga.js` | 999 | AllAnime API를 POST로 우회(Cloudflare JS 챌린지 회피) + 자체 hex 치환 암호표로 난독화 URL 복호화 → .mp4 직링크 추출 (`ani-cli` 기법 차용) |
| `downloads.js` | 980 | 다운로드 큐, 외부 바이너리(`vid-dl-cli-only`) spawn, 진행률 파싱, 임시파일 청소, 강제종료 |
| `player.js` | 753 | 자동 업데이트 다운로드(origin+경로 검증), mpv/VLC 외부 실행, 경로 화이트리스트 검증 |
| `subtitles.js` | 529 | 자막 검색 2소스(SubDL + Wyzie), ZIP 자동 해제, 로컬 파일 첨부 |
| `storage.js` | 267 | safeStorage 기반 API키 암호화 저장 + 예약 백업 + AppImage 키 마이그레이션 |
| `discordRpc.js` | 236 | Discord Rich Presence ("지금 보는 중") |
| `blockStats.js` | 88 | 차단한 광고/트래커 카운팅 |

### `src/utils/` — 로직 유틸 (24개)

- `api.js` (499줄) — TMDB + AniList 통합 게이트웨이
  - 동시요청 4개 제한 세마포어 (`_acquireSlot` / `_releaseSlot`)
  - 5분 메모리 캐시 + 7일 localStorage 캐시
  - `PLAYER_SOURCES` 정의 (videasy / vidsrc / vidking / allmanga)
- `gamepad.js`, `gamepadSpatialNav.js`, `useGamepadNav.js`, `playerGamepadScript.js` — 게임패드 지원 (공간 탐색 네비게이션)
- `aniSkip.js` — 애니 오프닝/엔딩 자동 스킵
- `ageRating.js` — 연령등급 기반 부모통제
- `backup.js`, `updates.js`, `appearance.js`, `storage.js`, `episodeMappings.js` 등

### `src/components/` — 컴포넌트 17개

`DownloadModal`(1,237) / `UpdateModal`(876) / `SubtitleDownloaderModal`(679) /
`TrendingCarousel`(446) / `Icons`(363) / `WyzieKeyModal`(354) /
`KeyboardShortcutsModal`(314) / `Sidebar`(277) 등

### `.github/`

- `codeql.yml` — CodeQL 보안 스캔 (main push / PR / 매주 월요일 05:45)
- `build.yml` — macOS universal 빌드 (수동 트리거)
- `ISSUE_TEMPLATE/` — 버그 리포트 / 기능 요청 템플릿

---

## 4. 작동 원리 (쉬운 설명)

### 비유: 음식 배달 앱

- 메뉴 사진·설명 = TMDB에서 가져옴 (직접 요리 안 함)
- 실제 음식 = 제휴 식당(스트리밍 사이트)이 만듦
- 배달원이 몰래 레시피 메모 = m3u8 링크 낚아채기

**Streambert는 영상을 1바이트도 갖고 있지 않다.**

### 데이터 흐름

```
사용자가 검색
      ↓
TMDB API → 포스터 / 줄거리 / 평점 / 트렌딩    (합법 공식 API)
      ↓ (애니면 AniList GraphQL로 전환)
"재생" 클릭
      ↓
<webview src="https://www.vidking.net/embed/tv/{id}/{season}/{ep}">
      ↓
메인 프로세스가 webRequest로 트래픽 감시
      ├ 광고 도메인 요청  → cancel: true (차단 카운터 +1)
      └ *.m3u8 / *.vtt   → URL 저장 후 통과
      ↓
"다운로드" 클릭 → vid-dl-cli-only 실행 (ffmpeg 필요) → mp4 파일
```

### m3u8 스니핑이란?

HLS 스트리밍은 영상을 10초 단위 조각으로 쪼개고, 그 조각 목록을 `.m3u8`
플레이리스트로 제공한다. **플레이리스트 하나만 확보하면 영상 전체를 받을 수 있다.**

```js
// index.js (요지)
const MEDIA_URLS = ["*://*/*.m3u8*", "*://*/*.vtt*"];
playerSession.webRequest.onBeforeRequest({ urls: MEDIA_URLS }, (details, cb) => {
  if (details.url.includes(".m3u8")) {
    mainWindow.webContents.send("m3u8-found", details.url);  // 낚아챔
  }
  cb({});  // 통과 → 재생은 정상 동작
});
```

### 왜 웹앱이 아니고 Electron인가

| 필요 기능 | 브라우저 | Electron |
|---|---|---|
| 타 도메인 트래픽 관찰 | 불가 (보안 정책) | 가능 (`webRequest`) |
| `X-Frame-Options` / CSP 헤더 제거 | 불가 | 가능 (`onHeadersReceived`) |
| 광고 도메인 차단 | 확장프로그램 필요 | 코드로 바로 |
| 파일 다운로드 + ffmpeg 실행 | 불가 | `child_process` |
| OS 네이티브 알림 / 키체인 | 제한적 | 가능 |

### 광고차단 방식

`index.js`에 **50여 개 도메인이 하드코딩된 블록리스트**(`BLOCKED_HOSTS`)를
`onBeforeRequest`로 전부 차단. 광고 도메인이 랜덤 문자열이라 수동 업데이트가 계속 필요함.

### 부가 기능

- PiP 팝아웃 창 (항상 위 고정, 별도 BrowserWindow)
- 자막 다운로드 + 로컬 파일 자동 첨부
- 게임패드 조작 (TV 연결 시나리오)
- 시즌별 아이콘 (크리스마스 / 할로윈 / 프라이드)
- 예약 자동 백업 (startup / daily / weekly / monthly, 개수 제한 프루닝)
- 인앱 자동 업데이트 (GitHub + Codeberg 이중 소스)
- 부모통제 (연령등급 필터)
- Discord Rich Presence

---

## 5. 설치 및 사용법

### 방법 A: 프리빌트 바이너리 (약 5분)

**1단계 — TMDB 토큰 발급 (필수)**

1. [themoviedb.org](https://www.themoviedb.org/signup) 무료 가입
2. [API 설정](https://www.themoviedb.org/settings/api) → "click here" 클릭
3. "Personal use only" 선택
4. 양식 작성 → Subscribe
5. **API Read Access Token** 복사 (`eyJ`로 시작하는 긴 것. 짧은 API Key 아님!)

> 상세 가이드: [`tmdb-tutorial.md`](../tmdb-tutorial.md)

**2단계 — 설치**

릴리스: https://codeberg.org/truelockmc/streambert/releases/latest

| OS | 설치 |
|---|---|
| Windows | `Streambert Setup *.exe` 실행 |
| macOS | `Streambert-*-universal.dmg` → Applications로 드래그 |
| Debian/Ubuntu | `sudo dpkg -i streambert_*.deb` |
| Arch | `sudo pacman -U streambert-*.pacman` 또는 `yay -S streambert-bin` |
| Fedora/RHEL | `sudo rpm -i streambert-*.rpm` |
| 범용 | `chmod +x Streambert-x64.AppImage && ./Streambert-x64.AppImage` |

**3단계 — 첫 실행**: 토큰 붙여넣기 → "Let's go" (OS 키체인에 암호화 저장, 1회만)

**4단계 — 다운로드 기능 (선택)**

1. [vid-dl-cli-only](https://github.com/truelockmc/vid-dl-cli-only/releases/latest) 바이너리 배치
2. [ffmpeg](https://ffmpeg.org/download.html) 설치
3. Settings → Downloads에서 바이너리 경로 지정

### 방법 B: 소스 빌드 (개발용)

```bash
# 요구사항: Node.js >= 22.12.0
git clone https://github.com/bmshin94/streambert.git
cd streambert
npm install

npm start            # vite build + electron .  (가장 자주 쓸 명령)
npm run dev          # vite build --watch (변경 감지)

npm run dist:win     # Windows exe
npm run dist:mac     # macOS universal dmg
npm run dist:appimage
npm run dist:deb
npm run dist:rpm
npm run dist:arch
npm run dist         # 전 플랫폼
```

**Arch 빌드 에러 대응**
- `libcrypt.so.1` → `sudo pacman -S libxcrypt-compat`
- `http-parser` → `yay -S http-parser`

**앱 내 단축키**: `Ctrl+K` / `Ctrl+F` 검색, `Ctrl+Z` 뒤로, `?` 단축키 목록

---

## 6. 플러그인? 스킬? MCP?

**셋 다 아니다. 완전히 독립된 데스크톱 애플리케이션이다.**

| 구분 | 정체 | 실행 방식 | Streambert |
|---|---|---|---|
| 플러그인 | 호스트 앱 확장 | 호스트가 로드 | 아님 (호스트 없음) |
| 스킬 | Claude용 마크다운 지침서 | Claude가 읽음 | 아님 |
| MCP 서버 | AI에 도구 제공 프로토콜 | stdio/HTTP로 AI와 통신 | 아님 |
| **독립 앱** | 혼자 실행되는 프로그램 | 사용자가 실행 | **이것** |

### 근거

```json
// package.json
"main": "index.js",
"devDependencies": { "electron": "^40.4.1", "electron-builder": "^26.7.0" }
```

MCP였다면 있어야 할 것 — **전부 없음**: `@modelcontextprotocol/sdk` 의존성,
`mcp.json` / `server.json`, stdio transport 코드, tool/resource/prompt 정의.
스킬이었다면 있어야 할 `SKILL.md` + frontmatter도 없음.

### 헷갈리는 이유

- 이 저장소의 `CLAUDE.md`(카리나 페르소나)는 **포크 후 직접 추가한 개발 메모**로,
  앱 실행과는 무관하다. (커밋 `6a44761`, PR #1 머지 `f02c617`)
- `src/ipc/` 의 **IPC = Inter-Process Communication**(Electron 표준 기능)이며
  **MCP와는 완전히 다른 개념**이다.

---

## 7. API 토큰 정리

| 토큰 | 필수 | 비용 | 용도 | 저장 위치 |
|---|---|---|---|---|
| TMDB Read Access Token | **필수** | 무료 | 포스터/줄거리/검색/트렌딩/평점 | OS 키체인 (safeStorage) |
| Wyzie API Key | 선택 | 무료(이메일 인증) | 자막 검색 (프리미엄 엔드포인트) | OS 키체인 |
| SubDL API Key | 선택 | 무료 | 자막 검색 (대체 소스) | OS 키체인 |
| AniList | 불필요 | 무료 | 애니 메타데이터 (공개 GraphQL) | - |
| 스트리밍 소스 | 불필요 | - | Videasy / VidSrc / Vidking / AllManga | - |

### 저장 구현 (배울 만한 패턴)

```js
// src/ipc/storage.js
if (safeStorage.isEncryptionAvailable()) {
  store[key] = safeStorage.encryptString(value).toString("base64");
  // Windows: DPAPI / macOS: Keychain / Linux: libsecret
} else {
  store[key] = Buffer.from(value, "utf8").toString("base64");  // 폴백
}
```

저장 파일: `<userData>/secure-store.json` (암호화 상태)

**AppImage 특수 대응**: AppImage 업데이트 시 바이너리 교체로 기존 복호화가
불가능해지는 문제를, 종료 직전 평문 임시파일(`mode 0o600`) → 다음 실행 시 읽고
즉시 삭제 → 재암호화, `quit` 이벤트에 안전망 삭제 로직으로 해결했다.

- Wyzie 키 발급 가이드: [`wyzie-tutorial.md`](../wyzie-tutorial.md)
- **주의**: TMDB 무료 티어는 비상업적 사용 조건 → 상업화 시 별도 라이선스 필요

---

## 8. 왜 GitHub에서 유명한가

1. **Trendshift 등재** (레포 #31115) — GitHub 트렌딩 진입 이력
2. **"Zero Ads" 후크** — 무료 스트리밍 사이트의 최악(팝업, 가짜 다운로드 버튼,
   악성코드)을 정면으로 해결
3. **영리하게 설계된 법적 포지셔닝** — GPL-3.0, Codeberg 미러(DMCA 대비),
   중앙 서버 0개, 긴 법적 디스클레이머, "수익 없음" 명시
4. **진짜 크로스플랫폼** — 6개 포맷 + AUR 패키지 (리눅스 유저 충성도)
5. **실제로 좋은 코드 품질** — `contextIsolation: true`, CodeQL 스캔,
   업데이터 origin 검증, 경로 화이트리스트 + 심볼릭링크 방어,
   `Fixed multiple security issues (#149)` 같은 커밋
6. **기능 물량** — 스트리밍 + 다운로드 + 자막 + 라이브러리 + 애니 전용 소스 +
   게임패드 + Discord RPC + PiP + 예약백업 + 부모통제 + 오프닝 스킵
7. **애니 지원 = 거대한 니치** — `ani-cli` 팬층 흡수
8. **활발한 커뮤니티 운영** — 이슈 번호 182+, 외부 PR 지속 머지

**솔직한 결론**: 가장 큰 이유는 "무료로 영화를 볼 수 있다"는 수요다.
즉 **좋은 기술 + 수요 폭발 카테고리 + 똑똑한 법적 포지셔닝**의 조합.

---

## 9. 로컬 AI 에이전트 구축에 도움이 되는가

### 도움 안 되는 부분

LLM 호출 코드, 프롬프트/컨텍스트 관리, 에이전트 루프, tool use 패턴,
벡터DB/RAG, MCP 클라이언트·서버 — **전부 없음.** AI 로직은 배울 게 없다.

### 도움 되는 부분 (데스크톱 앱 인프라는 거의 완성품)

| 에이전트에 필요한 것 | 참고 파일 | 재사용도 |
|---|---|---|
| API 키 안전 저장 | `src/ipc/storage.js` | 높음 (LLM 키에 그대로) |
| 외부 프로세스 실행 + 실시간 출력 | `src/ipc/downloads.js` | 높음 (도구/스크립트 실행) |
| 안전한 IPC 브릿지 | `preload.js` | 높음 (화이트리스트 패턴) |
| 레이트리밋 큐 | `src/utils/api.js` (`_acquireSlot`) | 높음 (LLM 동시요청 제한) |
| 다단 캐시 | `src/utils/api.js` | 높음 (응답 캐싱) |
| 스트리밍 진행률 UI | `download-progress` IPC | 높음 (토큰 스트리밍) |
| **작업 큐 UI** | `src/pages/DownloadsPage.jsx` | **매우 높음** |
| 자동 업데이트 (보안검증) | `src/ipc/player.js` | 높음 (배포 필수) |
| 파일 경로 검증 | `validateMediaPath()` | **매우 높음** (에이전트 샌드박싱) |
| 에러 바운더리 | `src/components/ErrorBoundary.jsx` | 높음 |
| OS 알림 | `show-notification` IPC | 높음 |
| 크로스플랫폼 패키징 | `package.json` > `build` | 높음 |

### 제안 구조

```
index.js (메인)
  ├ ipc/llm.js      ← Anthropic/OpenAI/Ollama 호출 (api.js 큐 패턴 재사용)
  ├ ipc/tools.js    ← 도구 실행 (downloads.js spawn 패턴 재사용)
  ├ ipc/storage.js  ← 거의 그대로 (API키 암호화)
  ├ ipc/mcp.js      ← MCP 서버 연결 (신규)
  └ ipc/files.js    ← validateMediaPath() 응용 경로 샌드박싱

src/
  ├ ChatPage.jsx     ← 대화 UI
  ├ ToolLogPage.jsx  ← DownloadsPage.jsx 구조 이식 (작업큐 = 다운로드큐)
  └ SettingsPage.jsx ← 모델선택/키관리 (구조 참고)
```

`DownloadsPage.jsx`의 "여러 작업 동시 실행 + 개별 진행률 + 취소 + 재시도 +
결과물 열기" UI는 **에이전트 작업 큐 UI와 구조가 동일하다.**

**평가: AI 로직 0점 / 데스크톱 앱 인프라 95점**

---

## 10. React / PHP로 만들 수 있는가

### React — 이미 100% React다

`src/App.jsx`(1,441줄), 컴포넌트 17개, 페이지 6개, `useState`/`useEffect`/
`useMemo`/`useRef`/`lazy`/`Suspense` 전부 사용 중.

단, **순수 브라우저 React만으로는 핵심 기능이 막힌다**:

| 기능 | 브라우저 React | Electron React |
|---|---|---|
| TMDB 메타데이터 | 가능 | 가능 |
| UI 전체 | 가능 | 가능 |
| 스트리밍 사이트 임베드 | 부분적 (`X-Frame-Options`로 거부) | 가능 (헤더 제거) |
| **m3u8 스니핑** | **불가능** | 가능 |
| 광고 차단 | 불가 | 가능 |
| 파일 다운로드 / ffmpeg | 불가 | 가능 |

`index.js`가 응답에서 `X-Frame-Options`와 `Content-Security-Policy`를
**삭제**하고 있다 — 브라우저에서는 절대 불가능한 동작.

### PHP — 가능하지만 "다른 제품"이 된다

PHP는 서버 사이드 언어라 데스크톱 앱이 안 된다. 가능한 구조:

```
[브라우저 (React/Vue)]
      ↕ HTTP
[PHP 서버 (Laravel/Slim)]
      ├ TMDB 프록시 (API키 서버 보관)
      ├ MySQL: 유저별 시청기록/보관함 (다중 디바이스 동기화)
      ├ Guzzle 스크레이핑
      └ exec("ffmpeg ...") 서버측 다운로드
```

| PHP가 더 좋은 점 | PHP로 불가능한 점 |
|---|---|
| API키 서버 보관 (유저 발급 불필요) | **m3u8 트래픽 스니핑 불가** |
| 다중 기기 동기화 | 광고 차단 불가 |
| 설치 불필요 | iframe 헤더 우회 불가 |
| MySQL 통계/추천 | 서버 비용 발생 |
| 즉시 업데이트 배포 | **트래픽이 전부 내 서버 경유 → 법적 책임 직격** |

### 권장 순위

1. **Electron + React** — 현재 방식. 이 저장소를 개조하는 게 가장 빠름
2. **Tauri + React** — Rust 기반. 용량 150~250MB → 5~15MB, RAM 300~500MB →
   80~150MB. React 코드는 거의 그대로, `src/ipc/*` 만 Rust로 재작성.
   Tauri도 webview 요청 가로채기 지원
3. PHP + React (웹) — 다른 제품이 되고 법적 리스크 최대

**하이브리드 대안**: Electron 앱(재생/차단/다운로드는 로컬) +
PHP 백엔드(계정/동기화/추천/통계/결제) → PHP 실력 활용 + 리스크 분리

---

## 11. 수익화 아이디어

### 먼저: 이 앱을 "그대로" 수익화하면 안 되는 이유

- 저작권 침해 (영리 목적은 형사 가중 사유)
- TMDB 약관 위반 (무료 티어는 비상업적 조건 → 키 즉시 정지)
- GPL-3.0 위반 가능 (소스 비공개 시)
- 한국 저작권법 제136조: 영리 목적 상습 → 5년 이하 징역 또는 5천만원 이하 벌금
- 결제 게이트웨이(Stripe/PayPal/토스) 계정 영구 정지
- 앱스토어 전면 리젝

전례: Popcorn Time 해산, Showbox/Terrarium TV 폐쇄, 국내 누누티비 운영자 기소.
원본 저자가 후원 링크조차 달지 않고 "돈 안 번다"고 명시한 이유가 이것이다.

### 전략: 콘텐츠가 아니라 "기술"을 판다

#### TIER 1 — 최우선 추천 (리스크 없음)

**① 로컬 AI 에이전트 데스크톱 앱**

- 근거: Streambert 껍데기 90% 재사용 가능 (9장 참고), AI 데스크톱 앱 시장 성장,
  법적 리스크 0
- 제품 예: 사내 문서 AI 검색기(온프레미스), 개발자용 로컬 에이전트 GUI,
  법무/의료 문서 요약기, 번역 워크벤치
- 온프레미스가 프리미엄 가격을 받는 이유: 기업이 데이터를 외부로 안 보내려 함
- 예상: 월 100만 ~ 1,000만원 / 개발 2~4개월

**② Electron 프로덕션 보일러플레이트 판매**

- 구성: OS 키체인 키저장, 보안 검증 자동 업데이터, 경로 샌드박싱,
  레이트리밋 큐, 3단 캐시, 작업 큐 UI, 커스텀 타이틀바, PiP 창,
  6포맷 빌드 설정, 게임패드 네비, 테마 시스템, 한/영 문서
- 판매처: Gumroad($79 개인 / $249 팀), GitHub Sponsors, 인프런/클래스101 강의
- 예상: 월 $500 ~ $3,000 / 개발 3~6주 (난이도 최저, 패시브 인컴)
- **GPL 주의**: 코드를 복사하면 GPL이 전염됨 → **패턴만 배우고 직접 작성**

**③ 합법 미디어 센터 / 자체 서버 플레이어**

- 소스 교체: VidSrc/Videasy → Jellyfin/Plex/Emby, AllManga → 공식 API(Laftel 등),
  m3u8 스니핑 → 로컬 NAS 파일, TMDB는 유지(상업용 라이선스)
- 재사용: 라이브러리/시청기록/진행률, 트렌딩 카루셀, 자막 다운로더,
  게임패드 네비(TV용), 다운로드 매니저 UI, Discord RPC, 오프닝 스킵
- 수익: Steam/Gumroad $19 단매, Pro 구독 $4/월, 시놀로지 패키지센터 등록
- 근거: Jellyfin 공식 클라이언트 UI 불만 → "예쁜 Jellyfin 클라이언트" 수요 존재
- 예상: 월 $1,000 ~ $5,000 / 개발 2~3개월

#### TIER 2 — 좋은 선택

**④ 한국 특화 OTT 통합 허브**

- 넷플릭스/디즈니+/티빙/웨이브/쿠팡플레이/왓챠 통합 검색
  (**TMDB watch/providers API 사용, 스크레이핑 금지**)
- 기능: 어느 OTT에 있는지 검색, 신작 알림, 통합 시청기록,
  "안 본 구독 서비스" 절약 리포트, AI 추천
- 수익: 프리미엄 3,900원/월 + OTT 어필리에이트 + 광고
- 시장: 한국 OTT 구독자 2,000만+ / 경쟁자 적음
- 예상: 월 50만 ~ 1,000만원 / 개발 3~5개월

**⑤ 광고차단 SDK / 클라우드 블록리스트 서비스**

- 문제: 현재 블록리스트 50개가 하드코딩 → 새 도메인마다 앱 업데이트 필요
- 제품: 클라우드 블록리스트 API + Electron/Tauri SDK,
  실시간 업데이트, 통계 대시보드, 커스텀 규칙
- 타겟: Electron 앱 개발자, 브라우저 제작사, 키오스크 업체
- 예상: 월 $500 ~ $5,000 / 개발 1~2개월

**⑥ 트래픽 인스펙터 / 미디어 스니퍼 개발자 도구**

- HLS/DASH 스트림 분석기, API 디버깅 트래픽 뷰어,
  QA용 광고·트래커 탐지, 웹 성능 감사
- 예상: 월 $300 ~ $1,500 / 개발 1개월

#### TIER 3 — 간접 수익화

**⑦ 기술 콘텐츠 & 교육** — 블로그/유튜브(애드센스·스폰서),
인프런·유데미 강의, 전자책(Gumroad), 컨퍼런스 발표 → 컨설팅 연결.
월 30만 ~ 500만원 + 포트폴리오/인지도 효과

**⑧ 오픈소스 후원 모델** — 합법 버전을 오픈소스로 공개 후
GitHub Sponsors / Open Collective / Ko-fi + 기업 지원 티어 +
유료 클라우드 동기화 애드온

### 비교표

| # | 아이디어 | 리스크 | 개발기간 | 예상 월수익 | 추천도 |
|---|---|---|---|---|---|
| 1 | 로컬 AI 에이전트 앱 | 없음 | 2~4개월 | 100만~1,000만원 | ★★★★★ |
| 2 | Electron 보일러플레이트 | 없음 | 3~6주 | $500~3,000 | ★★★★★ |
| 3 | 합법 미디어 센터 | 낮음 | 2~3개월 | $1,000~5,000 | ★★★★ |
| 4 | 한국 OTT 허브 | 중간 | 3~5개월 | 50만~1,000만원 | ★★★★ |
| 7 | 기술 교육 콘텐츠 | 없음 | 상시 | 30만~500만원 | ★★★★ |
| 5 | 광고차단 SDK | 낮음 | 1~2개월 | $500~5,000 | ★★★ |
| 6 | 트래픽 인스펙터 | 없음 | 1개월 | $300~1,500 | ★★★ |
| 8 | 오픈소스 후원 | 없음 | 상시 | 10만~100만원 | ★★ |
| X | Streambert 그대로 | **형사** | - | - | 금지 |

### 권장 로드맵

```
1개월     보일러플레이트 정리 + Gumroad 출시  (빠른 첫 수익 + 기술 정리)
2~3개월   로컬 AI 에이전트 앱 개발 (껍데기 재사용) → 베타 유저 확보
상시      개발 과정을 블로그/유튜브로 → 마케팅 + 부수익
6개월+    합법 미디어 센터 또는 한국 OTT 허브로 확장
```

**핵심 원칙**
1. 저작권 있는 콘텐츠는 건드리지 않고 **도구만** 판다
2. **기술(코드 패턴)을 판다** — 이 저장소의 진짜 가치
3. 합법이 오래 간다 — 불법은 한 번 걸리면 전부 소실

---

## 12. 리스크 & 주의사항

### 법적

- 무허가 스트리밍 소스(VidSrc/Videasy/Vidking/AllManga) 사용 → 한국에서도 회색지대
- README에 "교육/개인용" 디스클레이머가 길게 있으나 면책이 보장되지는 않음
- **개인 사용 ≠ 상업적 배포.** 수익화 시 신분이 "영리 침해자"로 바뀜

### 기여 정책 (원본 저장소)

`CONTRIBUTING.md` 명시 사항:

- AI는 보조 도구로 사용 가능하나 PR 제출 시 **공개(disclose) 필수**
- **AI 에이전트 계정이 authored/co-authored 한 커밋은 머지 거부**
- AI만으로 생성한 Issue/PR 금지, 제출 코드는 반드시 본인이 이해해야 함

→ **따라서 본 작업은 포크(`bmshin94/streambert`)에만 적용. 업스트림 PR 대상 아님.**

### 코드 개선 포인트

| 항목 | 내용 |
|---|---|
| `SettingsPage.jsx` 4,508줄 | 리팩토링 1순위 (섹션별 파일 분리 권장) |
| 블록리스트 하드코딩 | 광고 도메인이 랜덤 문자열 → 앱 업데이트 없이 갱신 불가 |
| 미사용 의존성 의심 | `package.json`의 `"approve": "^0.0.12"`, `"scripts": "^0.1.0"` — 사용처 불명, 타이포스쿼팅 검토 필요 |
| 테스트 없음 | 유닛/E2E 테스트 부재 |
| CI 빌드 | `build.yml`이 macOS 수동 트리거만. 전 플랫폼 자동화 여지 |

### 보안 측면에서 잘 된 점 (참고할 가치)

- `contextIsolation: true`, `nodeIntegration: false`
- preload에서 화이트리스트 API만 노출
- 자동 업데이터 `TRUSTED_UPDATE_SOURCES` — origin + pathPrefix + 리다이렉트 호스트 검증
- `validateMediaPath()` — 확장자 화이트리스트 + `realpathSync`로 심볼릭링크 우회 차단
- 웹뷰 팝업 전면 차단 (`setWindowOpenHandler` → `deny`)
- CodeQL 자동 스캔

---

## 13. 참고 링크

### 저장소

- 포크 (작업 대상): https://github.com/bmshin94/streambert
- 원본: https://github.com/truelockmc/streambert
- Codeberg 미러 (릴리스 배포): https://codeberg.org/truelockmc/streambert
- 릴리스 최신: https://codeberg.org/truelockmc/streambert/releases/latest
- AUR: https://aur.archlinux.org/packages/streambert-bin
- Trendshift: https://trendshift.io/repositories/31115

### 관련 프로젝트

- 다운로더 바이너리: https://github.com/truelockmc/vid-dl-cli-only
- 애니 스크레이핑 원류: https://github.com/pystardust/ani-cli
- ffmpeg: https://ffmpeg.org/download.html

### API / 서비스

- TMDB: https://www.themoviedb.org/ (API 설정: https://www.themoviedb.org/settings/api)
- AniList GraphQL: https://graphql.anilist.co
- Wyzie 자막: https://store.wyzie.io/redeem
- SubDL: https://subdl.com

### 문서 (이 저장소 내)

- [README.md](../README.md)
- [CONTRIBUTING.md](../CONTRIBUTING.md)
- [tmdb-tutorial.md](../tmdb-tutorial.md)
- [wyzie-tutorial.md](../wyzie-tutorial.md)
- [CLAUDE.md](../CLAUDE.md)
- [LICENSE](../LICENSE) (GPL-3.0)

### 대안 기술 스택

- Tauri: https://tauri.app
- Jellyfin: https://jellyfin.org

---

*이 문서는 저장소 전수조사(파일 84개, 코드 약 28,000줄)를 기반으로 작성되었습니다.*
*수익 예상치는 유사 제품 사례를 참고한 추정이며 보장 수치가 아닙니다.*
*법적 판단이 필요한 사안은 반드시 전문가 상담을 받으시기 바랍니다.*
