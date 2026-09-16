# 1주일 Supabase + PostgreSQL 학습 플랜
> 독서기록 AI 추천 웹 개발과 병행하는 실전 중심 학습

- 스택: Node.js + Express / Supabase / HTML·CSS·JS / 카카오 도서 API + Claude API
- 배포: Render
- 하루 투자 시간: 1~3시간
- 사전 지식: MySQL / Oracle 실무 2년

---

## 학습 원칙

- 이론만 보지 않고 **그날 배운 내용을 프로젝트에 바로 적용**합니다.
- MySQL/Oracle과 다른 점 위주로 빠르게 훑습니다.
- Supabase SQL Editor를 적극 활용합니다 (설치 없이 바로 실습).

---

## 전체 흐름

```
1일차  Supabase 프로젝트 세팅 + DB 설계
2일차  PostgreSQL 기본 문법 차이 + 테이블 생성
3일차  RLS 설정 + Auth 연동
4일차  PostgreSQL 고급 문법 (JSON, 윈도우 함수)
5일차  Storage + 카카오 도서 API 연동
6일차  Claude API + RPC(함수) 연동
7일차  전체 복습 + 보완
```

---

## 1일차 — Supabase 프로젝트 세팅 + DB 설계

### 목표
- Supabase 프로젝트 생성
- 독서기록 앱의 테이블 설계

### 학습 내용

**Supabase 구조 복습**
```
브라우저 (Publishable Key)
      ↓
  API Gateway
      ↓
Auth / Database / Storage
      ↓
RLS (행 단위 접근 제어)
```

**키 종류**
| 키 | 위치 | 용도 |
|---|---|---|
| `sb_publishable_...` | 프론트엔드 | 공개 가능 |
| `sb_secret_...` | server.js (env) | 절대 노출 금지 |

### 실습 — 테이블 설계 및 생성

Supabase SQL Editor에서 실행합니다.

```sql
-- 책 테이블
CREATE TABLE books (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id     UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  isbn        TEXT,
  title       TEXT NOT NULL,
  author      TEXT,
  publisher   TEXT,
  cover_url   TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 독서 기록 테이블
CREATE TABLE reading_records (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id     UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id     BIGINT REFERENCES books(id) ON DELETE CASCADE,
  status      TEXT CHECK (status IN ('reading', 'done', 'want')),
  rating      INT CHECK (rating BETWEEN 1 AND 5),
  memo        TEXT,
  started_at  DATE,
  finished_at DATE,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- AI 추천 기록 테이블
CREATE TABLE ai_recommendations (
  id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id     UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  prompt      TEXT,
  result      TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

> **MySQL과 다른 점**
> - `AUTO_INCREMENT` → `GENERATED ALWAYS AS IDENTITY`
> - `DATETIME` → `TIMESTAMPTZ` (타임존 포함)
> - `VARCHAR(255)` → `TEXT` (PostgreSQL은 TEXT가 성능 차이 없음)

### 체크리스트
- [ ] Supabase 프로젝트 생성
- [ ] `.env`에 URL, Publishable Key 저장
- [ ] 테이블 3개 생성 확인

---

## 2일차 — PostgreSQL 문법 차이 + 데이터 조작

### 목표
- MySQL/Oracle과 다른 문법 파악
- 기본 CRUD 실습

### MySQL → PostgreSQL 문법 대조표

| 기능 | MySQL | PostgreSQL |
|---|---|---|
| 자동 증가 | `AUTO_INCREMENT` | `GENERATED ALWAYS AS IDENTITY` |
| 현재 시간 | `NOW()` | `NOW()` (동일) |
| 날짜 포맷 | `DATE_FORMAT(d, '%Y-%m-%d')` | `TO_CHAR(d, 'YYYY-MM-DD')` |
| NULL 대체 | `IFNULL(a, b)` | `COALESCE(a, b)` |
| 문자열 합치기 | `CONCAT(a, b)` | `a \|\| b` 또는 `CONCAT(a, b)` |
| 페이징 | `LIMIT 10 OFFSET 20` | `LIMIT 10 OFFSET 20` (동일) |
| UPSERT | `INSERT ... ON DUPLICATE KEY UPDATE` | `INSERT ... ON CONFLICT DO UPDATE` |

### 실습 — CRUD 쿼리 작성

```sql
-- INSERT
INSERT INTO books (user_id, isbn, title, author, publisher)
VALUES (
  '유저UUID',
  '9791162540',
  '아몬드',
  '손원평',
  '창비'
);

