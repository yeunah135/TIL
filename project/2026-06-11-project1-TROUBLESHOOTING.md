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
Uncaught ReferenceError: kakao is not defined at map_kakao.js
```

**원인**
카카오맵 SDK가 로드되기 전에 `map_main.js`가 실행됨

**해결**
`map.html`에서 카카오맵 스크립트를 동적으로 생성하고 `onload` 콜백 안에서 `map_main.js`를 로드

```html
<script src="/config.js"></script>
<script>
  const _s = document.createElement('script');
  _s.src = `https://dapi.kakao.com/v2/maps/sdk.js?appkey=${window.KAKAO_MAP_KEY}&autoload=false&libraries=clusterer`;
  _s.onload = () => {
    kakao.maps.load(() => {
      const _m = document.createElement('script');
      _m.type = 'module';
      _m.src = '../js/map_main.js';
      document.head.appendChild(_m);
    });
  };
  document.head.appendChild(_s);
</script>
```

---

## 4. `kakao.maps.LatLng is not a constructor` 오류

**증상**
```
Uncaught TypeError: kakao.maps.LatLng is not a constructor at map_kakao.js
```

**원인**
`autoload` 옵션 없이 동적 로드 시 카카오맵 내부 초기화가 완료되기 전에 실행됨

**해결**
- SDK URL에 `autoload=false` 추가
- `kakao.maps.load()` 콜백 안에서 `map_main.js` 로드
- `map_kakao.js`의 `initMap()` 내부에서 `kakao.maps.load()` 중복 제거

---

## 5. `map_sync.js` ESM/CommonJS 충돌

**증상**
```
SyntaxError: Cannot use import statement outside a module
```

**원인**
`package.json`이 `"type": "commonjs"`인데 `map_sync.js`가 ESM(`import`) 문법을 사용

**해결**
`import` → `require()`로 전체 교체

```js
// 수정 전 (ESM)
import { createClient } from '@supabase/supabase-js';

// 수정 후 (CommonJS)
require('dotenv').config();
const { createClient } = require('@supabase/supabase-js');
```

---

## 6. TourAPI `Unexpected errors` 응답

**증상**
```
TourAPI 응답 파싱 실패. 응답 내용: Unexpected errors
```

**원인**
당일 신청한 API 키는 활성화까지 수 시간 ~ 1일 소요됨

**해결**
공공데이터포털에서 실제 API 응답 데이터를 받아 Supabase SQL Editor에서 직접 INSERT로 임시 대체

```sql
-- 기존 테스트 데이터 삭제
DELETE FROM public.festivals WHERE content_id LIKE 'test%';

-- 실제 API 응답 데이터 INSERT
INSERT INTO public.festivals (content_id, title, start_date, end_date, addr, lat, lng, detail_url, area_code)
VALUES (...);
```

> API 키 활성화 후 `node js/map_sync.js` 실행 시 자동으로 최신 데이터로 갱신됨
