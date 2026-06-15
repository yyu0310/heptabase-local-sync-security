# Heptabase Local Sync Security

**프라이버시 우선, AI 안전. Heptabase를 구조화된 Markdown으로 지속 동기화하는 로컬 도구.**

AI 에이전트가 볼 수 있는 카드를 정확히 당신이 결정합니다. 나머지는 에이전트의 시야에 들어오지 않습니다.

[English](README.md) · [繁體中文](README.zh-TW.md) · [简体中文](README.zh-CN.md) · [日本語](README.ja.md) · 한국어 · [Tiếng Việt](README.vi.md) · [Español](README.es.md) · [Français](README.fr.md) · [Deutsch](README.de.md) · [العربية](README.ar.md) · [עברית](README.he.md) · [Русский](README.ru.md) · [Українська](README.uk.md)

> 비공식 도구입니다. Heptabase와 제휴 또는 보증 관계가 없습니다. 현재는 macOS만 지원합니다.

---

## 왜 만들었나

단순한 목표에서 시작했습니다. Heptabase를 Claude Code에 연결해 AI 에이전트가 내 노트를 읽게 하고 싶었습니다.

공식 경로는 Heptabase 자체 [CLI](https://github.com/heptameta/heptabase-cli-skills)로, 앱 내 Settings, AI Features, CLI에서 활성화합니다. **fail-open** 방식을 채택하여 한 번 승인하면 에이전트가 전체 지식 베이스를 읽을 수 있습니다. `heptabase-mcp` 서버 같은 서드파티 도구도 같은 방식입니다. 지식 베이스의 모든 카드를 공유해도 괜찮다면 문제없습니다. 하지만 대부분의 사람처럼 AI에게 보여주고 싶은 카드 바로 옆에 기밀 카드를 두고 있다면, 이 경로는 작동하지 않습니다.

문제의 핵심은 프라이버시 벽이 'AI가 무엇을 읽을 수 있는가'라는 경계에 있어야 한다는 점입니다. 그 경계는 Heptabase 외부, 즉 노트를 에이전트에 전달하는 방식에 있습니다. 그래서 이런 도구가 진정으로 해야 할 일은 **기밀 카드를 AI가 닿을 수 없는 곳에 두고, 나머지만 AI가 읽을 수 있는 Markdown으로 내보내는 것**입니다. 노트를 동기화하는 것은 쉬운 절반에 불과합니다.

## 이름에 대하여

이름은 세 부분으로 나뉘며 각각 다른 것을 설명합니다. `heptabase`는 데이터 소스, `local-sync`는 메커니즘, `security`는 역할을 나타냅니다.

Security는 두 가지 의미를 담고 있습니다. 첫 번째는 보안입니다. 이 도구는 AI와 지식 베이스 사이에 서서 fail-closed 메커니즘으로 어떤 카드가 AI의 시야에 들어가고 어떤 것이 차단되는지를 결정합니다. 데이터 유출을 걱정할 필요가 없습니다. 두 번째는 성실한 비서와 같은 역할입니다. 백그라운드에서 조용히 노트를 정리하고 설정한 위치에 15분마다 자동 동기화하여 AI 도구를 열면 바로 사용할 수 있는 컨텍스트가 준비되어 있습니다. 문지기와 잡무 처리, 그것이 이 이름의 의미입니다.

## 기능

Heptabase Local Sync Security는 로컬 Heptabase 데이터베이스를 직접 읽어 선택한 카드를 원하는 위치에 Markdown 파일로 저장합니다. `launchd` 작업이 15분마다 실행되어 Markdown을 노트와 동기화합니다. AI 에이전트는 내보낸 Markdown 폴더만 읽고, 데이터베이스에는 절대 접근하지 않습니다.

- **라이브 로컬 데이터베이스 직접 읽기.** Heptabase는 2025년 말 [자동 로컬 백업](https://support.heptabase.com/en/articles/11064116-how-does-auto-backup-work-in-heptabase) 제공을 중단했습니다. 라이브 DB 직접 읽기가 지속적인 로컬 동기화의 신뢰할 수 있는 경로가 된 이유입니다.
- **구조 보존 변환.** 표, bullet / todo / toggle 목록, 중첩 섹션, 동영상을 Heptabase의 ProseMirror 스키마에서 역공학으로 분석해 깔끔한 Markdown으로 변환합니다.
- **임의 목적지 라우팅.** 각 화이트보드를 개별 폴더에 저장할 수 있으며, 절대 경로로 특정 보드를 별도 프로젝트에 직접 배치하는 것도 가능합니다.

## fail-closed 프라이버시 모델

카드는 두 개의 명시적 허용 목록 중 하나에 해당할 때만 내보냅니다. 기본적으로 아무것도 읽지 않습니다.

| 출처 | 규칙 |
|---|---|
| **`whitelist_whiteboards`** | 지정한 화이트보드. 각 보드의 *표면*에 있는 카드만 읽습니다. 서브 화이트보드는 따라가지 않습니다. 포함하려면 이름을 추가하세요. |
| **`card_map`** | `제목 -> 정확한 경로` 레이어. 이 제목들은 항상 동기화되고, 경로가 우선입니다. |
| **`blacklist_whiteboards`** | 이 보드의 카드는 내용을 읽기 *전에* 제외됩니다. 블랙리스트가 화이트리스트보다 우선하므로, 실수로 두 보드에 놓인 카드도 차단됩니다. |
| **서브 화이트보드(미지정)** | 카드를 서브 화이트보드로 옮기면 `whiteboard_id`가 바뀌어 표면 스캔에서 보이지 않습니다. 규칙을 기억하지 않아도 구조 자체가 제외합니다. |

더 중요한 것은, 카드의 제목이나 내용에 접근하는 모든 쿼리는 화이트리스트 화이트보드 id 또는 `card_map` 제목으로 제한된다는 점입니다. 화이트리스트에 없는 카드의 제목과 내용은 메모리에 전혀 읽히지 않습니다.

여기서 두 가지 설계 원칙이 도출됩니다. **구조적 제외가 감산적 제외보다 낫습니다**, 즉 쿼리가 닿지 않는 카드가 읽은 후 필터링하는 것보다 안전합니다. 이 도구는 카드가 유출되었는지 걱정하지 않아도 되도록 설계되었으며, 그 보장은 사후 확인이 아닌 구조에 의해 담보됩니다.

## 비교

| | Heptabase Local Sync Security | 공식 Heptabase CLI | 기타 내보내기 도구 |
|---|---|---|---|
| 프라이버시 모델 | Fail-closed 허용 목록 | Fail-open (전체 지식 베이스) | 전체 내보내기 |
| 지속적 로컬 동기화 | 있음 (`launchd`, 15분) | 요청 시 읽기 | 일회성 내보내기 |
| 라이브 로컬 DB 직접 읽기 | 있음 | 경우에 따라 다름 | 백업 파일 필요한 경우 많음 |
| 구조 보존 Markdown | 표, 목록, 섹션, 동영상 | 경우에 따라 다름 | 경우에 따라 다름 |
| 보드별 목적지 라우팅 | 있음, 절대 경로 포함 | 없음 | 없음 |

Heptabase의 완전한 대체품이 아닙니다. '공식보다 낫다'는 주장은 세 가지 축에서만 성립합니다: 제어 가능한 프라이버시, 지속적 로컬 동기화, 구조 충실성. 대상은 의도적으로 좁습니다. AI가 볼 수 있는 것에 신경 쓰는 Heptabase 헤비 유저 macOS 사용자입니다. 그것이 당신이라면, 이 도구는 정확히 당신의 상황을 위해 만들어졌습니다.

## 설치

요구 사항: macOS, Python 3.9+, Heptabase 데스크톱 앱 설치.

```bash
git clone https://github.com/yyu0310/heptabase-local-sync-security.git
cd heptabase-local-sync-security
cp config.example.json config.json
```

`config.json`을 편집합니다 (모든 필드에 인라인 주석 설명이 있습니다):

1. `db_path`가 로컬 `hepta.db`를 가리키는지 확인합니다.
2. `base_output_dir`과 `board_output_dir`을 Markdown을 저장할 위치로 설정합니다.
3. `whitelist_whiteboards`에 내보낼 화이트보드를 나열합니다.
4. `card_map`에 정확한 경로 재정의가 필요한 제목을 추가합니다.

먼저 미리보기를 실행합니다 (아무것도 저장하지 않습니다):

```bash
python3 heptabase_sync.py --dry
```

계획이 맞으면 실제로 실행합니다:

```bash
python3 heptabase_sync.py
```

### 15분마다 자동 동기화

```bash
cp com.example.heptasieve.plist ~/Library/LaunchAgents/
# 복사한 파일을 편집합니다: 절대 경로를 설정하고 python3 경로를 확인합니다
launchctl load ~/Library/LaunchAgents/com.example.heptasieve.plist
```

## AI 에이전트와 함께 사용하기

이 도구는 에이전트가 읽을 수 있는 문서를 포함하고 있어, 직접 단계를 따르는 대신 AI 코딩 에이전트와 대화로 설정할 수 있습니다:

- [`AGENTS.md`](AGENTS.md)와 [`CLAUDE.md`](CLAUDE.md)는 에이전트가 이 도구를 이해하고 설정하는 방법을 알려줍니다.
- [`llms.txt`](llms.txt)는 LLM을 위한 문서 인덱스입니다.
- [`skills/setup-heptasieve/`](skills/setup-heptasieve/)는 한 번의 요청으로 전체 설정을 안내하는 Claude Code 스킬입니다.

에이전트를 내보낸 Markdown 폴더로 향하게 하고, `hepta.db`에는 절대 향하지 않게 하세요. 그 분리가 이 도구의 핵심입니다.

### 시나리오 1: Claude Code 프로젝트 어시스턴트

양적 트레이딩 도구를 개발하고 있다고 가정합시다. Heptabase에는 `트레이딩 전략 연구`, `시스템 설계 노트`, `개인 재무 기록` 세 가지 화이트보드가 있습니다. 처음 두 개만 `whitelist_whiteboards`에 추가하면 도구가 15분마다 `/projects/trading/heptabase-export/`에 동기화합니다. 그런 다음 프로젝트의 CLAUDE.md에 해당 폴더 경로를 추가하면 Claude Code가 배경 지식으로 읽습니다. 재무 기록 화이트보드는 허용 목록에 없으므로 Claude Code는 제목조차 볼 수 없습니다.

### 시나리오 2: 도구 간 개인 기억 레이어

서로 다른 AI 도구 간에는 공통 메모리가 없어 도구를 바꿀 때마다 컨텍스트를 처음부터 설명해야 합니다. 자주 사용하는 참조 노트, 작업 컨텍스트, 연구 요약을 Markdown 폴더에 동기화해 두면 로컬 파일을 읽을 수 있는 어떤 AI 도구든 즉시 활용할 수 있습니다. 어떤 화이트보드가 허용 목록에 들어가고 어떤 것이 잠기는지는 설정으로 결정됩니다. AI 도구의 판단을 신뢰하는 것이 아니라 당신이 제어합니다.

## 작동 원리

아키텍처 세부 사항은 [`ARCHITECTURE.md`](ARCHITECTURE.md)를 참조하세요. 데이터 흐름, `build_plan` 내의 fail-closed 순서, 읽는 데이터베이스 테이블, 코드 수정 시 보존해야 할 프라이버시 불변 조건이 포함되어 있습니다.

## 제한 사항과 솔직한 주의점

- **스키마는 취약합니다.** Heptabase의 내부 데이터베이스 구조에 의존합니다. Heptabase 업데이트로 작동이 중단될 수 있습니다. 본질적으로 비공식 도구입니다.
- **라이브 DB 읽기는 공식 승인을 받지 않았습니다.** 실제로는 잘 작동하고 읽기 전용이지만, 지원되는 통합 방식이 아님을 알아두세요.
- **macOS만 지원합니다.** 현재 경로와 `launchd` 설정은 macOS를 전제로 합니다.

## 라이선스

[MIT](LICENSE).
