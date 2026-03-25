# claude-code-plugins

프론트엔드 팀 공용 Claude Code 플러그인 마켓플레이스

## 설치 방법

```shell
# 1. 마켓플레이스 추가
/plugin marketplace add djatjdwns28/claude-code-plugins

# 2. 플러그인 설치
/plugin install fe-code-review@claude-code-plugins

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
    "fe-code-review@claude-code-plugins": true
  }
}
```

## 포함된 플러그인

### fe-code-review (v1.0.0)

프론트엔드 코드 리뷰 스킬. 보안, 성능, 아키텍처 관점에서 코드를 분석합니다.

**사용법:**
```
/fe-code-review
코드 리뷰 시작해줘
```

**주요 기능:**
- 사전 자동 분석 (Security / Performance / Architecture)
- 파일별 인터랙티브 리뷰
- 3관점 전문가 평가
- 리뷰 체크리스트 기반 검사
- 개선 제안 및 참고 자료 제공

## 업데이트

```shell
/plugin marketplace update claude-code-plugins
/plugin update fe-code-review@claude-code-plugins
/reload-plugins
```
