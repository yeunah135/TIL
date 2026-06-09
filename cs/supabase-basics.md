# Supabase 기본 상식

## 목차
1. [전체 구조](#1-전체-구조)
2. [Auth — 로그인/인증](#2-auth--로그인인증)
3. [Database — PostgreSQL](#3-database--postgresql)
4. [RLS (Row Level Security)](#4-rls-row-level-security)
5. [API Keys](#5-api-keys)
6. [Storage](#6-storage)

---

## 1. 전체 구조

```
브라우저 / 앱                서버 / Edge Function
(Publishable Key)           (Secret Key)
        ↓                           ↓
        ────────── API Gateway ──────────
                  REST · GraphQL · Realtime
                          ↓
         ┌────────────────┼────────────────┐
        Auth           Database          Storage
    (로그인/세션)      (PostgreSQL)     (파일 저장)
                          ↓
                  RLS (Row Level Security)
                  auth.uid() 기반 행 단위 접근 제어
```

---

## 2. Auth — 로그인/인증

Supabase가 `auth.users` 테이블을 자동으로 관리합니다.  
로그인이 성공하면 **JWT 토큰**이 발급되고, 이 토큰에 담긴 `auth.uid()`가 RLS 정책에서 "지금 요청한 사람이 누구인지" 확인하는 핵심 값입니다.

```js
// 세션 확인
const { data: { session } } = await supabase.auth.getSession()
const userId = session.user.id  // = auth.uid()

// 로그아웃
await supabase.auth.signOut()
```

### 지원 로그인 방식
- 이메일 / 비밀번호
- OAuth (Google, GitHub, Kakao 등)
- Magic Link (이메일 링크)
- 전화번호 OTP

---

## 3. Database — PostgreSQL

일반 PostgreSQL이라 SQL을 그대로 사용합니다.  
Supabase가 REST API를 자동 생성해주기 때문에 JS에서 SQL 없이도 데이터를 조회할 수 있습니다.

```js
// SELECT * FROM posts WHERE category = '여행 리뷰' ORDER BY created_at DESC
const { data, error } = await supabase
  .from('posts')
  .select('*')
  .eq('category', '여행 리뷰')
  .order('created_at', { ascending: false })

// INSERT
const { data, error } = await supabase
  .from('posts')
  .insert({ title: '제목', content: '내용', user_id: userId })
  .select()
  .single()

// DELETE
const { error } = await supabase
  .from('posts')
  .delete()
  .eq('id', postId)
```

---

## 4. RLS (Row Level Security)

> **가장 중요한 개념입니다.**

테이블에 RLS를 켜면 기본적으로 **모든 접근이 차단**됩니다.  
허용할 행동을 Policy로 하나씩 열어줘야 합니다.

```sql
-- RLS 활성화
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- 누구나 읽기 허용
CREATE POLICY "public read" ON posts
  FOR SELECT USING (true);

-- 로그인한 유저만 INSERT 가능 (user_id 자동 검증)
CREATE POLICY "auth insert" ON posts
  FOR INSERT TO authenticated
  WITH CHECK (user_id = auth.uid());

-- 본인 글만 삭제 가능
CREATE POLICY "owner delete" ON posts
  FOR DELETE USING (auth.uid() = user_id);
```

### RLS 없이 Publishable Key를 쓰면?
DB 전체가 외부에 열리므로 **RLS는 필수**입니다.

### Policy 종류
| 명령 | 설명 |
|---|---|
| `FOR SELECT` | 조회 |
| `FOR INSERT` | 삽입 |
| `FOR UPDATE` | 수정 |
| `FOR DELETE` | 삭제 |
| `FOR ALL` | 위 네 가지 전부 |

### USING vs WITH CHECK
| 구문 | 적용 시점 | 주로 사용 |
|---|---|---|
| `USING` | 기존 행에 대한 접근 | SELECT, UPDATE, DELETE |
| `WITH CHECK` | 새로 쓰려는 행 검증 | INSERT, UPDATE |

---

## 5. API Keys

### 키 종류

| 키 | 형태 | 용도 | 노출 |
|---|---|---|---|
| **Publishable Key** | `sb_publishable_...` | 프론트엔드, 모바일 앱 | 공개 가능 |
| **Secret Key** | `sb_secret_...` | 서버, Edge Function | 절대 노출 금지 |
| ~~anon key~~ | `eyJ...` (JWT) | 레거시 → Publishable로 교체 | 공개 가능 |
| ~~service_role key~~ | `eyJ...` (JWT) | 레거시 → Secret으로 교체 | 절대 노출 금지 |

> 2025년 11월 이후 생성된 프로젝트는 anon/service_role 키를 기본 제공하지 않습니다.  
> 레거시 키는 2026년 말에 완전히 삭제될 예정입니다.

### Publishable Key가 공개돼도 괜찮은 이유
- 낮은 권한만 가집니다 (anon role)
- 실제 데이터 접근은 RLS가 제어합니다
- **RLS가 없으면 의미 없습니다** — RLS는 반드시 설정해야 합니다

### 프로젝트에서 사용하는 위치

```js
// community.js — 프론트엔드 (Publishable Key 사용)
supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

// server.js — 백엔드 (Secret Key 사용 권장)
// process.env.SUPABASE_SERVICE_KEY = "sb_secret_..."
```

### default vs project 키
대시보드에서 보이는 두 Publishable Key는 **같은 종류**이며 이름(레이블)만 다릅니다.
- `default` — 새 키 시스템 도입 시 자동 생성된 키
- `project` (또는 직접 지은 이름) — 직접 추가로 만든 키

둘 중 하나만 골라서 사용하면 됩니다.

---

## 6. Storage

파일(이미지, 영상 등)을 저장하는 공간입니다.  
**버킷 → 폴더 → 파일** 구조로 관리됩니다.

### 버킷 종류

| 타입 | 특징 | 사용 예 |
|---|---|---|
| **Public** | URL만 알면 누구나 접근 가능 | 게시글 이미지, 썸네일 |
| **Private** | 인증된 사용자만 접근 가능 | 개인 파일, 계약서 |

### Storage RLS 정책

Storage도 `storage.objects` 테이블에 RLS 정책이 필요합니다.  
버킷을 만들어도 정책이 없으면 업로드가 차단됩니다.

```sql
-- 로그인한 유저만 업로드 가능
CREATE POLICY "auth upload" ON storage.objects
  FOR INSERT TO authenticated
  WITH CHECK (bucket_id = 'post-images');

-- 누구나 이미지 URL로 읽기 가능 (Public 버킷)
CREATE POLICY "public read" ON storage.objects
  FOR SELECT TO public
  USING (bucket_id = 'post-images');

-- 본인이 올린 파일만 삭제 가능
CREATE POLICY "owner delete" ON storage.objects
  FOR DELETE TO authenticated
  USING (bucket_id = 'post-images' AND owner = auth.uid());
```

### JS에서 사용하는 방법

```js
// 업로드
const { error } = await supabase.storage
  .from('post-images')
  .upload(`${userId}/${Date.now()}.webp`, file, {
    cacheControl: '3600',
    upsert: false
  })

// 공개 URL 가져오기
const { data } = supabase.storage
  .from('post-images')
  .getPublicUrl(path)

const imageUrl = data.publicUrl
// https://<project>.supabase.co/storage/v1/object/public/post-images/<path>

// 파일 삭제
const { error } = await supabase.storage
  .from('post-images')
  .remove([path])
```

### 이미지 업로드 흐름 (이번 프로젝트 기준)

```
파일 선택
  → 유효성 검사 (5MB 이하, JPG/PNG/GIF/WEBP)
  → FileReader로 미리보기 (로컬, 서버 전송 없음)
  → 등록 버튼 클릭 시 Storage 업로드
  → 공개 URL 반환
  → posts 테이블의 image_url 컬럼에 저장
  → 렌더링 시 <img src="..."> 에 표시
```

---

## 요약

| 개념 | 한 줄 정리 |
|---|---|
| Auth | 로그인 처리 + JWT 발급. `auth.uid()`가 RLS의 핵심 |
| Database | PostgreSQL. JS에서 `.from().select()` 등으로 조작 |
| RLS | 행 단위 접근 제어. 켜면 기본 차단, Policy로 허용 |
| Publishable Key | 프론트용. 공개돼도 RLS가 방어 |
| Secret Key | 서버용. 절대 노출 금지 |
| Storage | 파일 저장. 버킷도 RLS 정책 필요 |
