# CLAUDE.md — EVE Clever 재배환경 모니터링 대시보드

> 이 파일은 이 폴더에서 작업하는 AI 코딩 어시스턴트(Claude Code 등)를 위한 지침입니다.
> 작업 시작 전에 이 문서를 먼저 읽고, 아래 **설계 원칙**과 **제약**을 지켜서 이어가세요.
> 최종 업데이트: 2026-07-03

---

## 1. 프로젝트 개요

- **무엇:** 성호에스아이코퍼레이션 농업 로봇 **EVE Clever(SUNGHO-RS)** 의 재배환경 데이터를 그래프로 보여주는 웹 대시보드.
- **누가 보나:** KIRIA 현장 시연(수요일 발표) 및 외부 손님(금요일) 대상 데모. 태블릿·스마트폰에서 **외부망**으로 접근.
- **왜 이 구조인가:** 사내 로컬 서버가 과열로 종종 꺼지므로, 대시보드를 그 서버와 **완전히 분리(decouple)** 했다. 백엔드 없이 정적 파일 하나로 돌아가고, 무료·항상 켜진 정적 호스팅에 올려 서버가 죽어도 대시보드는 영향받지 않게 한다.
- **최우선 가치:** **안정성 > 기능**. 데모 주간에는 스택을 크게 흔들지 않는다.

---

## 2. 현재 상태

- 단일 파일 **`index.html`** — 빌드 과정 없음. 더블클릭으로도 열리고, 정적 호스팅에 그대로 올라간다.
- 외부 의존성: **Chart.js v4** (cdnjs), **IBM Plex Sans KR / IBM Plex Mono** (Google Fonts). 둘 다 폴백 폰트가 있어 오프라인/차단 시에도 깨지지 않는다.
- 데이터: 지금은 `index.html` 안 **`ENV` 객체에 하드코딩된 샘플 스냅샷**(24시간 hourly + 최근 7일 daily).
- 실시간 탭: 실제 센서가 아니라 **브라우저에서 만든 모의 스트림**(1.5초 간격).
- 메뉴 6개: 개요 / 온·습도 / 토양·수분 / 대기·광량 / 실시간 모니터 / 데이터 연동.

---

## 3. 코드 지도 (핵심 위치)

`index.html` 하단 `<script>` 안에 전부 있다. 수정 시 아래 지점을 기준으로 잡을 것.

| 심볼 | 역할 | 비고 |
|---|---|---|
| `ENV` (객체) | **모든 원본 데이터** | 실측 전환 시 여기를 채우게 된다 |
| `loadData()` | 데이터 로딩 진입점 | **CSV 연동은 이 함수만 고치면 됨** (현재는 즉시 반환) |
| `vpd(t, rh)` | 온도·습도로 VPD(kPa) 계산 | 데이터 갱신 후 `ENV.vpd` 재계산 필요 |
| `build()` | 모든 차트 생성 | 생성된 차트는 `charts` 객체에 보관 |
| `line() / baseOpts() / areaFill()` | 차트 공통 헬퍼 | **새 차트는 반드시 이걸 재사용** |
| `renderReadout()` | 상단 KPI 판독 스트립 렌더 | 현재 시각 인덱스 기준 |
| `pushLiveReading()` | 실시간 모의 값 1건 생성 | **실측 스트림 연결 지점** (API/WebSocket로 교체) |
| `go(view)` | 탭 라우팅 + 실시간 타이머 제어 | |

---

## 4. 설계 원칙 · 컨벤션 (지킬 것)

- **단일 파일 유지.** 특별한 이유 없이 빌드 도구(webpack/vite 등)나 npm 의존성을 도입하지 않는다. 이 프로젝트의 강점은 "파일 하나 = 배포 끝"이다.
- **외부 스크립트는 cdnjs만.** 아티팩트 미리보기·정적 호스팅 호환을 위해 새 라이브러리도 `cdnjs.cloudflare.com`에서 가져온다.
- **디자인 토큰을 벗어나지 말 것.** 새 색을 임의로 만들지 말고 아래 팔레트를 쓴다. 수치는 반드시 모노스페이스(IBM Plex Mono).

  | 토큰 | 값 | 용도 |
  |---|---|---|
  | `--leaf` | `#256B4E` | 주 강조(온도/정상) |
  | `--sky` | `#3E6E8E` | 보조 데이터(습도/수분) |
  | `--soil` | `#B4622E` | CO₂ 등 |
  | `--amber` | `#C98A21` | 경고·VPD·광량 |
  | `--ink` `--haze` `--line` | `#182320` `#6C776F` `#E2E7DE` | 텍스트/보조/선 |

- **차트는 헬퍼 재사용.** 색·툴팁·격자 스타일을 직접 쓰지 말고 `line()`과 `baseOpts()`를 통해 만든다. 그래야 톤이 일관된다.
- **한글 우선.** UI 문구는 한국어, 문장부호와 단위 표기 일관성 유지.
- **접근성 하한선 유지:** 키보드 포커스 표시, 모바일(≈380px)까지 반응형, `prefers-reduced-motion` 존중 — 이미 들어가 있으니 깨뜨리지 말 것.

---

## 5. 다음 작업 — 실측 CSV 연동 (가능함, 이렇게 하면 됨)

가능하다. 지금 구조가 이미 이걸 전제로 비워둔 상태다. `ENV` 하드코딩을 CSV 로딩으로 바꾸기만 하면 된다.

### 5-1. CSV 스키마 (권장)

`data/hourly.csv` — 24시간(또는 그 이상) 시계열, 헤더 고정:

