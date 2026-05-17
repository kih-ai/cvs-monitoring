# 편의점 브랜드 모니터링 대시보드 — 주간 자동 갱신

## 개요
매주 월요일 오전 9시에 실행되는 주간 데이터 수집 및 Supabase 업데이트 작업.
Claude Code는 이 파일의 지시에 따라 크롤링 → new_posts.json 저장 → collect.py 실행까지 순서대로 처리한다.

---

## 실행 순서

1. 수집 기간 계산
2. 사이트별 크롤링 (브라우저 자동화)
3. new_posts.json 저장
4. collect.py 실행 → Supabase upsert

---

## Step 1: 수집 기간 계산

실행 당일 날짜 기준으로 자동 계산한다.

- `END_DATE` = 오늘 날짜 (예: 2026-05-19)
- `START_DATE` = 오늘 기준 6일 전 (예: 2026-05-13)
- 형식: `YYYY-MM-DD`

```python
from datetime import date, timedelta
end   = date.today()
start = end - timedelta(days=6)
```

---

## Step 2: 크롤링 대상

### 수집 키워드

| 키워드 | brand |
|--------|-------|
| CU     | cu    |
| 씨유   | cu    |
| GS25   | gs    |

> 포켓CU, 포켓씨유, 우동지, 우리동네GS는 해당 기간 내 게시글이 없는 경우가 많으므로
> 검색 후 결과가 있을 때만 수집한다.

### 수집 사이트 및 방법

#### 1. 더쿠 (theqoo.net)
- 로그인 상태에서 진행 (브라우저 세션 활용)
- 검색 URL 패턴:
  - talk: `https://theqoo.net/?_filter=search&act=&mid=talk&search_target=title_content&search_keyword={키워드}`
  - theqdeal: `https://theqoo.net/?_filter=search&act=&mid=theqdeal&search_target=title_content&search_keyword={키워드}`
  - square: `https://theqoo.net/?_filter=search&act=&mid=square&search_target=title_content&search_keyword={키워드}`
- 수집 항목: 제목, URL, 날짜
- 날짜 필터: START_DATE ~ END_DATE 범위 내 게시글만
- 개별 게시글 방문하여 본문(article 태그), 조회수, 댓글수 수집

#### 2. 에펨코리아 (fmkorea.com)
- 검색 URL 패턴:
  `https://www.fmkorea.com/search.php?mid=home&act=IS&search_target=title_content&is_keyword={키워드}&where=document&page=1&sph_sort=recentness`
- 편의점 관련 게시글만 수집 (게임/스포츠 관련 CU 언급 글 제외)
- 날짜 필터: START_DATE ~ END_DATE 범위 내 게시글만
- 개별 게시글 방문하여 본문, 조회수(`조회 수` 텍스트 파싱), 댓글수(`.fdb_lst_ul > li` 카운트) 수집

#### 3. 인스티즈 (instiz.net)
- 일상 게시판 검색 URL 패턴:
  `https://www.instiz.net/bbs/list.php?k={키워드}&id=name&stype=9&category=1`
- 로그인 상태에서 진행 (브라우저 세션 활용)
- 날짜 필터: START_DATE ~ END_DATE 범위 내 게시글만
- 개별 게시글 방문하여 본문(article 태그), 조회수(`조회` 텍스트 파싱), 댓글수 수집

---

## Step 3: new_posts.json 저장

수집된 데이터를 아래 형식으로 `new_posts.json`에 저장한다.

### 필드 정의

| 필드 | 타입 | 설명 | 필수 |
|------|------|------|------|
| brand | string | `"cu"` 또는 `"gs"` | ✅ |
| title | string | 게시글 제목 | ✅ |
| site | string | `"더쿠"`, `"에펨코리아"`, `"인스티즈"` | ✅ |
| keyword | string | 검색 키워드 (`"CU"`, `"씨유"`, `"GS25"` 등) | ✅ |
| post_date | string | 게시글 작성일 (`YYYY-MM-DD`) | ✅ |
| url | string | 게시글 원문 URL (고유값) | ✅ |
| content | string | 본문 앞 200자 | - |
| views | integer | 조회수 (없으면 null) | - |
| comments | integer | 댓글수 (없으면 null) | - |

### 예시

```json
[
  {
    "brand": "cu",
    "title": "씨유 영양제 할인 행사 또 시작됐다",
    "site": "더쿠",
    "keyword": "씨유",
    "post_date": "2026-05-19",
    "url": "https://theqoo.net/talk/4201234567",
    "content": "이번엔 10개 이상 사면 더 할인해준대",
    "views": 142,
    "comments": 5
  },
  {
    "brand": "gs",
    "title": "GS25 신상 도시락 먹어봤어",
    "site": "인스티즈",
    "keyword": "GS25",
    "post_date": "2026-05-18",
    "url": "https://www.instiz.net/name/66500000",
    "content": "진짜 맛있다 강추",
    "views": 88,
    "comments": 0
  }
]
```

### 주의사항
- url이 없는 게시글은 수집에서 제외한다
- 중복 게시글은 (brand, url) 복합 기준으로 판단하며, 같은 조합이 있으면 한 건만 포함한다
  (Supabase UNIQUE 제약: brand + url)
- 날짜 범위(START_DATE ~ END_DATE) 외 게시글은 포함하지 않는다
- 편의점과 무관한 게시글(게임 용어 CU, 인명 씨유 등)은 제외한다

---

## Step 4: collect.py 실행

```bash
cd dashboard_project
python collect.py
```

실행 완료 후 출력 확인:
- `✅ 총 N건 Supabase 적재 완료` 메시지가 나오면 성공
- `backups/` 폴더에 날짜별 백업 파일 생성 확인

---

## 폴더 구조

```
dashboard_project/
├── index.html          # 대시보드 (GitHub Pages 배포)
├── CLAUDE.md           # 이 파일 (Claude Code 지시서)
├── init_upload.py      # 초기 적재용 (1회성)
├── collect.py          # 주간 갱신용
├── new_posts.json      # Claude Code가 생성 (매주 덮어쓰기)
├── requirements.txt    # Python 의존성
├── .env                # Supabase 키 (git 제외)
└── backups/            # collect.py 실행 후 자동 생성
    └── posts_YYYYMMDD_HHMMSS.json
```

---

## 오류 대응

| 상황 | 조치 |
|------|------|
| 더쿠 로그인 세션 만료 | 브라우저에서 직접 로그인 후 재시도 |
| 인스티즈 로그인 세션 만료 | 브라우저에서 직접 로그인 후 재시도 |
| new_posts.json 유효성 오류 | 오류 메시지 확인 후 해당 게시글 데이터 수정 |
| Supabase 연결 오류 | .env 파일의 SUPABASE_URL, SUPABASE_SERVICE_KEY 확인 |
| collect.py UNIQUE 충돌 | 이미 적재된 데이터 — upsert이므로 무시해도 됨 |

---

## 환경 변수 (.env)

```
SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
SUPABASE_SERVICE_KEY=eyJ...
```

> `.env` 파일은 절대 GitHub에 커밋하지 않는다.
