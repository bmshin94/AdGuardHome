# AdGuard Home 전수조사 & 활용/수익화 분석 보고서

> 작성: Claude Code (페르소나: 카리나 / CLAUDE.md 기준)
> 작성일: 2026-09-28
> 대상 저장소: https://github.com/bmshin94/AdGuardHome
> 원본(업스트림): https://github.com/AdguardTeam/AdGuardHome
> 분석 기준 커밋: `3ac1642` (Merge pull request #1 from bmshin94/feat/claude-guide)
> 분석 브랜치: `claude/practical-carson-nn640v`

---

## 목차

1. [저장소 정체 파악](#1-저장소-정체-파악)
2. [핵심 동작 원리 - DNS 싱크홀](#2-핵심-동작-원리---dns-싱크홀)
3. [폴더/코드 전수조사 결과](#3-폴더코드-전수조사-결과)
4. [조사 중 발견한 특이사항](#4-조사-중-발견한-특이사항)
5. [언제 쓰는가 / 어떤 도움이 되는가](#5-언제-쓰는가--어떤-도움이-되는가)
6. [설치 및 사용법](#6-설치-및-사용법)
7. [플러그인? 스킬? MCP? - 정체 분류](#7-플러그인-스킬-mcp---정체-분류)
8. [API 토큰 필요 여부](#8-api-토큰-필요-여부)
9. [GitHub에서 유명한 이유](#9-github에서-유명한-이유)
10. [로컬 에이전트 구축에 도움이 되는가](#10-로컬-에이전트-구축에-도움이-되는가)
11. [React / PHP 로 만들 수 있는가](#11-react--php-로-만들-수-있는가)
12. [수익화 아이디어 7가지 상세](#12-수익화-아이디어-7가지-상세)
13. [최종 결론 및 로드맵](#13-최종-결론-및-로드맵)

---

## 1. 저장소 정체 파악

### 결론

이 저장소는 **AdGuard Home** — 네트워크 전체를 커버하는 **광고/트래커 차단 DNS 서버**다.
`AdguardTeam/AdGuardHome`(원본)을 `bmshin94` 계정으로 fork 한 상태이며,
현재까지 추가된 **사용자 고유 커밋은 `CLAUDE.md` 페르소나 문서 1건뿐**이다.

### Git 상태 (조사 시점)

| 항목 | 값 |
|---|---|
| origin | `https://github.com/bmshin94/AdGuardHome` |
| 브랜치 | `master`, `claude/practical-carson-nn640v` |
| HEAD | `3ac1642` |
| 사용자 커밋 | `bf401e1 docs: created CLAUDE.md persona guide` |
| 나머지 커밋 | AdGuard 본사 개발자 (Ildar Kamalov, Dimitry Kolyshev, Fedor Setrakov 등) |
| 커밋 티켓 체계 | `AGH-xx`, `AGDNS-xxxx`, `ADG-xxxxx` (Jira 연동) |

### 최근 업스트림 커밋 예시

```
d6fadea AGDNS-4262 add TLS setup wizard        # TLS 설정 마법사 (client_v2 + 30개 언어)
7d1d3af AGH-19  Fix-logger-disable (24)        # dhcpd 로거 비활성화 버그 수정
aad2e15 AGH-31  Add-confs                      # configmgr 구조 확장 (dns/http/web)
c173fa7 AGH-58  add full stats pages           # 통계 페이지 신설 (SolidJS)
9373a4a AGH-32  Update dnsproxy
```

---

## 2. 핵심 동작 원리 - DNS 싱크홀

### DNS 란

도메인 이름(`naver.com`)을 IP 주소(`223.130.200.107`)로 바꿔주는 "인터넷 전화번호 안내" 역할.
브라우저/앱은 접속 전에 **반드시** DNS에 물어본다.

### AdGuard Home 이 하는 일

```
[평소]
스마트폰 → "ads.doubleclick.net IP?" → 통신사 DNS → "142.x.x.x" → 광고 로딩

[AdGuard Home 적용 후]
스마트폰 → "ads.doubleclick.net IP?" → AdGuard Home
                                        ├─ 블랙리스트 조회 → HIT!
                                        └─ "0.0.0.0" 응답
         → 광고 서버 접속 실패 → 광고 사라짐
```

광고를 **가리는(hide)** 것이 아니라 **광고 서버로 가는 길 자체를 없애는(sinkhole)** 방식.
따라서 기기에 앱/확장을 설치할 필요가 없다.

### 실제 처리 흐름 (시간순)

```
[0.001s] 폰: "youtube.com?"          → AGH
[0.002s] AGH: filtering 검사 → 정상
[0.003s] AGH: 업스트림(1.1.1.1, DoH 암호화)에 질의
[0.030s] AGH: "142.250.x.x" 응답     → querylog 에 ✅ 기록
[0.031s] 폰: 유튜브 열림

[0.100s] 폰: "doubleclick.net?"      → AGH
[0.101s] AGH: filtering 검사 → 🚨 차단 규칙 HIT
[0.101s] AGH: "0.0.0.0" 응답 (업스트림 질의 안 함) → querylog 에 🚫 기록
[0.102s] 폰: 광고 로딩 실패
```

**부수 효과**: 차단된 질의는 외부로 나가지 않으므로 체감 속도가 오히려 빨라진다.

### 원리적 한계 (README 에 명시)

| 차단 불가 | 이유 |
|---|---|
| YouTube 영상 내 광고 | 광고와 영상이 동일 도메인(`googlevideo.com`) |
| Twitch 광고 | 동일 |
| Instagram / Facebook 스폰서 게시물 | 동일 |

**"콘텐츠와 도메인을 공유하는 광고는 DNS 레벨에서 차단 불가"**
→ 실전 권장 조합: `AdGuard Home(전체 기기) + uBlock Origin(PC 브라우저)`

---

## 3. 폴더/코드 전수조사 결과

### 3-1. 백엔드 (Go) — 총 97,191줄 / 451개 `.go` 파일

`internal/` 아래 30개 패키지. 크기순:

| 패키지 | 줄 수 | 파일 | 역할 (패키지 doc 주석 기준) |
|---|---|---|---|
| `filtering/` | 14,653 | 51 | DNS 요청/응답 필터 엔진. 차단규칙·세이프브라우징·자녀보호·세이프서치 |
| `home/` | 12,952 | 38 | AdGuard Home HTTP API 메서드 + 웹 관리 서버 (앱의 심장) |
| `dnsforward/` | 12,328 | 35 | DNS 포워딩 서버. 업스트림 중계 |
| `dhcpsvc/` | 9,359 | 32 | 신규 DHCP 서비스 |
| `dhcpd/` | 8,638 | 33 | 구 DHCP 서버 (go.mod 에 제거 예정 TODO 존재) |
| `querylog/` | 5,449 | 20 | 쿼리 로그 기록/검색 |
| `client/` | 4,704 | 12 | DNS 클라이언트(기기)별 개별 설정 |
| `configmigrate/` | 4,275 | 41 | YAML 설정 파일 버전 자동 업그레이드 |
| `aghnet/` | 3,753 | 33 | 네트워킹 유틸리티 |
| `next/` | 3,204 | 26 | **차세대 아키텍처 실험** (`go:build next`) |
| `ossvc/` | 3,003 | 24 | 플랫폼 독립 서비스 등록 (systemd/launchd/Windows) |
| `stats/` | 2,478 | 7 | 통계 집계 |
| `aghtls/` | 1,850 | 10 | TLS 인증서 관리 |
| `updater/` | 1,310 | 5 | 자체 자동 업데이트 |
| `aghos/` | 1,258 | 20 | OS 추상화 |
| `arpdb/` | 1,101 | 10 | Network Neighborhood(ARP) DB |
| `aghuser/` | 1,011 | 8 | 웹 관리자 계정/인증 |
| `schedule/` | 871 | 2 | 시간표 기반 스케줄링 |
| `ipset/` | 807 | 4 | Linux ipset 연동 |
| `permcheck/` | 641 | 7 | 파일 권한 점검 |
| `aghtest/` | 600 | 4 | 테스트 헬퍼/더블 |
| `whois/` | 562 | 2 | WHOIS 조회 |
| `configmgr/` | 544 | 6 | 온디스크 설정 엔티티 |
| `aghalg/` | 432 | 5 | 알고리즘/자료구조 |
| `aghhttp/` | 430 | 5 | HTTP 헬퍼 |
| `rdns/` | 286 | 2 | 역방향 DNS 조회 |
| `aghrenameio/` | 276 | 4 | 원자적 파일 교체 |
| `version/` | 197 | 3 | 버전 정보 |
| `agh/` | 167 | 1 | 핵심 인터페이스 정의 |
| `aghslog/` | 52 | 1 | 구조화 로깅(slog) |

### 3-2. 엔트리포인트 — 빌드 태그로 2개 분리

```go
// main.go       (//go:build !next)  → internal/home.Main()      : 현행 아키텍처
// main_next.go  (//go:build  next)  → internal/next/cmd.Main()  : 차세대 아키텍처
```

두 파일 모두 `//go:embed build` 로 **빌드된 프론트엔드를 바이너리에 내장**한다.
→ 결과물이 단일 실행파일이 되는 이유.

### 3-3. 프론트엔드 — 2개 세대 공존

| 디렉터리 | 버전 | 스택 |
|---|---|---|
| `client/` | 0.1.0 | React 16 + Redux + redux-thunk + webpack + recharts + react-i18next + react-table |
| `client_v2/` | 3.0.0 | **SolidJS** + Ark UI + @modular-forms/solid + chart.js + orval + PostCSS |

`Makefile` 기본값이 이미 신버전을 가리킨다:

```makefile
CLIENT_DIR = client_v2
```

`client_v2/` 특징:
- `AGENTS.md` — AdGuard 팀이 작성한 **AI 코딩 에이전트용 프론트엔드 가이드**
- `orval.config.ts` — `openapi.yaml` → TypeScript API 클라이언트 자동 생성
- `src/__locales/*.json` — 30개 언어 (`ko.json` 한국어 포함)
- `scripts/check-translations.js`, `generate-locales.js`, `check-locales.js`

공통 개발 스크립트: `dev`, `build-prod`, `lint`, `typecheck`, `test`(vitest), `test:e2e`(Playwright)

### 3-4. 기타 디렉터리

| 경로 | 내용 |
|---|---|
| `openapi/openapi.yaml` | **REST API 명세 3,389줄 / 엔드포인트 81개** |
| `openapi/next.yaml` | 차세대 API 명세 |
| `scripts/install.sh` | 원클릭 설치 스크립트 |
| `scripts/make/*.sh` | 빌드·린트·테스트·릴리스 스크립트 15개 |
| `scripts/hooks/` | git `pre-commit`, `pre-merge-commit` 훅 |
| `scripts/translations/` | 번역 업/다운로드 (Go) |
| `scripts/blocked-services/` | 차단 서비스 목록 생성기 (Go) |
| `scripts/vetted-filters/` | 검증된 필터 목록 생성기 (Go) |
| `docker/` | Dockerfile 4종 (build / ci / frontend / snapcraft) |
| `snap/` | Linux Snap 패키징 |
| `bamboo-specs/` | Atlassian Bamboo 사내 CI (bamboo / release / snapcraft) |
| `.github/workflows/` | GitHub Actions 4개 (build / mirror / workflow / private) |
| `AGHTechDoc.md` | **기술 문서 45KB** — 내부 알고리즘/API 상세 |
| `CHANGELOG.md` | **150KB** — 누적 변경 이력 |
| `HACKING.md` | 코딩 컨벤션 |
| `.twosky.json` | AdGuard 자체 번역 플랫폼 설정 |
| `LICENSE.txt` | **GPL-3.0** (35KB) |

### 3-5. 빌드 환경

```
Go       1.26.8  (Makefile GOTOOLCHAIN 으로 고정)
Node.js  24.10.0+
npm      10.8+
```

```bash
make init          # git hooks 설정 (core.hooksPath → scripts/hooks)
make               # = deps + quick-build
make build-docker  # 로컬 도커 이미지
make build-release CHANNEL='...' VERSION='...'
env GOOS='linux' GOARCH='arm64' make   # 크로스 컴파일
```

**주의**: README 에 경고 — 병렬 빌드(`make -j 4`) 미지원. `MAKEFLAGS` 에 `-j` 가 있으면 `make -j 1` 로 덮어써야 한다.

### 3-6. 주요 의존성

```
AdguardTeam/dnsproxy  v0.84.2   ← DNS 프록시 코어 (AdGuard DNS 와 공유)
AdguardTeam/urlfilter v0.23.4   ← 필터 규칙 엔진
AdguardTeam/golibs    v0.35.15  ← 공통 라이브러리 (service, slogutil 등)
AdguardTeam/dnscrypt  v0.0.2
miekg/dns             v1.1.72   ← DNS 프로토콜
quic-go/quic-go       v0.61.0   ← DoQ (DNS-over-QUIC)
go.etcd.io/bbolt      v1.5.0    ← 임베디드 KV 저장소
kardianos/service     v1.2.4    ← OS 서비스 등록
insomniacslk/dhcp, gopacket, mdlayher/* ← DHCP/저수준 패킷
```

린트 툴체인(`tool` 블록): `gocyclo`, `gocognit`, `gosec`, `govulncheck`, `staticcheck`, `errcheck`, `ineffassign`, `unparam`, `misspell`, `gofumpt`, `shfmt`, `fieldalignment`, `nilness`, `shadow`, `yamlfmt`

---

## 4. 조사 중 발견한 특이사항

### 4-1. 차단 가능 서비스 151개 — 그 안에 Claude 도 있다

`internal/filtering/servicelist.go` 에 서비스 정의 151개가 하드코딩되어 있고,
각 항목이 `ID` / `Name` / `IconSVG`(로고 SVG 바이너리) / `Rules` / `GroupID` 를 가진다.

```go
ID:      "claude",
Name:    "Claude",
IconSVG: []byte("<svg ...>"),   // Claude 로고 SVG 내장
Rules:   []string{ "||anthropic.com^", ... }
```

즉 관리 UI 의 Blocked services 에서 **Claude 를 원클릭 차단**할 수 있다. (켜면 Claude 사용 불가)

### 4-2. go.mod 의 AI SDK 들은 실제로 쓰이지 않는다

`go.mod` indirect 블록에 다음이 존재한다.

```
github.com/anthropics/anthropic-sdk-go v1.63.1  // indirect
github.com/openai/openai-go/v3        v3.52.0  // indirect
google.golang.org/genai               v1.68.0  // indirect
```

그러나 저장소 전체 `.go` 파일을 검색한 결과 **이들을 import 하는 코드는 0줄**이다.
(유일한 `anthropic` 문자열 등장은 위 4-1 의 차단 규칙 `||anthropic.com^`)

→ 툴체인/전이 의존성으로 딸려온 것. **"AdGuard Home 에 AI 기능이 들어갔다"는 해석은 오류.**

### 4-3. 점진적 리아키텍처 패턴

백엔드와 프론트엔드가 **동일한 전략**을 쓴다.

| 영역 | 현행 | 차세대 | 분리 수단 |
|---|---|---|---|
| 백엔드 | `internal/home/` | `internal/next/` | `//go:build next` 태그 |
| 프론트 | `client/` (React) | `client_v2/` (SolidJS) | `Makefile CLIENT_DIR` |
| API | `openapi/openapi.yaml` | `openapi/next.yaml` | `NEXTAPI=0/1` |

"전부 멈추고 새로 짜기"가 아니라 **병행 운영 → 점진적 전환**. 대형 코드베이스 리팩토링의 모범 사례.

### 4-4. AI 개발을 전제한 저장소 구조

```
/CLAUDE.md            ← 사용자가 추가 (카리나 페르소나 지침)
client_v2/AGENTS.md   ← AdGuard 본사가 추가 (AI 에이전트용 프론트엔드 규칙)
```

두 파일 모두 **AI 도구용 문서**이며 애플리케이션 런타임과 무관하다.
특히 `AGENTS.md` 는 최근 두 커밋(`c173fa7`, `d6fadea`)에서 계속 갱신되고 있어,
AdGuard 팀이 실제로 AI 에이전트를 개발 워크플로에 편입했음을 시사한다.

---

## 5. 언제 쓰는가 / 어떤 도움이 되는가

### 5-1. 사용 시나리오

| 상황 | AdGuard Home 의 역할 |
|---|---|
| 스마트TV 광고 | 앱 설치가 불가능한 기기도 DNS 변경만으로 차단 |
| 앱 내부 광고 | 브라우저 확장으로 못 막는 네이티브 앱 광고 차단 |
| 집 전체 기기 | 공유기 DNS 1회 설정 → 폰/PC/TV/IoT 전체 적용 |
| 자녀 보호 | 성인 사이트 차단 + 구글/유튜브/빙 세이프서치 강제 |
| DNS 도청 방지 | DoH / DoT / DoQ / DNSCrypt 업스트림 암호화 |
| 기기 감시 | 쿼리 로그로 앱의 외부 통신 전수 확인 |
| 공유기 DHCP 대체 | 내장 DHCP 서버로 IP 할당 + 호스트명 관리 |
| 기기별 차등 정책 | 아이 태블릿은 엄격, 본인 PC 는 느슨하게 |
| 시간대별 정책 | `schedule` 패키지로 "밤 10시~아침 7시 SNS 차단" |

### 5-2. 실용적 이득

- 월 0원으로 집 전체 광고 차단 (라즈베리파이 / 미니PC / NAS 도커)
- 네트워크 가시성 확보 (최다 접속 도메인, 기기별 트래픽)
- 차단된 질의는 외부로 안 나가므로 체감 속도 개선
- 가족 기기 광고 제거 → 체감 만족도 높음

### 5-3. 개발자 관점 이득

| 배울 것 | 위치 |
|---|---|
| 인터페이스 기반 모듈 설계 | `internal/agh/` (167줄, 핵심 인터페이스만) |
| 설정 마이그레이션 아키텍처 | `internal/configmigrate/` (41개 파일, 버전별 분리) |
| 서비스 라이프사이클 관리 | `internal/next/` + `golibs/service` |
| 테스트 더블/헬퍼 패턴 | `internal/aghtest/` |
| 구조화 로깅(slog) 도입 | `internal/aghslog/`, `internal/home/log.go` |
| 크로스플랫폼 OS 추상화 | `internal/aghos/`, `internal/ossvc/` |
| 자기 자신 업데이트 구현 | `internal/updater/` |
| 파일 권한 보안 점검 | `internal/permcheck/` |
| 점진적 리아키텍처 | `go:build next` 전략 |
| SolidJS 대규모 실전 사례 | `client_v2/` |
| OpenAPI → TS 클라이언트 자동생성 | `client_v2/orval.config.ts` |

---

## 6. 설치 및 사용법

### 6-1. 방법 1 — 원클릭 스크립트 (Linux / macOS / FreeBSD / OpenBSD)

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
# 또는
wget --no-verbose -O - https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

| 옵션 | 의미 |
|---|---|
| `-c <channel>` | 채널 지정 |
| `-r` | 재설치 |
| `-u` | 삭제 |
| `-v` | 상세 로그 |

`-r` 과 `-u` 는 상호 배타적.

### 6-2. 방법 2 — Docker (권장)

```bash
docker run -d --name adguardhome \
  --restart unless-stopped \
  -v /my/own/workdir:/opt/adguardhome/work \
  -v /my/own/confdir:/opt/adguardhome/conf \
  -p 53:53/tcp -p 53:53/udp \
  -p 3000:3000/tcp \
  -p 80:80/tcp -p 443:443/tcp -p 443:443/udp \
  -p 853:853/tcp \
  adguard/adguardhome:latest
```

| 포트 | 용도 |
|---|---|
| 53 TCP/UDP | DNS 본체 (필수) |
| 3000 | 초기 설치 마법사 UI |
| 80 / 443 | 관리 UI + DoH 서버 |
| 853 | DoT 서버 |
| 784 / 8853 | DoQ 서버 |
| 67 / 68 | DHCP (host 네트워크 필요) |

### 6-3. 방법 3 — 소스 빌드

```bash
cd /home/user/AdGuardHome
make init     # git hooks 설정
make          # 프론트 + 백엔드 전체 빌드
```

요구사항: Go 1.26.8 / Node.js 24.10.0+ / npm 10.8+
크로스 컴파일: `env GOOS='linux' GOARCH='arm64' make`
병렬 빌드(`-j`) 미지원 주의.

### 6-4. 설치 후 초기 설정 흐름

```
1. http://<서버IP>:3000 접속
   → /install/get_addresses → /install/check_config → /install/configure (3단계 마법사)
2. 관리자 계정(ID/PW) 생성
3. 웹 포트(80) / DNS 포트(53) 확정
4. [핵심] 공유기 DHCP 설정의 "기본 DNS 서버"를 AdGuard Home IP 로 변경
   (또는 기기별 Wi-Fi 설정에서 DNS 직접 지정)
5. 대시보드에서 차단 통계 확인
```

설정 파일: `AdGuardHome.yaml` (작업 디렉터리).
버전 업그레이드 시 `internal/configmigrate/` 가 자동 마이그레이션.

### 6-5. 기타 배포 채널

- Snap Store: `snapcraft.io/adguard-home`
- Docker Hub: `hub.docker.com/r/adguard/adguardhome`
- GitHub Releases

---

## 7. 플러그인? 스킬? MCP? - 정체 분류

### 결론: 셋 다 아니다. **독립 실행형 서버 애플리케이션**이다.

| 분류 | 해당 여부 | 근거 |
|---|---|---|
| 플러그인 | ❌ | `main.go` 를 가진 단독 실행 바이너리. 호스트 앱이 없다 |
| Claude 스킬 | ❌ | `SKILL.md` / `.claude/skills/` 없음 |
| MCP 서버 | ❌ | MCP 프로토콜/JSON-RPC 핸들러 구현 0건 |
| **독립 서버 앱(데몬)** | ✅ | `ossvc` 가 systemd / launchd / Windows 서비스로 등록 |

정확한 분류: **네트워크 인프라 소프트웨어 / 셀프호스팅 서비스**
(Nginx, Redis, PostgreSQL 과 같은 계열. VSCode 확장 / Figma 플러그인 계열이 아니다.)

### 혼동의 원인 — 두 개의 AI 문서

```
/CLAUDE.md            → 사용자가 추가한 Claude Code 작업 지침 (카리나 페르소나)
client_v2/AGENTS.md   → AdGuard 본사의 AI 에이전트용 프론트엔드 규칙
```

둘 다 **AI 도구가 읽는 문서**이며, 앱 실행 시에는 전혀 로드되지 않는다.

### 단, "확장 지점"은 풍부하다

```
AdGuard Home (본체)
  └─ REST API 81개
       ├─ Home Assistant 공식 통합 (존재)
       ├─ PyPI: adguardhome (Python 클라이언트)
       └─ [기회] 직접 MCP 서버를 만들어 감쌀 수 있다
```

---

## 8. API 토큰 필요 여부

### 결론: **토큰 불필요. 계정 등록도, 외부 서비스 키도 없다.**

`openapi/openapi.yaml` 3386행 — 보안 스킴이 단 하나다.

```yaml
'securitySchemes':
  'basicAuth':
    'type': 'http'
    'scheme': 'basic'
```

| 방식 | 지원 | 비고 |
|---|---|---|
| 세션 쿠키 | ✅ | `POST /login` → 쿠키. 웹 UI 가 사용 |
| HTTP Basic Auth | ✅ | `curl -u admin:pw`. 자동화 권장 |
| API 토큰 | ❌ | 개념 자체가 없음 |
| OAuth / JWT | ❌ | 없음 |
| 외부 서비스 키 | ❌ | AdGuard 본사 로그인 불필요 |

### 사용 예시

```bash
# 로그인 (쿠키)
curl -X POST http://192.168.1.10/control/login \
  -H 'Content-Type: application/json' \
  -d '{"name":"admin","password":"mypassword"}'

# Basic Auth (자동화 권장)
curl -u admin:mypassword http://192.168.1.10/control/status
curl -u admin:mypassword http://192.168.1.10/control/stats
curl -u admin:mypassword http://192.168.1.10/control/querylog

# 보호 기능 5분간 일시정지
curl -u admin:mypassword -X POST \
  http://192.168.1.10/control/protection \
  -H 'Content-Type: application/json' \
  -d '{"enabled":false,"duration":300000}'
```

### 장점

```
클라우드 의존 0%  → 인터넷 단절 시에도 로컬 동작
사용량 과금 0원   → API 호출 무제한
데이터 유출 0     → DNS 기록이 외부로 나가지 않음
회원가입 0        → 이메일 수집 없음
```

### 보안 주의

Basic Auth 는 HTTP 상에서 자격증명이 평문 전송된다.
외부 노출 시 반드시 HTTPS(`internal/aghtls/`) 또는 VPN(WireGuard/Tailscale) 내부에서만 사용.
최근 커밋 `AGDNS-4262 add TLS setup wizard` 가 이 TLS 설정을 UI 마법사로 단순화했다.

### 엔드포인트 81개 카테고리

| 카테고리 | 엔드포인트 |
|---|---|
| 상태/통계 | `/status` `/stats` `/stats_reset` `/stats_info` `/stats/config` `/stats/config/update` |
| 쿼리로그 | `/querylog` `/querylog_clear` `/querylog_info` `/querylog/config` `/querylog/config/update` |
| DNS | `/dns_info` `/dns_config` `/test_upstream_dns` `/cache_clear` `/protection` |
| 필터링 | `/filtering/status` `/filtering/config` `/filtering/add_url` `/filtering/remove_url` `/filtering/set_url` `/filtering/refresh` `/filtering/set_rules` `/filtering/check_host` |
| 보호기능 | `/safebrowsing/{enable,disable,status}` `/parental/{enable,disable,status}` `/safesearch/{enable,disable,settings,status}` |
| 클라이언트 | `/clients` `/clients/add` `/clients/delete` `/clients/update` `/clients/find` `/clients/search` |
| DHCP | `/dhcp/status` `/dhcp/interfaces` `/dhcp/set_config` `/dhcp/find_active_dhcp` `/dhcp/{add,remove,update}_static_lease` `/dhcp/reset` `/dhcp/reset_leases` |
| TLS | `/tls/status` `/tls/configure` `/tls/validate` |
| 접근제어 | `/access/list` `/access/set` |
| 서비스차단 | `/blocked_services/services` `/blocked_services/all` |
| 업데이트 | `/version.json` `/update` |
| 인증 | `/login` `/logout` `/profile` `/profile/update` |
| 설치 | `/install/get_addresses` `/install/check_config` `/install/configure` |

---

## 9. GitHub에서 유명한 이유

### 9-1. 타이밍 — 셀프호스팅 + 프라이버시 트렌드의 중심

GDPR, 추적 광고 논란 이후 "내 데이터를 내가 통제" 수요 급증.
r/selfhosted, r/homelab 커뮤니티 성장의 대표 앱.

### 9-2. Pi-hole 이라는 명확한 라이벌 구도

README 가 비교표를 정면에 배치한다.

| 기능 | AdGuard Home | Pi-Hole |
|---|---|---|
| 광고/추적 차단 | O | O |
| 차단목록 커스터마이징 | O | O |
| 내장 DHCP 서버 | O | O |
| 관리 UI HTTPS | O | lighttpd 수동 설정 필요 |
| 암호화 DNS 업스트림(DoH/DoT/DNSCrypt) | O | X (추가 SW 필요) |
| 크로스플랫폼 | O | X (Docker 로만) |
| DoH/DoT **서버** 역할 | O | X (추가 SW 필요) |
| 피싱/멀웨어 도메인 차단 | O | X (비기본 목록 필요) |
| 자녀보호(성인 도메인) | O | X (비기본 목록 필요) |
| 세이프서치 강제 | O | X |
| 기기별 설정 | O | O |
| 접근 제어 | O | X |
| root 권한 없이 실행 | O | X |

구조적 차이: Pi-hole 은 `dnsmasq + lighttpd + PHP + FTL` 조립형,
AdGuard Home 은 **Go 단일 바이너리 일체형**.

### 9-3. 단일 바이너리의 압도적 편의성

```bash
./AdGuardHome -s install   # 설치 완료
```

Go 정적 링킹 → 런타임 의존성 0.
지원 범위: Linux / macOS / Windows / FreeBSD / OpenBSD × amd64 / arm64 / armv7 / armv6 / 386 / mips / mipsle / ppc64le ...
→ 라즈베리파이 Zero, 시놀로지 NAS, OpenWrt 공유기까지 커버.

### 9-4. 기업이 유지보수하는 오픈소스라는 신뢰

- `CHANGELOG.md` 150KB — 수년간 꾸준한 릴리스
- `bamboo-specs/` — Atlassian Bamboo 사내 CI
- `.twosky.json` — 자체 번역 플랫폼, 30개 언어
- Jira 티켓 체계 (`AGH-`, `AGDNS-`, `ADG-`)
- 린터 체인(gosec, govulncheck, staticcheck 등) + `HACKING.md` 컨벤션
- 상용 제품 AdGuard DNS 와 코드 공유 ("both share a lot of code" — README)

### 9-5. Docker Hub 다운로드 수 뱃지

README 상단 `docker pulls` 뱃지가 대규모 실사용을 증명 (사회적 증거).

### 9-6. 생태계 유입

```
Home Assistant 공식 통합   → 스마트홈 커뮤니티 전체 유입
PyPI: adguardhome          → Python 생태계
Snap Store                 → 우분투 사용자
"Projects that use AdGuard Home" 섹션 → 파생 프로젝트
```

### 9-7. UI 품질 + README 마케팅

README 상단에 동작 GIF 데모 배치. Pi-hole 대비 현대적 UI.
현재 SolidJS 로 v2 개편 중이라 추가 개선 예정.

### 9-8. 개발자 학습 수요

잘 구조화된 대형 Go 실전 코드베이스(97,000줄)는 학습 자원으로 가치가 높다.
특히 `internal/next/` 의 점진적 리아키텍처 사례는 희귀하다.

---

## 10. 로컬 에이전트 구축에 도움이 되는가

### 결론: 매우 도움이 된다. 단 "뇌"가 아니라 "손발과 뼈대" 영역에서.

### 레벨 1 — 에이전트가 조작할 대상으로 최적

로컬 에이전트 개발의 최대 난제는 "에이전트가 실제로 무엇을 조작할 것인가"다.
AdGuard Home 은 그 조건을 거의 완벽히 충족한다.

| 요구 조건 | AdGuard Home |
|---|---|
| 명확한 REST API | 81개 엔드포인트 |
| 기계 판독 가능 스펙 | `openapi.yaml` 3,389줄 |
| 단순 인증 | Basic Auth 1줄 |
| 로컬 실행 | 완전 오프라인 |
| 조회 + 변경 모두 | GET / POST 양쪽 제공 |
| 실패 복구 용이 | 설정 롤백 쉬움 |

설계 가능한 MCP 툴 세트:

```
get_dns_stats       DNS 차단 통계 조회
search_querylog     쿼리 로그 검색/필터
block_domain        차단 규칙 추가
unblock_domain      예외(@@) 규칙 추가
pause_protection    N분간 보호 일시정지
list_clients        네트워크 기기 목록
set_client_policy   기기별 정책 변경
toggle_service      151개 서비스 차단 on/off
analyze_anomaly     이상 트래픽 분석 (자체 로직)
generate_report     기간별 리포트 생성 (자체 로직)
```

### 레벨 2 — 에이전트의 "감각 기관"

`internal/querylog/` 가 생산하는 데이터는 로컬 환경의 실시간 행동 로그다.

```
쿼리로그 필드: 기기(IP/MAC/이름) × 시각 × 도메인 × 처리결과(허용/차단/규칙) × 응답시간
```

활용 예:
- 이상 탐지: 신규 기기가 특정 도메인에 비정상 빈도로 질의
- 자동 리포트: 주간 차단 건수, 신규 추적기 발견
- 자동 튜닝: 반복 차단 → 반복 허용 패턴 → 화이트리스트 제안
- 육아 보조: 특정 기기의 심야 접속 시도 알림

로컬 에이전트의 실질적 가치는 "내 환경의 실제 데이터"를 가질 때 발생한다.

### 레벨 3 — Go 에이전트 아키텍처 교과서

| 참고 대상 | 위치 |
|---|---|
| 인터페이스 기반 플러그인 구조 | `internal/agh/` |
| 설정 관리 + 자동 마이그레이션 | `internal/configmgr/`, `internal/configmigrate/` |
| 서비스 라이프사이클(start/stop/reload) | `internal/next/` + `golibs/service` |
| 테스트 더블 패턴 | `internal/aghtest/` |
| 구조화 로깅 | `internal/aghslog/`, `internal/home/log.go` |
| 크로스플랫폼 OS 추상화 | `internal/aghos/`, `internal/ossvc/` |
| **자기 자신 업데이트** | `internal/updater/` (에이전트 자동 업데이트에 직접 전용 가능) |
| 파일 권한 검사 | `internal/permcheck/` |
| 점진적 리아키텍처 | `go:build next` 태그 분리 |

### 레벨 4 — 에이전트 실행 환경의 네트워크 게이트

역방향 활용. AdGuard Home 을 에이전트 전용 방화벽으로 사용.

```
[로컬 AI 에이전트] → DNS 질의 → [AdGuard Home] → 인터넷
                                   ├─ allowlist 도메인만 통과 (/access/set)
                                   ├─ 그 외 전부 차단
                                   └─ 모든 외부 통신 querylog 에 기록
```

자율 에이전트의 외부 통신을 **감시·통제**할 수 있다. 코드 실행 샌드박스의 네트워크 계층 보완.

### 도움이 되지 않는 영역 (명확히)

```
X LLM 호출 로직        (go.mod 의 AI SDK 는 실제 미사용, 코드 0줄)
X 프롬프트 엔지니어링
X 벡터DB / RAG
X 에이전트 루프 / 툴 콜링
X MCP 프로토콜 구현
```

---

## 11. React / PHP 로 만들 수 있는가

### 11-1. React — 프론트엔드는 이미 React 로 구현되어 있다

```
client/  →  React 16 + Redux + redux-thunk + webpack + recharts + react-i18next
```

```bash
cd client
npm install
npm run watch:hot   # webpack-dev-server 핫리로드
npm run lint
npm run test        # vitest
npm run test:e2e    # playwright
```

| 작업 | 난이도 |
|---|---|
| 기존 `client/` 대시보드 커스터마이징 | 낮음 |
| 별도 React 앱으로 새 대시보드 (API 호출만) | 낮음 |
| 다중 인스턴스 통합 관리 패널 | 중간 |
| React Native 모바일 앱 | 중간 |
| 리포트/분석 SaaS 프론트엔드 | 중간 |

`client_v2/orval.config.ts` 방식을 활용하면 `openapi.yaml` 에서 타입 포함 TS 클라이언트를 자동 생성할 수 있다.

```bash
npx orval --input ../openapi/openapi.yaml --output ./src/api
```

### 11-2. PHP — 관리 패널은 가능, DNS 서버 코어는 비현실적

#### 가능한 영역

```php
$ch = curl_init('http://192.168.1.10/control/stats');
curl_setopt($ch, CURLOPT_USERPWD, 'admin:password');
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$stats = json_decode(curl_exec($ch), true);
```

| 작업 | 적합성 |
|---|---|
| Laravel 멀티 인스턴스 관리 SaaS | 적합 |
| 고객사별 리포트 생성/메일 발송 | 적합 |
| 결제/구독 (Cashier) | 적합 |
| 필터 목록 배포 서버 | 적합 |
| WordPress 플러그인형 관리도구 | 가능 |

#### 비현실적인 영역 — DNS 서버 본체

| 요구사항 | PHP 현실 |
|---|---|
| 초당 수천~수만 UDP 패킷 처리 | PHP-FPM 은 요청/응답 모델. 상시 UDP 루프 부적합 |
| 상주 프로세스 + 상태 유지 | 요청마다 상태 초기화가 기본 철학 |
| 대규모 동시성 | goroutine 대응물 없음 (Swoole/RoadRunner 필요) |
| DoH / DoT / DoQ 서버 | QUIC 생태계 사실상 부재 |
| 메모리 내 대용량 필터 트라이 | 요청마다 규칙 로딩은 비현실적 |
| 단일 바이너리 배포 | PHP 런타임 필요 |
| 1ms 이하 지연 | 인터프리터 오버헤드 |

### 11-3. 권장 아키텍처

```
+-------------------------------------------------+
|  AdGuard Home (Go) — 수정 없이 그대로 사용       |
|  DNS/DHCP 코어. REST API 81개로 노출             |
+-----------------------+-------------------------+
                        | HTTP + Basic Auth
        +---------------+----------------+
        v                                v
+---------------------+     +---------------------------+
| React / Next.js     |     | PHP / Laravel             |
| - 관리 대시보드      |     | - 멀티테넌트 백엔드        |
| - 실시간 통계/차트   |     | - 인증/권한(RBAC)          |
| - 모바일 UI         |     | - 결제/구독               |
| - 리포트 뷰         |     | - 스케줄러/이메일          |
+---------------------+     +---------------------------+
```

이유:
1. 강점 분업 — Go(네트워크) / React(UI) / PHP(비즈니스 로직)
2. 개발 속도 — DNS 서버 재작성 수년 vs API 연동 수주
3. 버그 리스크 최소화 — DNS/DHCP 예외 처리는 이미 검증된 97,000줄
4. 라이선스 안전 — 별도 프로세스 + HTTP API 호출은 GPL 파생저작물로 보지 않는 것이 일반적

---

## 12. 수익화 아이디어 7가지 상세

### 12-0. 전제 — 라이선스(GPL-3.0)

`LICENSE.txt` = **GPL-3.0**.

| 행위 | GPL-3.0 |
|---|---|
| 설치해서 사용 | 자유 |
| 수정해서 **내부에서만** 사용 | 자유 (공개 의무 없음) |
| 수정해서 **배포/판매** | 수정 소스 전체 공개 + GPL-3.0 유지 |
| 브랜딩 변경 후 판매 | 소스 공개 필요 + AdGuard 상표 사용 불가 |
| SaaS 로 호스팅해 과금 | GPL-3.0 은 AGPL 이 아니므로 네트워크 사용을 배포로 보지 않음 → 가능 |
| **별도 앱에서 REST API 만 호출** | 파생저작물 아님 → 자체 코드 비공개 가능 |

핵심 전략:

```
[비권장] AdGuard Home 코드를 수정해 "우리 제품"으로 판매
         → 소스 공개 강제 + 상표 문제 + 업스트림 추적 부담

[권장]   AdGuard Home 은 원본 그대로 사용하고,
         그 위에 얹는 관리/분석/자동화 레이어로 수익화
         → 자체 코드 비공개 가능 + 업데이트 자동 반영 + 법적 명확성
```

"AdGuard Home 을 팔지 말고, AdGuard Home 위의 가치를 팔아라."

상표: "AdGuard" 는 등록상표. 제품명에 사용 불가. "AdGuard Home 기반" 정도의 사실 서술은 허용 범위.
(실제 사업화 시 법률 검토 필수.)

---

### 아이디어 1 — B2B 멀티테넌트 관리 SaaS  ★★★★★

#### 문제

AdGuard Home 은 "인스턴스 1개 = 관리 화면 1개" 구조.
IT 관리업체(MSP)가 고객사 30곳에 배포하면 브라우저 탭 30개, 자격증명 30개, 정책 변경 30회 반복이 발생한다.

#### 제품 구조

```
+--------------------------------------------------+
| 통합 관제 대시보드 (React / Next.js)              |
| - 전체 고객사 단일 화면                            |
| - 정책 템플릿 일괄 배포                            |
| - 이상 징후 알림 (Slack / 카카오 / 이메일)          |
| - 월간 리포트 PDF 자동 생성                        |
+------------------------+-------------------------+
                         |
+------------------------v-------------------------+
| 백엔드 (Laravel / NestJS / Go)                    |
| - 테넌트 / RBAC                                   |
| - 인스턴스 레지스트리 + 헬스체크                    |
| - 폴링 워커 → 시계열 DB (TimescaleDB)              |
| - 결제/구독 (Stripe / 토스페이먼츠)                 |
+------------------------+-------------------------+
                         | Basic Auth over WireGuard/Tailscale
     +--------+----------+----------+--------+
     v        v          v          v        v
  고객사A   고객사B    고객사C    고객사D   고객사E
  [AGH]    [AGH]     [AGH]      [AGH]    [AGH]
```

#### 기능 티어링

| 기능 | Free | Pro ₩29,000/월 | Business ₩99,000/월 |
|---|---|---|---|
| 인스턴스 수 | 3 | 25 | 무제한 |
| 통합 대시보드 | O | O | O |
| 데이터 보존 | 7일 | 90일 | 2년 |
| 정책 템플릿 일괄 배포 | X | O | O |
| 알림 (Slack/카톡/메일) | X | O | O |
| PDF 리포트 (화이트라벨) | X | X | O |
| SSO / SAML | X | X | O |
| 감사 로그 | X | X | O |
| API 액세스 | X | 제한 | 무제한 |
| 지원 | X | 이메일 | 전화 + SLA |

#### 최대 기술 과제 — NAT 뒤 인스턴스 접속

| 방법 | 장점 | 단점 |
|---|---|---|
| Tailscale / Headscale | 설정 간단 | 외부 의존 |
| WireGuard 직접 | 통제력 | 설정 복잡 |
| **경량 에이전트 푸시 (권장)** | NAT 무관, 인바운드 포트 불필요 | 에이전트 개발 필요 |
| Cloudflare Tunnel | 무료 티어 | 벤더 종속 |

권장: 고객사에 소형 Go 에이전트 배포 → 로컬 AGH API 를 읽어 중앙 서버로 push.
인바운드 포트 개방이 불필요해 영업 장벽이 크게 낮아진다. 에이전트는 자체 코드이므로 GPL 무관.

#### 수익 시뮬레이션

```
타깃: 국내 중소 IT 관리업체 / 학원 체인 / 프랜차이즈 / 병원 네트워크

1년차(보수적)
  Pro   30 × ₩29,000 = ₩  870,000 /월
  Biz    5 × ₩99,000 = ₩  495,000 /월
  MRR              ≈ ₩1,365,000 /월   (ARR ≈ ₩1,638만)

2~3년차(성장)
  Pro  200 + Biz 40  → MRR ≈ ₩1,156만  (ARR ≈ ₩1.4억)
```

#### 평가

```
장점: 고통 명확 / 법인 고객(지불의사 높음) / GPL 안전 / 이탈률 낮음 / React+Laravel 스택으로 충분
리스크: AdGuard 사가 직접 구현 가능 (AdGuard DNS 비즈니스 플랜 존재)
        → 대응: 클라우드 사용이 불가한 "온프레미스 셀프호스팅" 니치에 집중
        초기 고객 확보 난이도 → 아이디어 5(컨설팅)로 선행 진입
```

---

### 아이디어 2 — 한국 시장 특화 필터 목록 구독  ★★★★★

#### 기회

글로벌 필터(EasyList 등)는 한국 도메인 커버리지가 낮다.

```
미흡: 국내 광고 네트워크 / 국내 성인·도박 사이트 / 통신사·제조사 앱 텔레메트리
     / 국내 쇼핑몰 리타게팅 픽셀
필요: 국내 은행·공공 사이트 화이트리스트 (오탐 시 인터넷뱅킹 장애 → 치명적)
```

#### 제품 구조

```
Free  (공개, GitHub + CDN)
  - 기본 한국 광고 차단 목록
  - 목적: 인지도 확보 = 마케팅 채널

Pro   (₩4,900/월 또는 ₩39,000/년)
  - 도박/불법 사이트 집중 차단 (수동 큐레이션, 일 단위 갱신)
  - 한국 성인 사이트 차단 (자녀보호)
  - 금융/공공 화이트리스트 (오탐 방지 — 핵심 차별 기능)
  - 국내 앱 텔레메트리 차단
  - 시간 단위 갱신 (Free 는 주 1회)
  - 개인화 목록 생성 (쿼리로그 업로드 → 맞춤 필터)

Business (₩49,000/월)
  - 화이트라벨 배포
  - 전용 도메인 CDN
  - 커스텀 규칙 제작
```

#### 기술 구현

```
1. 수집: 국내 사이트 크롤링 → 광고 도메인 추출 (Playwright)
2. 정제: 오탐 제거 파이프라인 (주요 사이트 500개 접속 자동 검증)
3. 배포: 텍스트 파일 + GitHub + Cloudflare CDN
4. 인증: Pro 는 고유 토큰 URL (https://cdn.example.kr/list/{token}.txt)
5. 결제: Stripe / 토스페이먼츠 / 부트페이
```

사용자 설정: AdGuard Home → Filters → DNS blocklists → Add blocklist → URL 입력 (자동 갱신은 AGH 담당)

#### 핵심 이점

```
AdGuard Home 코드를 전혀 수정하지 않는다 → GPL 완전 무관
Pi-hole / uBlock Origin / AdGuard 앱 / Blocky / Unbound 가 동일 포맷 지원
  → 시장이 AdGuard Home 사용자보다 훨씬 크다
원가 사실상 0 (텍스트 파일 + CDN)
```

#### 수익 시뮬레이션

```
Free 10,000명 → Pro 전환 2% = 200명
  Pro       200 × ₩4,900  = ₩  980,000 /월
  Business   10 × ₩49,000 = ₩  490,000 /월
  MRR                     ≈ ₩1,470,000 /월  (원가 극소 → 대부분 순이익)
```

#### 리스크

```
품질 유지에 지속적 노동 필요 (오탐 = 신뢰 붕괴)
무료 경쟁 목록 존재 → "금융 화이트리스트" 등 확실한 차별점 필수
```

---

### 아이디어 3 — AdGuard-MCP + AI 네트워크 에이전트  ★★★★★

#### 컨셉

"우리 집 네트워크에 자연어로 말을 걸 수 있게 만든다."

```
사용자 (자연어)
   |
   v
AI 에이전트 (Claude / 로컬 LLM)
   | MCP
   v
AdGuard-MCP Server   ← 제작 대상 (TypeScript/Go, 비공개 가능)
   | REST + Basic Auth
   v
AdGuard Home (수정 없음 → GPL 안전)
```

#### 사용 시나리오

```
Q "우리 집에서 수상한 기기 있어?"
A "IoT-Camera-02 가 03:00 에 미확인 도메인으로 1,200회 질의했습니다.
   정상 범위는 시간당 5회 미만입니다. 격리할까요?"

Q "막아줘"
A (set_client_policy) "해당 기기를 로컬 전용(allowlist) 모드로 전환했습니다."

Q "이번 달 리포트"
A "총 질의 1,240,392 / 차단 287,441 (23.2%)
   최다 차단 doubleclick.net (34,221)
   신규 추적기 12개 발견 → 차단 목록 추가 권장
   평균 응답 18ms (전월 대비 -3ms)"

Q "특정 사이트 이미지가 안 열려"
A (search_querylog) "thumbnail9.example-cdn.com 이 EasyPrivacy 목록에
   오탐으로 걸려 있습니다. 예외 규칙을 추가할까요?"
```

#### 수익 모델 3단계

```
1단계 — OSS 공개 (수익 0, 자산 축적)
  adguard-mcp 를 MIT 로 공개 / npm·Homebrew 배포 / GitHub 별 확보
  → 해당 분야 포지셔닝 확보

2단계 — Pro 유료화 (₩9,900/월)
  ML 기반 이상 탐지 (baseline 학습)
  자동 리포트 생성 + 이메일 발송
  멀티 인스턴스 지원
  클라우드 장기 히스토리

3단계 — B2B 확장 (아이디어 1과 통합)
  "AI 네트워크 보안 관제" 포지션
  MSP / 중소기업 대상 ₩199,000/월
  사람 관제사 없이 24/7 자동 감시
```

#### 평가

```
장점: 현재 AdGuard Home MCP 서버가 사실상 부재 → 선점 기회
      AI 트렌드 × 셀프호스팅 트렌드 교차점 (양쪽 커뮤니티 유입)
      기술 난이도 적정 (MCP SDK + REST 호출)
      GPL 무관 (별도 프로세스)
      데모의 시각적 임팩트가 커서 바이럴에 유리
리스크: 개인 사용자의 구독 저항 → B2B 전환 속도가 중요
        MCP 생태계 변화 속도 → 추상화 레이어 필요
        LLM API 비용 → 로컬 LLM(Ollama) 옵션 제공
```

---

### 아이디어 4 — 플러그앤플레이 하드웨어 박스  ★★★

#### 제품

```
Raspberry Pi 5 또는 N100 미니PC
 + AdGuard Home 사전 설치
 + 한국 특화 필터 기본 적용 (아이디어 2 연계)
 + QR 스캔 초기 설정 앱
 + 전용 케이스
```

타깃: 터미널 사용이 어려운 일반 소비자 / IT 담당자 없는 소규모 사업장 / 선물용

#### 수익 구조

```
원가  Pi5(8GB) ₩120,000 + SD ₩15,000 + 케이스·전원 ₩25,000 + 조립·검수 ₩20,000 ≈ ₩180,000
판가  ₩299,000 → 대당 마진 ≈ ₩119,000 (약 40%)

구독 결합 (실질 수익원)
  "프리미엄 필터 + 원격 관리 + AI 리포트" ₩9,900/월
  100대 판매 → MRR ₩990,000  (하드웨어는 구독 획득 비용으로 간주)
```

#### 리스크

```
재고 / 물류 / A/S 부담 (소프트웨어와 완전히 다른 사업 구조)
GPL: 하드웨어 동반 배포 시 소스 제공 의무 발생
     → 대응: 수정 없이 공식 바이너리 그대로 탑재 + LICENSE 동봉 + 소스 URL 명시
전기용품 안전 인증(KC) → 완제품 미니PC 리패키징이 안전
중국산 저가 경쟁
```

권장: 아이디어 1~3 검증 후 구독 획득 수단으로 도입. 초기 단계에서는 자본 리스크가 크다.

---

### 아이디어 5 — 설치·구축 컨설팅 서비스  ★★★★

#### 서비스 메뉴

| 상품 | 가격 | 내용 |
|---|---|---|
| 가정용 원격 설치 | ₩89,000 | 원격 구축 + 공유기 설정 + 사용 교육 30분 |
| 소규모 사무실 | ₩390,000 | 온사이트 구축 + 정책 설계 + 문서화 |
| 학원 / 학교 | ₩890,000~ | 자녀보호 정책 + 시간표 + 관리자 교육 |
| 월 관리 | ₩49,000/월 | 모니터링 + 오탐 대응 + 업데이트 |
| 긴급 대응 | ₩150,000/건 | 장애 출동 |

#### 전략적 가치 (가장 중요)

```
자본 0원, 즉시 시작 가능
실제 고객의 진짜 고통을 직접 수집 → 아이디어 1~3 의 제품 스펙이 된다
첫 SaaS 고객이 여기서 나온다 (이미 신뢰 형성됨)
레퍼런스/후기 확보
한계: 시간을 파는 모델이라 확장 불가 → SaaS 전환의 계단으로만 활용
```

SaaS 를 먼저 만들면 "아무도 쓰지 않는 제품"이 될 확률이 높다.
컨설팅으로 고객 10명을 만나고, 반복되는 고통을 발견해 자동화한 것이 SaaS 가 되어야 한다.

---

### 아이디어 6 — 교육기관용 자녀보호 솔루션  ★★★★

#### 기회

학교 / 학원 / 도서관 / 청소년 시설은 유해 사이트 차단 수요가 상시 존재하나,
기존 상용 솔루션은 고가(연 수백만원) · 성능 저하 · 관리 난이도 문제를 갖는다.

#### 활용 가능한 기존 기능 (모두 이미 구현됨)

```
internal/filtering/      성인 사이트 차단 (parental control)
/safesearch/enable       구글·유튜브·빙 세이프서치 강제
internal/schedule/       "수업 시간(09~15시)만 SNS 차단" 시간표
internal/client/         학년별/교실별 차등 정책
internal/querylog/       접속 기록 (감사 대응)
blocked_services 151개   유튜브·틱톡·인스타 원클릭 차단
/access/set              허용 목록 전용 모드 (완전 통제)
```

#### 패키징 (가칭 EduGuard)

```
기본형  연 ₩1,200,000 (~200 학생)
  AGH 구축 + 교육용 필터 정책 프리셋
  학년별 정책 템플릿
  교사용 초단순 관리 웹 (React)
  월간 리포트

확장형  연 ₩3,600,000 (~1,000 학생)
  다중 캠퍼스 통합 관리 (아이디어 1 연계)
  학생 계정 연동
  학부모 알림 포털
  온사이트 지원 + 연 2회 교육
```

#### 평가

```
장점: 예산이 편성된 시장 / 갱신율 매우 높음 / 레퍼런스 파급력 큼 / 상용 대비 저가 가능
리스크: 조달·입찰 절차, 공공 영업 사이클 6개월~1년
        책임 소재 (차단 실패 사고) → 계약서 면책 조항 필수
```

---

### 아이디어 7 — 콘텐츠 + 교육  ★★

```
유튜브   "우리 집 광고 완전 차단하기" 시리즈
        → 애드센스 + 하드웨어 어필리에이트 + 제품 유입

전자책   "AdGuard Home 완전정복" ₩19,000
        → AGHTechDoc.md(45KB) 를 한국어로 재구성

온라인강의 "Go 로 배우는 네트워크 프로그래밍 — AdGuard Home 코드 분석"
        → 97,000줄 실전 코드 분석. 희소성 높음
        → ₩99,000 × 200명 = ₩19,800,000

기술블로그 → 개발자 포지셔닝 → 강의/컨설팅 유입
```

단독 수익보다 아이디어 1~3 의 마케팅 엔진으로 평가하는 것이 적절하다.

---

## 13. 최종 결론 및 로드맵

### 13-1. 종합 비교

| # | 아이디어 | 초기비용 | 난이도 | 수익잠재 | 실현속도 | GPL리스크 | 종합 |
|---|---|---|---|---|---|---|---|
| 1 | B2B 관리 SaaS | 중 | 높음 | 매우높음 | 느림 | 안전 | ★★★★★ |
| 2 | 한국 필터 구독 | 0 | 낮음 | 중상 | 빠름 | 무관 | ★★★★★ |
| 3 | AdGuard-MCP | 낮음 | 중간 | 높음 | 보통 | 안전 | ★★★★★ |
| 4 | 하드웨어 박스 | 높음 | 중간 | 중상 | 보통 | 주의 | ★★★ |
| 5 | 컨설팅 | 0 | 낮음 | 중 | 즉시 | 무관 | ★★★★ |
| 6 | 교육기관 | 중 | 중간 | 높음 | 느림 | 안전 | ★★★★ |
| 7 | 콘텐츠 | 0 | 낮음 | 낮음 | 보통 | 무관 | ★★ |

### 13-2. 권장 실행 로드맵

```
0~3개월    아이디어 5(컨설팅) + 7(블로그) 시작
           → 현금흐름 확보 + 고객 고통 수집 + 인지도

2~5개월    아이디어 2(한국 필터) — Free 목록 우선 공개
           → GitHub 인지도 → Pro 전환
           → 자동화 후에는 유지 비용 극소

4~9개월    아이디어 3(adguard-mcp) OSS 공개
           → AI + 셀프호스팅 양 커뮤니티 확보
           → 데모 영상 바이럴

8~18개월   아이디어 1(B2B SaaS) 본격 개발
           → 컨설팅 고객을 첫 파일럿으로 전환 (신뢰 기반 확보)
           → 아이디어 3 을 Pro 기능으로 흡수

18개월+    아이디어 6(교육기관) 진출 또는 4(하드웨어) 번들
```

### 13-3. 핵심 원칙

> **"코드를 만들지 말고, 고통을 없애라."**
>
> AdGuard Home 은 이미 완성도가 높다. 97,000줄이 검증되어 있고 기업이 유지보수한다.
> 수익 기회는 "DNS 서버를 더 잘 만드는 것"이 아니라
> **"이것을 쓰는 사람들에게 남아 있는 고통"** 에 있다.
>
> | 남은 고통 | 해결 아이디어 |
> |---|---|
> | 인스턴스 수십 개 관리 부담 | 1 |
> | 한국 사이트 차단 미흡 / 금융 사이트 오탐 | 2 |
> | 로그가 많아 무엇을 봐야 할지 모름 | 3 |
> | 설치·설정을 못 함 | 4, 5 |
> | 교육기관용 정책 설계를 모름 | 6 |

### 13-4. 첫 실행 권장

**아이디어 2(한국 특화 필터 목록)** 부터 시작할 것을 권한다.

```
자본 0원 / 1인 수행 가능 / 원가 0 / 실패 시 손실 없음
성공 시 자동 수익 + 나머지 제품의 마케팅 채널로 전환
→ 리스크 대비 보상 비율이 가장 우수
```

---

## 부록 A. 참고 링크

| 대상 | URL |
|---|---|
| **이 저장소 (fork)** | https://github.com/bmshin94/AdGuardHome |
| 원본 저장소 (업스트림) | https://github.com/AdguardTeam/AdGuardHome |
| 공식 Wiki | https://github.com/AdguardTeam/AdGuardHome/wiki |
| Getting Started (KB) | https://adguard-dns.io/kb/adguard-home/getting-started/ |
| REST API 명세 | https://github.com/AdguardTeam/AdGuardHome/tree/master/openapi |
| Docker Hub | https://hub.docker.com/r/adguard/adguardhome |
| Snap Store | https://snapcraft.io/adguard-home |
| Python 클라이언트 | https://pypi.org/project/adguardhome/ |
| Home Assistant 통합 | https://www.home-assistant.io/integrations/adguard/ |
| AdGuard DNS (상용) | https://adguard-dns.io/ |
| AdGuard 공식 | https://adguard.com/ |
| 커뮤니티 (Reddit) | https://reddit.com/r/Adguard |

## 부록 B. 조사 방법 및 수치 근거

| 수치 | 확인 방법 |
|---|---|
| Go 97,191줄 / 451파일 | `find internal -name '*.go'` + `wc -l` |
| 패키지별 줄 수 | 디렉터리별 `find`+`cat`+`wc -l` 집계 |
| API 엔드포인트 81개 | `grep -cE "^  '?/" openapi/openapi.yaml` |
| openapi.yaml 3,389줄 | `wc -l openapi/openapi.yaml` |
| 차단 서비스 151개 | `grep -cE '^\s+ID:\s+"' internal/filtering/servicelist.go` |
| Claude 차단 항목 | `internal/filtering/servicelist.go:572-576` |
| securitySchemes basicAuth | `openapi/openapi.yaml:3386-3389` |
| AI SDK 미사용 | 전체 `.go` 대상 `grep` → import 0건 |
| CLIENT_DIR=client_v2 | `Makefile` |
| Go 1.26.8 고정 | `Makefile` `GOTOOLCHAIN` |
| 라이선스 GPL-3.0 | `LICENSE.txt` |
| 패키지 역할 | 각 패키지 `doc` 주석(`// Package xxx ...`) |

## 부록 C. 문서 메타

| 항목 | 값 |
|---|---|
| 작성 도구 | Claude Code (Opus 5) |
| 페르소나 | 카리나 (`CLAUDE.md` 정의) |
| 대화 주제 | AdGuard Home 전수조사 → 쉬운 재설명 → Q&A 7문 → 수익화 심화 → 문서화 |
| 작성 브랜치 | `claude/practical-carson-nn640v` |
| 기준 커밋 | `3ac1642` |
