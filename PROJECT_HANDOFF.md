# n8n Redmine PMS 일일 리포트 자동화 프로젝트

## 프로젝트 개요

Redmine PMS에서 업무 목록을 크롤링하여 Notion에 일일 리포트를 생성하고 Slack으로 알림을 발송하는 n8n 워크플로우 개발 프로젝트.

---

## 1. 시스템 구성

### 1.1 대상 시스템

| 시스템 | URL / 정보 |
|--------|------------|
| Redmine | `http://idc.watchtek.co.kr:8188/redmine` |
| n8n 버전 | **1.122.5** (Self-hosted) |

### 1.2 인증 정보

**보안 주의**: 모든 인증 정보는 환경변수로 관리됩니다.

```bash
# n8n 환경변수 설정 필요
REDMINE_BASE_URL=http://idc.watchtek.co.kr:8188/redmine
REDMINE_USERNAME=<your_username>
REDMINE_PASSWORD=<your_password>
NOTION_DATABASE_ID=<your_database_id>  # 32자리 Notion 데이터베이스 ID

# n8n Credentials에서 별도 설정 필요
# - Notion API: API Key 설정
# - Slack API: Bot Token 설정

# Slack
# Channel: #pms (배열로 관리, 추후 확장 가능)
```

### 1.3 Notion Database ID 확인 방법

1. Notion에서 해당 데이터베이스 페이지 열기
2. URL 확인:
   ```
   https://www.notion.so/workspace/1234abcd5678efgh9012ijkl3456mnop?v=...
                                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                   이 부분이 Database ID
   ```

---

## 2. 크롤링 대상 프로젝트 (5개)

| 프로젝트명 | URL |
|-----------|-----|
| 결함 | `/projects/dcim_adv/issues?query_id=1477&per_page=100` |
| 신규기능개발 | `/projects/dcim_adv/issues?query_id=1478&per_page=100` |
| KT cloud | `/projects/dcim_adv/issues?query_id=1486&per_page=100` |
| 애플망고-사이트지원 | `/projects/applemango_site/issues?per_page=100&query_id=1087` |
| 애플망고-결함처리 | `/projects/release_issues_applemango/issues?query_id=1049&per_page=100` |

### 2.1 프로젝트별 컬럼 구조 (HTML 클래스 기준)

각 프로젝트마다 컬럼 구조가 다르므로, 파싱 로직이 이를 처리합니다.

| 클래스명 | 설명 | 사용 프로젝트 |
|---------|------|--------------|
| `subject` | 제목 | 전체 |
| `status` | 상태 | 전체 |
| `priority` | 우선순위 | 전체 |
| `assigned_to` | 담당자 | 전체 |
| `due_date` | 완료기한 | 전체 |
| `created_on` | 등록일 | 전체 |
| `cf_44` | 완료일 (커스텀필드) | 결함, 신규기능개발, KT cloud, 애플망고-결함처리 |
| `closed_on` | 완료일 | 애플망고-사이트지원 |
| `start_date` | 시작시간 | 애플망고-결함처리 |
| `done_ratio` / `progress-N` | 진척도 | 전체 |

---

## 3. 데이터 분류 기준

| 분류 | 조건 |
|------|------|
| **신규** | `등록일 = 오늘` |
| **처리 완료** | `완료일 = 오늘` |
| **처리 중** | `완료일 = 없음` AND `진척도 >= 10%` AND `상태 ≠ "처리 완료"` AND `상태 ≠ "완료"` |
| **잔여** | 위 3가지에 해당하지 않는 모든 건 |

---

## 4. 출력 포맷

### 4.1 Notion 페이지

**타이틀**: `[PMS 일일 리포트] 2025-12-07`

**속성**:
- `Name` (title): 페이지 제목
- `Date` (date): 보고서 날짜
- `Tags` (select): `일간`

**콘텐츠 포맷**:
```markdown
### 결함 (신규 3건, 처리 완료 5건, 처리중 2건, 잔여 42건)

> 신규
- [#101623](http://...) [보통] UPS 다이어그램 오류 확인의 건 (25/12/12)

> 처리 완료
- [#101500](http://...) [보통] API 응답 지연 수정 (25/12/05) (25/12/07) - 장성환

> 처리 중
- [#101490](http://...) [개선] 이벤트 필터 영역 height 반응형 처리 (25/12/15) - 이동엽

> 잔여
- [#101447](http://...) [보통] 장비 규격 메뉴 API 조회 실패 (-) - 미배정

---
```

**날짜 포맷**: 타이틀 제외 모든 날짜는 `YY/MM/DD` 형식

### 4.2 Slack 알림