```csv
timestamp,temp,humidity,co2,soil,ppfd
2026-07-03T00:00,18.2,78,690,41.0,0
2026-07-03T01:00,17.9,79,700,40.6,0
2026-07-03T06:00,18.4,78,680,44.5,20
```

- `timestamp`: ISO8601. 화면 라벨은 `HH:MM`만 사용.
- 단위: temp=°C, humidity=%, co2=ppm, soil=%, ppfd=µmol/m²/s.

`data/daily.csv` — 최근 N일 집계:

```csv
date,tmin,tmax,dli,irrig
6/27,16.9,26.8,16.2,12
6/28,17.2,27.5,18.9,12
```

- dli=일적산광량(mol/m²/일), irrig=관수량(L/일).

### 5-2. `loadData()` 교체 예시 (무의존 파서, 실패 시 샘플 폴백)

```js
async function loadData(){
  try{
    const rows = parseCSV(await (await fetch('data/hourly.csv')).text());
    ENV.hours    = rows.map(r => r.timestamp.slice(11,16));
    ENV.temp     = rows.map(r => +r.temp);
    ENV.humidity = rows.map(r => +r.humidity);
    ENV.co2      = rows.map(r => +r.co2);
    ENV.soil     = rows.map(r => +r.soil);
    ENV.ppfd     = rows.map(r => +r.ppfd);
    ENV.vpd      = ENV.temp.map((t,i)=> vpd(t, ENV.humidity[i])); // 반드시 재계산

    const d = parseCSV(await (await fetch('data/daily.csv')).text());
    ENV.days  = d.map(r=> r.date);
    ENV.tmin  = d.map(r=> +r.tmin);
    ENV.tmax  = d.map(r=> +r.tmax);
    ENV.dli   = d.map(r=> +r.dli);
    ENV.irrig = d.map(r=> +r.irrig);
  }catch(e){
    console.warn('실측 CSV 로드 실패 — 내장 샘플로 표시:', e.message); // ← 데모 안전장치
  }
  return ENV;
}

// 헤더 기반 초경량 파서 (쉼표에 값이 없는 단순 CSV 전제)
function parseCSV(text){
  const [head, ...lines] = text.trim().split(/\r?\n/);
  const cols = head.split(',').map(s=>s.trim());
  return lines.filter(Boolean).map(line=>{
    const cells = line.split(',');
    return Object.fromEntries(cols.map((c,i)=>[c, cells[i]?.trim()]));
  });
}
```

- **폴백 패턴 유지:** CSV가 없거나 깨져도 `catch`가 내장 샘플로 되돌린다. 데모 중 파일 문제로 화면이 비는 사고를 막는 안전장치이니 지우지 말 것.
- 값에 쉼표·따옴표가 들어가는 복잡한 CSV라면 무의존 파서 대신 **PapaParse**(cdnjs)로 교체. 그 외엔 위 파서로 충분.
- `irrigationHours`(관수 시점 마커)는 daily의 관수 정보나 별도 컬럼에서 유도하도록 나중에 확장.

### 5-3. ⚠️ 로컬 테스트 주의

`file://`(더블클릭)로 열면 브라우저 보안 때문에 `fetch('data/...csv')`가 **막힌다**. CSV 연동 테스트는 반드시 로컬 서버로:

```bash
python3 -m http.server 8000   # → http://localhost:8000
```

배포본(Netlify/GitHub Pages)에서는 정상 동작한다. 현재 내장 샘플은 파일에 박혀 있어 더블클릭으로도 보인다.

---

## 6. 배포

정적 파일이라 서버를 켜둘 필요가 없다(sleep·콜드스타트 없음).

- **가장 빠름 — Netlify Drop:** `netlify.com/drop`에 `index.html`(+`data/` 폴더) 드래그 → 즉시 HTTPS 주소. 로그인 시 주소 고정.
- **GitHub Pages:** 리포에 커밋 → Settings → Pages → 브랜치 지정. 기존 조직 `sunghocorp`과 잘 맞음.
- **Cloudflare Pages:** Git 연결 또는 직접 업로드.

세 곳 모두 무료·항상 켜짐. 로컬 과열 서버와 독립.

---

## 7. 로드맵

1. **CSV 스냅샷 연동** (§5) — 현재 다음 단계.
2. **주기적 갱신** — 로컬 서버가 하루 몇 번 `data/*.csv`를 export해 리포에 push(또는 호스팅에 업로드). 대시보드는 새로고침 시 최신 반영.
3. **(선택) 실시간** — `pushLiveReading()`을 WebSocket/폴링 또는 BaaS(Firebase Realtime DB / Supabase) 구독으로 교체.
4. **(선택) Django 통합** — 로봇 대시보드에 흡수할 경우, Django는 REST API만 제공하고 이 정적 프론트가 화면을 담당하는 **분리형**으로. 서버 렌더링 템플릿에 억지로 얹지 말 것(배포 복잡도만 늘어남).

---

## 8. 제약 · 주의 (Guardrails)

- **데모 주간엔 안정성 최우선.** 큰 리팩터링·스택 교체는 데모 이후로. 변경 후엔 반드시 배포본 또는 로컬 서버에서 실제 태블릿/폰으로 확인.
- **비밀키·개인정보 하드코딩 금지.** 정적 파일은 공개된다. API 키가 필요한 연동은 프론트에 키를 넣지 말고 별도 처리.
- **실시간 탭은 "시연용 모의"임을 UI에 명시된 상태 유지** — 실측처럼 오인되게 라벨을 떼지 말 것.
- **폴백을 제거하지 말 것** (§5-2). 데모 사고 방지용.
