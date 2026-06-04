# 이 저장소를 Cursor와 함께 사용하기

> [English](./CURSOR.md) | 한국어

이 프로젝트에는 **Cursor 프로젝트 규칙**이 포함되어 있어, 여기서 작업할 때 Karpathy에서 영감을 받은 행동 가이드라인이 자동으로 적용됩니다.

## 이 저장소에서

1. Cursor에서 폴더를 엽니다.
2. 규칙 [`.cursor/rules/karpathy-guidelines.mdc`](.cursor/rules/karpathy-guidelines.mdc)는 `alwaysApply: true`로 커밋되어 있으므로, 추가 설치 단계가 필요 없습니다.
3. Cursor에서 **Settings → Rules**(또는 프로젝트 규칙 UI)에서 `karpathy-guidelines`가 나타나는 것을 확인할 수 있습니다.

## 다른 프로젝트에서 같은 가이드라인 사용하기

**Cursor (권장):** `.cursor/rules/karpathy-guidelines.mdc`를 해당 프로젝트의 `.cursor/rules/` 디렉터리에 복사하세요(필요하면 폴더를 만드세요). 원하는 대로 기존 규칙과 조정하거나 병합하세요.

**다른 도구:** 어떤 스택이 루트 지침 파일만 지원한다면, 대신 [`CLAUDE.md`](CLAUDE.md)를 그 프로젝트에 복사하세요(또는 그 내용을 기존 지침에 병합하세요).

## 선택 사항: 개인 Agent Skills

같은 내용을 `~/.cursor/skills` 아래의 재사용 가능한 스킬로 두고 싶다면, [`skills/karpathy-guidelines/SKILL.md`](skills/karpathy-guidelines/SKILL.md)를 사용하세요. 개인 스킬 디렉터리에 복사하거나 심볼릭 링크를 걸 수 있습니다. 다른 스킬에 쓰는 레이아웃을 그대로 사용하세요.

## Claude Code vs Cursor

- **Claude Code:** 플러그인 마켓플레이스와 [`README.md`](README.md) 안내를 통해 설치하세요. 플러그인은 이 저장소의 스킬을 노출합니다. 프로젝트별 사용은 `CLAUDE.md`에 의존할 수도 있습니다.
- **Cursor:** 위에서 설명한 대로 커밋된 `.cursor/rules/` 파일을 사용하세요. Cursor는 기본적으로 `.claude-plugin/`이나 `CLAUDE.md`를 읽지 않습니다.

## 기여자를 위해

네 가지 원칙을 변경할 때는 **[`CLAUDE.md`](CLAUDE.md)**와 **[`.cursor/rules/karpathy-guidelines.mdc`](.cursor/rules/karpathy-guidelines.mdc)**를 동기화 상태로 유지하세요. 게시된 스킬/플러그인 텍스트도 일치해야 한다면 **[`skills/karpathy-guidelines/SKILL.md`](skills/karpathy-guidelines/SKILL.md)**도 함께 업데이트하세요.