```
📋 *PMS 일일 리포트* (2025-12-07)

• 결함: 신규 3건 / 완료 5건 / 잔여 42건
• 신규기능개발: 신규 1건 / 완료 2건 / 잔여 15건
• KT cloud: 신규 0건 / 완료 3건 / 잔여 28건
• 애플망고-사이트지원: 신규 2건 / 완료 1건 / 잔여 8건
• 애플망고-결함처리: 신규 0건 / 완료 4건 / 잔여 12건

🔗 <Notion URL|상세 보기>
```

---

## 5. 워크플로우 아키텍처

```
[Manual Trigger / Schedule Trigger]
    ↓
[Set Config] - Code 노드로 환경변수($env.XXX) 읽기
    ↓
[HTTP Request: Get Login Page] - GET /redmine/login (fullResponse: true)
    ↓
[Code: Extract CSRF Token] - HTML에서 authenticity_token 추출, 쿠키 추출
    ↓
[HTTP Request: Login to Redmine] - POST /redmine/login (neverError: true)
    ↓
[Code: Extract Session Cookies] - 세션 쿠키 병합
    ↓
[Code: Split Projects] - 5개 프로젝트를 개별 아이템으로 분리
    ↓
[HTTP Request: Get Issues Page] - 각 프로젝트 업무 목록 GET
    ↓
[Code: Parse Issues HTML] - 컬럼 기반 정규식 파싱 (프로젝트별 구조 대응)
    ↓
[Code: Classify Issues] - 신규/처리완료/처리중/잔여 분류
    ↓
[Aggregate] - 5개 프로젝트 데이터 병합 (destinationFieldName: "projectsData")
    ↓
[Code: Generate Report] - Notion 마크다운 + Slack 요약 생성
    ↓
[Notion: Create Page] - DB에 페이지 생성 (type: "paragraph")
    ↓
[Code: Split Slack Channels] - 채널 배열을 개별 아이템으로 분리
    ↓
[Slack: Send Message] - 각 채널에 알림 발송
```

---

## 6. 핵심 기술 구현

### 6.1 Redmine CSRF 토큰 처리

```javascript
// authenticity_token 추출
const tokenMatch = html.match(/name="authenticity_token"[^>]*value="([^"]+)"/);
const csrfToken = tokenMatch ? tokenMatch[1] : '';
```

### 6.2 세션 쿠키 관리

```javascript
// Set-Cookie 헤더에서 쿠키 추출 및 병합
const cookieArray = Array.isArray(headers['set-cookie'])
  ? headers['set-cookie']
  : [headers['set-cookie']];
const cookies = cookieArray.map(c => c.split(';')[0]).join('; ');
```

### 6.3 HTML 파싱 (컬럼 기반 추출)

Redmine 테이블의 `done_ratio` 셀에 중첩 테이블이 있어 행 기반 파싱이 불가능합니다.
따라서 **컬럼 기반 추출** 방식을 사용합니다:

```javascript
// 각 컬럼별로 모든 값 추출
const extractAll = (regex) => {
  const matches = [];
  let match;
  while ((match = regex.exec(html)) !== null) {
    matches.push(match[1] ? match[1].trim() : '');
  }
  return matches;
};

// 이슈 ID 추출
const issueIds = [];
const issueIdRegex = /<tr[^>]*id="issue-(\d+)"[^>]*>/gi;
let idMatch;
while ((idMatch = issueIdRegex.exec(html)) !== null) {
  issueIds.push(idMatch[1]);
}

// 각 컬럼 추출
const subjects = extractAll(/<td class="subject"><a[^>]*>([\s\S]*?)<\/a><\/td>/gi);
const statuses = extractAll(/<td class="status">([^<]*)<\/td>/gi);
const priorities = extractAll(/<td class="priority">([^<]*)<\/td>/gi);
const dueDates = extractAll(/<td class="due_date"[^>]*>([^<]*)<\/td>/gi);
const createdOns = extractAll(/<td class="created_on"[^>]*>([^<]*)<\/td>/gi);

// 완료일 - 프로젝트별 다른 클래스
const cf44s = extractAll(/<td class="cf_44[^"]*"[^>]*>([^<]*)<\/td>/gi);
const closedOns = extractAll(/<td class="closed_on"[^>]*>([^<]*)<\/td>/gi);

// 시작시간 (애플망고-결함처리)
const startDates = extractAll(/<td class="start_date"[^>]*>([^<]*)<\/td>/gi);

// 담당자 (빈 셀 또는 링크 처리)
const assignedTos = [];
const assignedToRegex = /<td class="assigned_to"[^>]*>([\s\S]*?)<\/td>/gi;
let assignedMatch;
while ((assignedMatch = assignedToRegex.exec(html)) !== null) {
  const content = assignedMatch[1];
  const linkMatch = content.match(/<a[^>]*>([^<]*)<\/a>/);
  assignedTos.push(linkMatch ? linkMatch[1].trim() : (content.trim() || '미배정'));
}

// 진척도
const progresses = extractAll(/class="progress progress-(\d+)"/gi);

// 인덱스로 매칭하여 이슈 객체 생성
for (let i = 0; i < issueIds.length; i++) {
  const completedDate = cf44s[i] || closedOns[i] || '';
  const issue = {
    '#': issueIds[i],
    '제목': subjects[i] || '',
    '상태': statuses[i] || '',
    '완료일': completedDate,
    // ...
  };
}
```

