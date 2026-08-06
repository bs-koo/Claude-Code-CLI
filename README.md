# Claude Code CLI 가이드

> Windows 기준 설치부터 플러그인, 토큰 최적화까지

## 소개

사업부 내 Claude Code CLI 사용을 위한 가이드 페이지입니다.

**가이드 페이지**: https://bs-koo.github.io/Claude-Code-CLI/

## 포함 내용

| # | 섹션 | 설명 |
|---|------|------|
| 1 | 터미널 프로그램 추천 | WezTerm / Windows Terminal |
| 2 | 설치 전 확인 | 구독 요건, Windows 10 이상, Git for Windows |
| 3 | 설치 | Native Install / WinGet / npm 3가지 방법 |
| 4 | Alias 및 실행 모드 | cc/ccd/ccr alias, 바이패스 모드 권한 설정 |
| 5 | 화면 렌더링 | `/tui fullscreen` 권장 설정과 달라지는 점 |
| 6 | 슬래시 명령어 | /help, /config, /model, Context Window |
| 7 | 키보드 단축키 | Shift+Tab 모드 전환, Alt+V 이미지 붙여넣기 |
| 8 | Effort 설정 | low·medium·high·xhigh·max 5단계, ultracode |
| 9 | oh-my-claudecode | 멀티 에이전트 플러그인 설치 및 설정 |
| 10 | 공식 플러그인 마켓 | claude-plugins-official, superpowers |
| 11 | HUD 설정 | claude-hud 상태 바 설치·설정 |
| 12 | RTK | 토큰 60-90% 절감 도구 설치 및 연동 |
| 13 | 사업부 플러그인 | SEF-2026, oh-my-gx (선택) |

## 기술 스택

- 단일 HTML 파일 (외부 의존성 없음, Google Fonts만 사용)
- GitHub Pages로 정적 배포
- 반응형 디자인 (모바일/데스크톱)
- 디자인: Claude 디자인 시스템 기반 — cream 캔버스 + coral 액센트 + dark navy 3면 서피스, EB Garamond / Noto Serif KR 세리프 디스플레이

## 배포

`main` 브랜치에 push하면 GitHub Actions가 자동으로 GitHub Pages에 배포합니다.

## 참고 출처

- [Claude Code 공식 문서](https://code.claude.com/docs/en/overview)
- [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode)
- [RTK (Rust Token Killer)](https://github.com/rtk-ai/rtk)
