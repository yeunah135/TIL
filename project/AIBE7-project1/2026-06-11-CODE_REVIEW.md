# 코드 리뷰 - 기술 및 핵심 코드 정리

## 1. Supabase ESM vs CommonJS 분리

브라우저용과 Node.js용 Supabase 클라이언트를 다르게 불러와야 해요.

```js
// 브라우저 (map_api.js) - ESM CDN
import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js/+esm';

// Node.js (map_sync.js) - CommonJS
const { createClient } = require('@supabase/supabase-js');
```

> 브라우저는 `import`/`export` ESM 문법, Node.js는 `require`/`module.exports` CommonJS 문법을 써요.
> 같은 라이브러리라도 환경에 따라 불러오는 방식이 달라요.

---

## 2. 환경변수를 브라우저에 주입하는 패턴

`server.js`에서 `/config.js` 엔드포인트로 환경변수를 `window` 객체에 주입해요.
프론트엔드에서 `.env`를 직접 읽을 수 없는 문제를 해결하는 패턴이에요.

```js
// server.js
app.get('/config.js', (req, res) => {
  res.type('application/javascript').send(
    `window.SUPABASE_URL=${JSON.stringify(process.env.SUPABASE_URL)};
     window.SUPABASE_ANON_KEY=${JSON.stringify(process.env.SUPABASE_ANON_KEY)};
     window.KAKAO_MAP_KEY=${JSON.stringify(process.env.KAKAO_MAP_KEY)};`
  );
});
```

```html
<!-- map.html - 다른 스크립트보다 먼저 로드 -->
<script src="/config.js"></script>

<!-- 이후 window.SUPABASE_URL 등으로 접근 가능 -->
```

---

## 3. 카카오맵 동적 로드 패턴

카카오맵 SDK를 동적으로 로드할 때 `autoload=false` + `kakao.maps.load()` 콜백 패턴이에요.
SDK가 완전히 초기화된 후에 지도 관련 코드가 실행되는 것을 보장해요.

```js
const _s = document.createElement('script');
_s.src = `https://dapi.kakao.com/v2/maps/sdk.js?appkey=${key}&autoload=false&libraries=clusterer,services`;
_s.onload = () => {
  kakao.maps.load(() => {
    // 여기서부터 kakao.maps.* 사용 가능
    const _m = document.createElement('script');
    _m.type = 'module';
    _m.src = '../js/map_main.js';
    document.head.appendChild(_m);
  });
};
document.head.appendChild(_s);
```

> `autoload=false` 없이 동적 로드하면 SDK 내부 초기화 전에 코드가 실행되어
> `kakao.maps.LatLng is not a constructor` 오류가 발생해요.

---

## 4. 이벤트 위임 패턴

동적으로 생성된 요소에 이벤트를 붙일 때 개별 `addEventListener` 대신
부모 요소에 이벤트를 달아서 처리해요.

```js
// ❌ 개별 이벤트 - innerHTML 교체 시 이벤트가 사라짐
list.querySelectorAll('.card').forEach((el, i) => {
  el.addEventListener('click', () => handler(items[i]));
});

// ✅ 이벤트 위임 - innerHTML 교체해도 유지됨
list.onclick = (e) => {
  const card = e.target.closest('.search-result-card');
  if (!card) return;
  const idx = parseInt(card.dataset.idx);
  handler(items[idx]);
};
```

> `data-idx` 속성으로 인덱스를 HTML에 저장해두고 클릭 시 꺼내 쓰는 게 핵심이에요.

---

## 5. Haversine 공식 - 두 좌표 간 거리 계산

위도/경도 두 좌표 사이의 실제 거리(km)를 계산해요.
주변 축제 필터링에 사용했어요.

```js
function getDistance(lat1, lng1, lat2, lng2) {
  const R = 6371; // 지구 반지름 (km)
  const dLat = ((lat2 - lat1) * Math.PI) / 180;
  const dLng = ((lng2 - lng1) * Math.PI) / 180;
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos((lat1 * Math.PI) / 180) *
    Math.cos((lat2 * Math.PI) / 180) *
    Math.sin(dLng / 2) * Math.sin(dLng / 2);
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}

// 사용 예시 - 반경 20km 이내 축제 필터링
const nearby = allFestivals.filter(f =>
  getDistance(place.y, place.x, f.lat, f.lng) <= 20
);
```

---

## 6. Supabase RLS (Row Level Security) 패턴

테이블별 접근 권한을 DB 레벨에서 제어해요.
애플리케이션 코드가 아닌 DB에서 보안을 처리하는 방식이에요.

```sql
-- RLS 활성화 (기본적으로 모든 접근 차단)
alter table public.festival_bookmarks enable row level security;

-- 읽기: 본인 데이터만
create policy "본인_즐겨찾기_조회"
  on public.festival_bookmarks for select
  using (auth.uid() = user_id);

-- 쓰기: 본인 데이터만
create policy "본인_즐겨찾기_추가"
  on public.festival_bookmarks for insert
  with check (auth.uid() = user_id);

-- 삭제: 본인 데이터만
create policy "본인_즐겨찾기_삭제"
  on public.festival_bookmarks for delete
  using (auth.uid() = user_id);
```

> `auth.uid()`는 현재 로그인한 사용자의 UUID를 반환하는 Supabase 내장 함수예요.
> `anon` 키로 접근해도 RLS가 적용되어 다른 사람 데이터에 접근할 수 없어요.

---

## 7. TourAPI upsert 패턴

중복 없이 데이터를 넣거나 업데이트해요.
`content_id`가 같으면 UPDATE, 없으면 INSERT로 동작해요.

```js
await supabase
  .from('festivals')
  .upsert(chunk, { onConflict: 'content_id' });
```

> 매일 sync를 돌려도 같은 축제가 중복으로 쌓이지 않아요.
> `onConflict`에 unique 컬럼을 지정해야 해요.

---

## 8. Promise.all로 병렬 처리

두 개의 비동기 작업을 동시에 실행해서 시간을 절약해요.

```js
// ❌ 순차 실행 - 각각 기다림 (느림)
const festivals = await fetchFestivals(state);
const bookmarks = await loadBookmarks();

// ✅ 병렬 실행 - 동시에 시작 (빠름)
[allFestivals] = await Promise.all([
  fetchFestivals(state),
  loadBookmarks(),
]);
```

> 두 요청이 서로 의존하지 않을 때 `Promise.all`을 쓰면 전체 대기 시간을
> 가장 오래 걸리는 요청 하나로 줄일 수 있어요.