---

## 7. n8n 노드 설정 주의사항

### 7.1 Code 노드에서 환경변수 사용

Set 노드의 `{{ $env.XXX }}` 표현식은 jsonOutput 문자열 내에서 평가되지 않습니다.
**Code 노드**에서 `$env.XXX`를 직접 사용하세요:

```javascript
const REDMINE_BASE_URL = $env.REDMINE_BASE_URL || 'http://...';
const NOTION_DATABASE_ID = $env.NOTION_DATABASE_ID || '';
```

### 7.2 HTTP Request 302 리다이렉트 처리

로그인 성공 시 302 리다이렉트가 발생하는데, n8n에서 이를 오류로 처리합니다.
`neverError: true` 옵션을 추가하세요:

```json
{
  "options": {
    "response": {
      "response": {
        "fullResponse": true,
        "neverError": true
      }
    }
  }
}
```

### 7.3 Notion 블록 타입

`markdown` 타입은 지원되지 않습니다. `paragraph` 타입을 사용하세요:

```json
{
  "blockUi": {
    "blockValues": [
      {
        "type": "paragraph",
        "richText": false,
        "textContent": "={{ $json.notionContent }}"
      }
    ]
  }
}
```

### 7.4 Aggregate 노드 출력 구조

Aggregate 노드는 `{ projectsData: [...] }` 또는 `{ data: [...] }` 형태로 출력할 수 있습니다.
Generate Report 노드에서 다양한 구조를 처리하는 방어적 코드를 사용합니다.

---

## 8. 수정 완료 사항 (2025-12-07)

| 항목 | 상태 | 설명 |
|------|------|------|
| CSRF 토큰 추출 오류 | ✅ 완료 | fullResponse 구조 대응 (data vs body) |
| 환경변수 분리 | ✅ 완료 | Code 노드에서 $env.XXX 사용 |
| 302 리다이렉트 오류 | ✅ 완료 | neverError: true 옵션 추가 |
| HTML 파싱 오류 | ✅ 완료 | 컬럼 기반 추출 방식으로 변경 |
| 프로젝트별 컬럼 대응 | ✅ 완료 | cf_44, closed_on, start_date 처리 |
| Notion 블록 타입 | ✅ 완료 | markdown → paragraph |
| Aggregate 출력 구조 | ✅ 완료 | 다양한 구조 대응 로직 추가 |
| NOTION_DATASOURCE_ID 제거 | ✅ 완료 | 미사용 변수 정리 |

---

## 9. 환경변수 설정 방법

### Docker 사용 시

```bash
docker run -it --rm \
  -e REDMINE_BASE_URL="http://idc.watchtek.co.kr:8188/redmine" \
  -e REDMINE_USERNAME="your_username" \
  -e REDMINE_PASSWORD="your_password" \
  -e NOTION_DATABASE_ID="your_database_id" \
  n8nio/n8n
```

### docker-compose 사용 시

```yaml
services:
  n8n:
    environment:
      - REDMINE_BASE_URL=http://idc.watchtek.co.kr:8188/redmine
      - REDMINE_USERNAME=your_username
      - REDMINE_PASSWORD=your_password
      - NOTION_DATABASE_ID=your_database_id
```

---

## 10. 테스트 체크리스트

- [ ] n8n 환경변수 설정 완료
- [ ] Notion API Credential 설정
- [ ] Slack API Credential 설정
- [ ] Redmine 로그인 성공 확인
- [ ] 5개 프로젝트 이슈 목록 GET 성공
- [ ] HTML 파싱 정상 동작 (모든 프로젝트)
- [ ] 데이터 분류 로직 검증
- [ ] Notion 페이지 생성 확인
- [ ] Slack 메시지 발송 확인

---

## 11. 향후 작업

- [ ] 페이지네이션 처리 (100건 초과 프로젝트 대응)
- [ ] Schedule Trigger 설정 (운영 배포 시)
- [ ] 에러 핸들링 및 알림 추가

---

## 12. 프로젝트 파일 구조

```
watchtek-pms-cube/
├── redmine-pms-report-workflow.json  # n8n 워크플로우 (import용)
├── PROJECT_HANDOFF.md                # 이 문서
├── .env.example                      # 환경변수 템플릿
├── .gitignore                        # .env 제외
├── package.json                      # Playwright 테스트용
└── node_modules/                     # Playwright
```

---

## 13. 담당자 정보

- 사용자: Jeehaeng (허지행)
- 역할: 큐브파트 팀장, 프론트엔드 개발자
