# claude-code-plugins

프론트엔드 팀 공용 Claude Code 플러그인 마켓플레이스

## 설치 방법

```shell
# 1. 마켓플레이스 추가
/plugin marketplace add djatjdwns28/claude-code-plugins

# 2. 플러그인 설치 (한 번이면 모든 스킬 사용 가능)
/plugin install fe-tools@claude-code-plugins

# 3. 현재 세션에 반영
/reload-plugins
```

### 프로젝트 settings.json으로 자동 설치 (추천)

프로젝트 `.claude/settings.json`에 추가하면 팀원이 trust 시 자동으로 설치됩니다:

```json
{
  "extraKnownMarketplaces": {
    "claude-code-plugins": {
      "source": {
        "source": "github",
        "repo": "djatjdwns28/claude-code-plugins"
      }
    }
  },
  "enabledPlugins": {
    "fe-tools@claude-code-plugins": true
  }
}
```

## 포함된 스킬

### fe-tools 플러그인 (v1.0.0)

| 스킬 | 트리거 | 설명 |
|------|--------|------|
| fe-code-review | "코드 리뷰", "code review" | 보안/성능/아키텍처 관점 파일별 인터랙티브 코드 리뷰 |
| pr-create | "PR 올려줘", "PR 생성" | 변경사항 분석 -> PR 제목/본문 자동 생성 -> gh pr create |
| pr-fix | "리뷰 반영", "review fix" | PR 리뷰 코멘트 수집 -> 선택적 수정 -> diff 확인 -> 커밋 |
| component-gen | "컴포넌트 만들어 Button" | Vue/React 자동 감지, 팀 컨벤션 맞춤 보일러플레이트 생성 |
| bundle-check | "번들 체크", "bundle check" | import 분석, 위험 패턴 감지, 번들 최적화 제안 |

## 업데이트

```shell
/plugin marketplace update claude-code-plugins
/plugin update fe-tools@claude-code-plugins
/reload-plugins
```