-- SELECT (최근 읽은 책)
SELECT
  b.title,
  b.author,
  r.status,
  r.rating,
  r.finished_at,
  TO_CHAR(r.finished_at, 'YYYY.MM.DD') AS finished_str
FROM reading_records r
JOIN books b ON b.id = r.book_id
WHERE r.user_id = '유저UUID'
  AND r.status = 'done'
ORDER BY r.finished_at DESC
LIMIT 10;

-- UPDATE
UPDATE reading_records
SET status = 'done', finished_at = CURRENT_DATE, rating = 5
WHERE id = 1;

-- UPSERT (있으면 업데이트, 없으면 삽입)
INSERT INTO books (user_id, isbn, title, author)
VALUES ('유저UUID', '9791162540', '아몬드', '손원평')
ON CONFLICT (isbn)
DO UPDATE SET
  title = EXCLUDED.title,
  author = EXCLUDED.author;
```

### JS에서 Supabase로 동일 작업

```js
// SELECT
const { data } = await supabase
  .from('reading_records')
  .select(`*, books(title, author, cover_url)`)
  .eq('status', 'done')
  .order('finished_at', { ascending: false })
  .limit(10)

// INSERT
const { data, error } = await supabase
  .from('books')
  .insert({ user_id: userId, isbn, title, author })
  .select()
  .single()

// UPDATE
const { error } = await supabase
  .from('reading_records')
  .update({ status: 'done', rating: 5 })
  .eq('id', recordId)
```

### 체크리스트
- [ ] SQL Editor에서 CRUD 쿼리 직접 실행
- [ ] JS에서 supabase 클라이언트로 동일 작업 확인

---

## 3일차 — RLS 설정 + Auth 연동

### 목표
- 테이블마다 RLS 정책 적용
- 로그인/회원가입 구현

### RLS 핵심 개념

```
RLS를 켜면 → 기본적으로 모든 접근 차단
Policy를 추가해야 → 허용
```

```sql
-- 3개 테이블 모두 RLS 활성화
ALTER TABLE books ENABLE ROW LEVEL SECURITY;
ALTER TABLE reading_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_recommendations ENABLE ROW LEVEL SECURITY;

-- books 정책
CREATE POLICY "본인 책만 조회" ON books
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "본인 책만 등록" ON books
  FOR INSERT TO authenticated
  WITH CHECK (auth.uid() = user_id);

CREATE POLICY "본인 책만 수정" ON books
  FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "본인 책만 삭제" ON books
  FOR DELETE USING (auth.uid() = user_id);

-- reading_records, ai_recommendations도 동일하게 적용
-- (books 정책에서 테이블명만 바꿔서 실행)
```

> **USING vs WITH CHECK**
> - `USING` — 기존 행 접근 시 (SELECT, UPDATE, DELETE)
> - `WITH CHECK` — 새로 쓸 행 검증 시 (INSERT, UPDATE)

### Auth 연동 (JS)

```js
// 회원가입
const { data, error } = await supabase.auth.signUp({
  email: 'user@example.com',
  password: 'password123'
})

// 로그인
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password123'
})

// 세션 확인
const { data: { session } } = await supabase.auth.getSession()
const userId = session?.user?.id

// 로그아웃
await supabase.auth.signOut()
```

### 체크리스트
- [ ] 3개 테이블 RLS 활성화
- [ ] 각 테이블 Policy 4개씩 (SELECT/INSERT/UPDATE/DELETE) 생성
- [ ] 로그인/회원가입 페이지 구현
- [ ] RLS 동작 확인 (다른 유저 데이터 조회 안 되는지)

---

## 4일차 — PostgreSQL 고급 문법

### 목표
- JSONB, 윈도우 함수, CTE 실습
- AI 추천에 활용할 쿼리 작성

### JSONB — 카카오 API 원본 데이터 저장

카카오 도서 API 응답을 통째로 저장할 때 유용합니다.

```sql
-- books 테이블에 JSONB 컬럼 추가
ALTER TABLE books ADD COLUMN kakao_data JSONB;

-- 데이터 삽입
UPDATE books
SET kakao_data = '{"price": 14000, "sale_price": 12600, "category": "소설"}'
WHERE id = 1;

-- JSONB 조회
SELECT
  title,
  kakao_data->>'category' AS category,
  (kakao_data->>'price')::INT AS price
FROM books
WHERE kakao_data->>'category' = '소설';
```

### 윈도우 함수 — 독서 통계

```sql
-- 월별 독서량 + 누적 독서량
SELECT
  TO_CHAR(finished_at, 'YYYY-MM') AS month,
  COUNT(*) AS monthly_count,
  SUM(COUNT(*)) OVER (ORDER BY TO_CHAR(finished_at, 'YYYY-MM')) AS cumulative_count
