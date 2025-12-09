# CLAUDE.md - Redmine PMS Daily Report Workflow

이 파일은 Claude의 다른 세션에서 이 프로젝트를 이해하기 위한 참고 문서입니다.

## 프로젝트 개요

n8n 워크플로우를 통해 Redmine PMS에서 5개 프로젝트의 이슈를 크롤링하여 일일 리포트를 생성하고, Notion에 저장한 뒤 Slack으로 알림을 보내는 자동화 시스템입니다.

## 주요 파일

- `redmine-pms-report-workflow.json` - n8n 워크플로우 정의 파일
- `ecosystem.config.js` - PM2 설정 파일 (환경변수 포함, .gitignore에 추가됨)

## 워크플로우 구조 (22개 노드)

### 메인 플로우
```
[Triggers] ─────────────────────────────────────────────────────────────────
Manual Trigger ──────────┐
Schedule Morning (08:00) ├─→ Determine Base Date → Should Run? → Set Config
Schedule Friday PM (17:00)┘        ↓                    ↓
                              (기준일 계산)         (false = 스킵)
                                                        ↓
Set Config → Get Login Page → Extract CSRF Token → Login to Redmine
→ Extract Session Cookies → Split Projects → Get Issues Page → Parse Issues HTML
→ Classify Issues → Aggregate Projects → Generate Report → Create Notion Page
→ Prepare Append Chunks → Has More Blocks? → [Append Blocks to Page] → Prepare Slack Message
→ Send Slack Message
```

### 에러 처리 플로우
```
Error Trigger → Format Error Message → Send Error to Slack
```

## 스케줄링 로직

### 트리거 타입별 기준일 (TODAY)

| 트리거 | 수행 시간 | 기준일 (TODAY) | 비고 |
|--------|----------|----------------|------|
| Manual Trigger | 즉시 | 당일 | 수동 실행 |
| Schedule Morning | 08:00 화~금 | D-1 (어제) | 전날 리포트 |
| Schedule Friday PM | 17:00 금 | 당일 (금요일) | 금요일 추가 리포트 |

### 스킵 조건 (스케줄 트리거만 해당)
- **주말**: 기준일이 토/일인 경우 스킵
- 예: 월요일 08:00 → 기준일 일요일 → 스킵
- 예: 토요일 08:00 → 기준일 금요일 → 정상 실행

## 핵심 노드 설명

### 1. Determine Base Date (신규)
- 트리거 타입 판별 (manual / morning / friday_pm)
- 기준일 계산: Manual=당일, Morning=D-1, Friday PM=당일
- 주말 체크하여 `shouldRun` 플래그 설정
- 한국 시간대(KST) 적용

### 2. Should Run? (신규)
- `shouldRun === true`인 경우만 다음 노드로 진행
- `false`인 경우 워크플로우 종료 (스킵)

### 3. Set Config
- 환경변수에서 설정값 로드
- Determine Base Date에서 계산된 TODAY 값 사용
- 5개 프로젝트 URL 정의

### 4. Redmine 로그인 처리
- **Get Login Page**: 로그인 페이지에서 CSRF 토큰 추출
- **Extract CSRF Token**: `authenticity_token` 파싱
- **Login to Redmine**: POST 요청으로 로그인 (302 리다이렉트 예상)
- **Extract Session Cookies**: 로그인 후 새 세션 쿠키만 사용

### 5. Parse Issues HTML
- Redmine HTML에서 정규식으로 이슈 데이터 추출
- 추출 필드: 이슈ID, 제목, 상태, 우선순위, 담당자, 완료기한, 등록일, 완료일, 진척도

### 6. Classify Issues
이슈를 4가지 카테고리로 분류:
- **신규**: 기준일에 등록된 이슈 (`registeredDate === TODAY`)
- **처리완료**: 기준일에 완료 + 종료 상태 (`completedDate === TODAY && isClosedStatus`)
- **처리중**: 완료일 없음 + 진척도 10~79% + 종료 상태 아님
- **잔여**: 위 조건에 해당 안 함 + 진척도 < 80%

종료 상태 목록:
```javascript
const CLOSED_STATUSES = [
  '처리 완료', '처리 반려', '종료 [해결]',
  '종료 [미해결]', '종료 [폐기]', '종료 [테스트오류]'
];
```

### 7. Generate Report
- Notion API 형식으로 블록 생성
- **100개 블록 제한 처리**: 청크로 분할
- 페이지 구조: 요약 callout → 구분선 → 프로젝트별 heading_2 → 카테고리별 heading_3 → 이슈 목록

### 8. Notion 페이지 생성
- **Create Notion Page**: 첫 100개 블록으로 페이지 생성 (아이콘: ⛔)
- **Has More Blocks?**: 추가 블록 유무 확인
- **Append Blocks to Page**: PATCH로 나머지 블록 추가

### 9. 에러 처리
에러 유형 자동 분류:
- 로그인 실패, CSRF 토큰 오류, Notion API 오류, Slack API 오류, 타임아웃, 네트워크 오류

## 기술적 주의사항

### 로그인 쿠키 처리
- 로그인 후 새로 발급된 세션 쿠키만 사용
- 이전 쿠키와 합치면 인증 실패 발생

### Notion API 제한
- children 블록 최대 100개 제한
- 100개 초과 시 페이지 생성 후 PATCH로 append 필요

### If 노드 조건
- `typeValidation: "loose"` 사용 (strict 시 타입 오류)

### 시간대 처리
- `Determine Base Date` 노드에서 KST 명시적 적용
- 서버 시간대에 관계없이 한국 시간 기준으로 동작

## 환경변수

```
REDMINE_BASE_URL    # Redmine 서버 URL
REDMINE_USERNAME    # Redmine 계정
REDMINE_PASSWORD    # Redmine 비밀번호
NOTION_DATABASE_ID  # Notion 데이터베이스 ID
```

## PM2 실행 (Windows)

### ecosystem.config.js 구조
```javascript
module.exports = {
  apps: [{
    name: 'n8n',
    script: 'cmd.exe',
    args: '/c n8n start',
    interpreter: 'none',
    env: {
      // n8n 기본 설정
      N8N_PORT: 5678,
      GENERIC_TIMEZONE: 'Asia/Seoul',
      TZ: 'Asia/Seoul',
      // Redmine/Notion/Slack 환경변수
      REDMINE_BASE_URL: '...',
      REDMINE_USERNAME: '...',
      REDMINE_PASSWORD: '...',
      NOTION_DATABASE_ID: '...',
      SLACK_CHANNEL: '#pms',
    }
  }]
};
```

### PM2 명령어
```bash
# 시작
pm2 start ecosystem.config.js

# 상태 확인
pm2 status

# 로그 확인
pm2 logs n8n

# 재시작
pm2 restart n8n

# 중지 및 삭제
pm2 delete n8n
```

## n8n Credentials 필요

- `notionApi` - Notion API 인증
- `slackApi` - Slack API 인증

## 수정 시 주의사항

1. **HTML 파싱 로직 수정 시**: Redmine 버전에 따라 HTML 구조가 다를 수 있음
2. **분류 로직 수정 시**: `Classify Issues` 노드의 조건문 확인
3. **Notion 블록 구조 수정 시**: 100개 제한 고려
4. **새 프로젝트 추가 시**: `Set Config` 노드의 PROJECTS 배열에 추가
5. **스케줄 변경 시**: `Schedule Morning`, `Schedule Friday PM` 노드 수정
