# Redmine PMS Daily Report Workflow

Redmine PMS 이슈를 자동으로 수집하여 일일 리포트를 생성하고, Notion에 저장한 뒤 Slack으로 알림을 보내는 n8n 워크플로우입니다.

## 기능

- Redmine에서 5개 프로젝트의 이슈 자동 크롤링
- 이슈를 4가지 카테고리로 자동 분류 (신규, 처리완료, 처리중, 잔여)
- Notion 데이터베이스에 일일 리포트 페이지 생성
- Slack 채널로 요약 알림 전송
- 워크플로우 실패 시 Slack 에러 알림

## 이슈 분류 기준

| 카테고리 | 조건 |
|---------|------|
| 신규 | 기준일에 등록된 이슈 |
| 처리완료 | 기준일에 완료 + 종료 상태 |
| 처리중 | 완료일 없음 + 진척도 10~79% + 종료 상태 아님 |
| 잔여 | 위 조건에 해당 안 함 + 진척도 80% 미만 |

**종료 상태**: 처리 완료, 처리 반려, 종료 [해결], 종료 [미해결], 종료 [폐기], 종료 [테스트오류]

## 사전 요구사항

- [n8n](https://n8n.io/) (Self-hosted 또는 Cloud)
- Redmine 계정 (이슈 조회 권한 필요)
- Notion Integration 및 데이터베이스
- Slack App (Bot Token 필요)

## 설치 방법

### 1. 워크플로우 가져오기

1. n8n 대시보드에서 **Settings** > **Import from File** 선택
2. `redmine-pms-report-workflow.json` 파일 업로드
3. 워크플로우가 생성되면 **Inactive** 상태로 표시됨

### 2. 환경변수 설정

n8n 환경변수에 다음 값들을 설정합니다:

| 환경변수 | 설명 | 예시 |
|---------|------|------|
| `REDMINE_BASE_URL` | Redmine 서버 URL | `https://redmine.example.com` |
| `REDMINE_USERNAME` | Redmine 로그인 ID | `your_username` |
| `REDMINE_PASSWORD` | Redmine 비밀번호 | `your_password` |
| `NOTION_DATABASE_ID` | Notion 데이터베이스 ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |

**Docker 사용 시:**
```bash
docker run -d \
  -e REDMINE_BASE_URL=https://redmine.example.com \
  -e REDMINE_USERNAME=your_username \
  -e REDMINE_PASSWORD=your_password \
  -e NOTION_DATABASE_ID=your_database_id \
  n8nio/n8n
```

**docker-compose 사용 시:**
```yaml
environment:
  - REDMINE_BASE_URL=https://redmine.example.com
  - REDMINE_USERNAME=your_username
  - REDMINE_PASSWORD=your_password
  - NOTION_DATABASE_ID=your_database_id
```

### 3. n8n Credentials 설정

#### Notion API
1. [Notion Developers](https://developers.notion.com/)에서 Integration 생성
2. n8n에서 **Credentials** > **New** > **Notion API** 선택
3. Integration Token 입력
4. 대상 데이터베이스에 Integration 연결 (Share > Invite)

#### Slack API
1. [Slack API](https://api.slack.com/apps)에서 App 생성
2. OAuth & Permissions에서 Bot Token Scopes 추가:
   - `chat:write`
   - `chat:write.public`
3. n8n에서 **Credentials** > **New** > **Slack API** 선택
4. Bot User OAuth Token 입력

### 4. Notion 데이터베이스 설정

데이터베이스에 다음 속성이 필요합니다:

| 속성 이름 | 타입 |
|----------|------|
| Name | Title |
| Date | Date |
| Tags | Select |

### 5. 프로젝트 URL 수정

`Set Config` 노드에서 PROJECTS 배열을 환경에 맞게 수정합니다:

```javascript
PROJECTS: [
  {
    name: '프로젝트명',
    url: `${REDMINE_BASE_URL}/projects/project_id/issues?query_id=123&per_page=100`
  },
  // ... 추가 프로젝트
]
```

### 6. Slack 채널 수정

`Set Config` 노드에서 SLACK_CHANNELS 배열을 수정합니다:

```javascript
SLACK_CHANNELS: ['#your-channel']
```

## 실행 방법

### 수동 실행
1. n8n 대시보드에서 워크플로우 열기
2. **Execute Workflow** 버튼 클릭
3. 기준일: 실행 당일

### 자동 스케줄 실행

워크플로우에는 자동 스케줄이 내장되어 있습니다:

| 스케줄 | 수행 시간 | 기준일 | 비고 |
|--------|----------|--------|------|
| 오전 리포트 | 매일 08:00 | 전일 (D-1) | 화~금 실행 |
| 금요일 오후 | 금 17:00 | 당일 (금요일) | 금요일 추가 리포트 |

### 자동 스킵 조건

스케줄 실행 시 기준일이 **주말(토/일)**이면 자동으로 스킵됩니다.

예시:
- 월요일 08:00 실행 → 기준일 일요일 → **스킵**
- 화요일 08:00 실행 → 기준일 월요일 → **정상 실행**

## 출력 예시

### Notion 페이지
```
[PMS 일일 리포트] 2025-12-07

📊 요약
• 결함: 신규 2건 / 완료 5건 / 처리중 10건 / 잔여 3건
• 신규기능개발: 신규 1건 / 완료 2건 / 처리중 5건 / 잔여 2건
...

─────────────────────────

## 결함 (신규 2건, 완료 5건, 처리중 10건, 잔여 3건)

### 📋 신규
• #12345 [긴급] 로그인 오류 수정 (25/12/10) - 홍길동
...
```

### Slack 메시지
```
📋 PMS 일일 리포트 (2025-12-07)

• 결함: 신규 2건 / 완료 5건 / 처리중 10건 / 잔여 3건
• 신규기능개발: 신규 1건 / 완료 2건 / 처리중 5건 / 잔여 2건
...

🔗 상세 보기
```

## 문제 해결

### 로그인 실패
- Redmine 계정 정보 확인
- Redmine 서버 접근 가능 여부 확인
- CSRF 토큰 관련 오류 시 Redmine 버전 확인

### Notion API 오류
- Integration이 데이터베이스에 연결되었는지 확인
- 데이터베이스 속성 이름 확인 (Name, Date, Tags)
- API 요청 제한 초과 여부 확인

### 이슈가 표시되지 않음
- Redmine 쿼리 ID가 유효한지 확인
- 해당 쿼리에 접근 권한이 있는지 확인
- `per_page=100` 설정 확인

## 라이선스

MIT License