FROM reading_records
WHERE status = 'done'
  AND user_id = '유저UUID'
GROUP BY 1
ORDER BY 1;

-- 별점별 순위
SELECT
  b.title,
  r.rating,
  RANK() OVER (ORDER BY r.rating DESC) AS rank
FROM reading_records r
JOIN books b ON b.id = r.book_id
WHERE r.user_id = '유저UUID';
```

### CTE — AI 추천용 최근 독서 이력 조회

```sql
-- Claude API에 보낼 독서 이력 데이터 준비
WITH recent_books AS (
  SELECT
    b.title,
    b.author,
    r.rating,
    r.memo
  FROM reading_records r
  JOIN books b ON b.id = r.book_id
  WHERE r.user_id = '유저UUID'
    AND r.status = 'done'
  ORDER BY r.finished_at DESC
  LIMIT 5
),
high_rated AS (
  SELECT title, author
  FROM recent_books
  WHERE rating >= 4
)
SELECT * FROM high_rated;
```

### 체크리스트
- [ ] books 테이블에 kakao_data JSONB 컬럼 추가
- [ ] 월별 독서량 통계 쿼리 작성
- [ ] AI 추천용 독서 이력 조회 쿼리 작성

---

## 5일차 — Storage + 카카오 도서 API 연동

### 목표
- 책 커버 이미지 Storage 저장 (선택)
- 카카오 도서 API로 책 검색 구현

### Storage 설정

```sql
-- Storage RLS (post-images 버킷과 동일한 방식)
CREATE POLICY "본인만 업로드" ON storage.objects
  FOR INSERT TO authenticated
  WITH CHECK (bucket_id = 'book-covers');

CREATE POLICY "누구나 읽기" ON storage.objects
  FOR SELECT TO public
  USING (bucket_id = 'book-covers');
```

```js
// 책 커버 이미지 업로드
const { data, error } = await supabase.storage
  .from('book-covers')
  .upload(`${userId}/${isbn}.jpg`, imageBlob)

const { data: { publicUrl } } = supabase.storage
  .from('book-covers')
  .getPublicUrl(`${userId}/${isbn}.jpg`)
```

### 카카오 도서 API 연동 (server.js)

```js
// server.js
app.get('/api/books/search', async (req, res) => {
  const { query } = req.query
  if (!query) return res.status(400).json({ message: '검색어를 입력하세요.' })

  const response = await fetch(
    `https://dapi.kakao.com/v3/search/book?query=${encodeURIComponent(query)}&size=10`,
    {
      headers: { Authorization: `KakaoAK ${process.env.KAKAO_API_KEY}` }
    }
  )

  const data = await response.json()
  return res.json(data)
})
```

```js
// 프론트엔드
async function searchBooks(keyword) {
  const res = await fetch(`/api/books/search?query=${encodeURIComponent(keyword)}`)
  const data = await res.json()

  // data.documents 배열에 책 목록
  data.documents.forEach(book => {
    console.log(book.title, book.authors, book.thumbnail)
  })
}
```

### 체크리스트
- [ ] book-covers 버킷 생성 + RLS 설정
- [ ] 카카오 개발자 콘솔에서 API 키 발급
- [ ] `.env`에 `KAKAO_API_KEY` 추가
- [ ] 책 검색 API 엔드포인트 구현
- [ ] 프론트에서 검색 결과 렌더링

---

## 6일차 — Claude API + RPC(함수) 연동

### 목표
- Claude API로 AI 책 추천 구현
- Supabase RPC(PostgreSQL 함수)로 복잡한 로직 처리

### Claude API 연동 (server.js)

```js
// server.js
app.post('/api/recommend', async (req, res) => {
  const { userId } = req.body

  // 1. Supabase에서 독서 이력 조회 (Secret Key 사용)
  const supabaseAdmin = createClient(
    process.env.SUPABASE_URL,
    process.env.SUPABASE_SERVICE_KEY
  )

  const { data: records } = await supabaseAdmin
    .from('reading_records')
    .select(`*, books(title, author, kakao_data)`)
    .eq('user_id', userId)
    .eq('status', 'done')
    .order('finished_at', { ascending: false })
    .limit(5)

  // 2. Claude API 호출
  const bookList = records.map(r =>
    `- ${r.books.title} (${r.books.author}) 별점: ${r.rating}/5`
  ).join('\n')

  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': process.env.CLAUDE_API_KEY,
      'anthropic-version': '2023-06-01'
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1000,
      messages: [{
        role: 'user',
        content: `다음은 사용자가 최근 읽은 책 목록입니다:\n${bookList}\n\n이 독서 취향을 바탕으로 다음에 읽을 책 3권을 추천해주세요. 추천 이유도 함께 설명해주세요.`
      }]
    })
  })

  const result = await response.json()
  const recommendation = result.content[0].text

  // 3. 추천 결과 DB 저장
  await supabaseAdmin
    .from('ai_recommendations')
    .insert({ user_id: userId, prompt: bookList, result: recommendation })

  return res.json({ recommendation })
})
```

### Supabase RPC — PostgreSQL 함수

복잡한 쿼리를 DB 함수로 만들어두고 JS에서 간단하게 호출합니다.

```sql
-- 독서 통계 함수 생성
CREATE OR REPLACE FUNCTION get_reading_stats(p_user_id UUID)
RETURNS JSON AS $$
DECLARE
  result JSON;
