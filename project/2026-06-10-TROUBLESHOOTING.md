# 트러블슈팅 - 지도 페이지 개발 (2026-06-10)

## 1. `Invalid supabaseUrl` 오류

**증상**
```
Uncaught Error: Invalid supabaseUrl: Must be a valid HTTP or HTTPS URL.
```

**원인**
`map_api.js`에 Supabase URL이 `'YOUR_SUPABASE_URL'` 플레이스홀더 그대로였음

**해결**
`server.js`의 `/config.js` 엔드포인트에서 환경변수를 `window`에 주입하는 방식으로 변경

```js
// map_api.js
const SUPABASE_URL = window.SUPABASE_URL;
const SUPABASE_ANON_KEY = window.SUPABASE_ANON_KEY;
```

---

## 2. `/rest/v1/rest/v1/` 경로 중복 404 오류

**증상**
```
GET https://xxxx.supabase.co/rest/v1/rest/v1/festivals 404 (Not Found)
```

**원인**
`.env`의 `SUPABASE_URL` 값에 `/rest/v1`이 포함되어 있어 경로가 중복됨

**해결**
URL을 도메인만 남기도록 수정

```
# 잘못된 경우
SUPABASE_URL=https://xxxx.supabase.co/rest/v1

# 올바른 경우
SUPABASE_URL=https://xxxx.supabase.co
```

---

## 3. `kakao is not defined` 오류

**증상**
```
Uncaught ReferenceError: kakao is not defined
```

**원인**
카카오맵 SDK가 로드되기 전에 `map_main.js`가 실행됨

**해결**
`map.html`에서 카카오맵 스크립트를 동적으로 생성하고 `onload` 콜백 안에서 `map_main.js`를 로드하는 방식으로 변경

```html
<script>
  const script = document.createElement('script');
  script.src = `https://dapi.kakao.com/v2/maps/sdk.js?appkey=${KAKAO_MAP_KEY}&autoload=false&libraries=clusterer`;
  script.onload = () => {
    import('./js/map_main.js');
  };
  document.head.appendChild(script);
</script>
```

---

## 4. `kakao.maps.LatLng is not a constructor` 오류

**증상**
```
Uncaught TypeError: kakao.maps.LatLng is not a constructor
```

**원인**
`autoload` 없이 동적 로드 시 카카오맵 내부 초기화가 완료되기 전에 실행됨

**해결**
SDK URL에 `autoload=false` 추가 후 `kakao.maps.load()` 콜백 안에서 초기화

```js
// map_main.js
kakao.maps.load(() => {
  initMap();
  loadFestivals();
});
```

---

## 5. `window.KAKAO_MAP_KEY undefined`

**증상**
카카오맵이 로드되지 않거나 appkey가 빈 값으로 전달됨

**원인**
`/config.js` 엔드포인트가 빈 응답을 반환함 → 서버가 `.env` 파일을 읽지 못함

**해결**
`.env` 파일 값 형식 확인 후 서버 재시작

```
# 잘못된 경우 (따옴표, 공백 주의)
KAKAO_MAP_KEY = "abc123"

# 올바른 경우
KAKAO_MAP_KEY=abc123
```

---

## 6. `Cannot find table 'public.festivals'`

**증상**
```
relation "public.festivals" does not exist
```

**원인**
`map_supabase_setup.sql`을 아직 실행하지 않아 테이블이 없었음

**해결**
Supabase SQL Editor에서 테이블 생성 SQL 실행 후 재시도

---

## 7. `map_sync.js` ESM/CommonJS 충돌

**증상**
```
SyntaxError: Cannot use import statement in a module
```

**원인**
`package.json`이 `"type": "commonjs"`인데 `map_sync.js`가 `import` 문법(ESM)을 사용

**해결**
`import` → `require()`로 전체 교체

```js
// 변경 전 (ESM)
import { createClient } from '@supabase/supabase-js';

// 변경 후 (CommonJS)
const { createClient } = require('@supabase/supabase-js');
```

---

## 8. TourAPI `Unexpected errors` 응답

**증상**
```json
{ "response": { "header": { "resultCode": "99", "resultMsg": "Unexpected errors" } } }
```

**원인**
오늘 신청한 API 키가 아직 활성화되지 않음 (활성화까지 수 시간~1일 소요)

**해결**
Supabase SQL Editor에서 실제 API 응답 데이터를 직접 INSERT로 임시 대체

```sql
INSERT INTO public.festivals (content_id, title, addr1, mapx, mapy, ...)
VALUES ('real_id_001', '테스트 축제', '서울시 중구', 126.9784, 37.5665, ...);
```

> TourAPI 키 활성화 이후 `node js/map_sync.js` 실행하면 실제 데이터로 교체됨