BEGIN
  SELECT JSON_BUILD_OBJECT(
    'total',     COUNT(*),
    'done',      COUNT(*) FILTER (WHERE status = 'done'),
    'reading',   COUNT(*) FILTER (WHERE status = 'reading'),
    'want',      COUNT(*) FILTER (WHERE status = 'want'),
    'avg_rating', ROUND(AVG(rating)::NUMERIC, 1)
  ) INTO result
  FROM reading_records
  WHERE user_id = p_user_id;

  RETURN result;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

```js
// JS에서 RPC 호출
const { data } = await supabase.rpc('get_reading_stats', {
  p_user_id: userId
})

console.log(data)
// { total: 24, done: 18, reading: 2, want: 4, avg_rating: 4.2 }
```

### 체크리스트
- [ ] `.env`에 `CLAUDE_API_KEY` 추가
- [ ] AI 추천 API 엔드포인트 구현
- [ ] get_reading_stats RPC 함수 생성
- [ ] 프론트에서 추천 결과 렌더링

---

## 7일차 — 전체 복습 + 보완

### 목표
- 1~6일차 내용 점검
- 빠진 기능 보완
- Render 배포

### 복습 체크리스트

**Supabase**
- [ ] Publishable Key는 프론트, Secret Key는 서버에만 사용
- [ ] 모든 테이블에 RLS 활성화 확인
- [ ] Policy가 SELECT/INSERT/UPDATE/DELETE 모두 있는지 확인
- [ ] Storage 버킷 Public/Private 설정 확인

**PostgreSQL**
- [ ] GENERATED ALWAYS AS IDENTITY (AUTO_INCREMENT 대체)
- [ ] TIMESTAMPTZ (타임존 포함 날짜)
- [ ] COALESCE (IFNULL 대체)
- [ ] ON CONFLICT DO UPDATE (UPSERT)
- [ ] JSONB 컬럼 조회 (`->`, `->>`)
- [ ] 윈도우 함수 (ROW_NUMBER, RANK, SUM OVER)

**보안 점검**
```
✓ .env가 .gitignore에 포함되어 있는가?
✓ Secret Key가 프론트 코드에 노출되지 않는가?
✓ 모든 테이블에 RLS가 켜져 있는가?
✓ CORS 설정이 되어 있는가?
```

### Render 배포

```
1. GitHub에 코드 push
2. render.com 접속 → New Web Service
3. GitHub 저장소 연결
4. Environment Variables에 .env 내용 입력
   - SUPABASE_URL
   - SUPABASE_ANON_KEY
   - SUPABASE_SERVICE_KEY
   - KAKAO_API_KEY
   - CLAUDE_API_KEY
5. Deploy
```

### 이번 주 배운 것 요약

| 일차 | 핵심 개념 |
|---|---|
| 1일차 | Supabase 구조, 테이블 설계, IDENTITY/TIMESTAMPTZ |
| 2일차 | MySQL↔PostgreSQL 문법 차이, UPSERT, JS CRUD |
| 3일차 | RLS 정책, Auth 연동, USING vs WITH CHECK |
| 4일차 | JSONB, 윈도우 함수, CTE |
| 5일차 | Storage, 카카오 도서 API |
| 6일차 | Claude API, RPC 함수 |
| 7일차 | 전체 점검, Render 배포 |

---

## 참고 자료

| 자료 | 링크 |
|---|---|
| Supabase 공식 문서 | https://supabase.com/docs |
| PostgreSQL 공식 문서 | https://www.postgresql.org/docs |
| 카카오 도서 API | https://developers.kakao.com/docs/latest/ko/daum-search/dev-guide#search-book |
| Claude API 문서 | https://docs.anthropic.com |
| Render 배포 가이드 | https://render.com/docs |
