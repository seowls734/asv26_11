# CLAUDE.md — 행복탕 사장님 (동네 목욕탕 3D 경영 게임)

이 파일은 Claude가 이 프로젝트를 이어서 작업할 때 먼저 읽는 안내서입니다.

## 프로젝트 한눈에 보기

- **장르**: 3D 동네 목욕탕 경영 + 생활 시뮬레이션 (한국어 UI)
- **파일**: `index.html` 하나에 HTML·CSS·JS가 모두 들어 있음 (약 2,300줄). 빌드 과정 없음.
- **실행**: 브라우저로 `index.html`을 열면 끝. 3D 엔진 **Three.js r128**을 CDN에서 불러오므로 **인터넷 연결이 필요**함.
  - `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js` (전역 `THREE`)
  - 폰트: Google Fonts `Jua`, `Gowun Dodum`
- **게시**: claude.ai 아티팩트로 게시됨 (`https://claude.ai/artifact/7Nk6byytymY3eRD68tid8s`). 수정 후에는 같은 아티팩트에 다시 게시.
- **외부 라이브러리 추가 금지**: r128 전역 빌드만 사용. `CapsuleGeometry`(r139+), ES 모듈, OrbitControls 등은 쓰지 않음.

## 게임 규칙 요약

- 7일(`DAYS`) 동안 영업. 하루 영업 시간은 `DAY_LEN = 100`초 (화면 시계 06:00~22:00).
- 시작 화면에서 **🧑‍💼 사장 모드** 또는 **🧖 손님 모드** 선택 (`S.role = 'owner' | 'customer'`).
- **사장 모드**: 손님을 시설에 안내 → 요금 수입. 온도·청결 관리, 바나나우유 판매, 편의점 장보기, 하루 끝 정산 → 밤에 집에서 자기 → 아침 상점(업그레이드) → 다음 날.
- **손님 모드**: 지갑(`S.wallet`, 시작 50,000원, 매일 아침 +30,000원)으로 시설을 직접 이용해 **개운함**(`S.fresh`, 0~100)을 올림. 목욕탕은 `autoManage()`가 NPC 사장처럼 자동 운영.
- 시설 (`FAC`): `warm` 온탕(39~42℃), `cold` 냉탕(15~20℃), `sauna` 사우나, `scrub` 때밀이. 탕 온도는 계속 변함(`drift`).
- 손님 종류: 일반, **👑 VIP**(10%, 요금 3배·팁 15,000원·까다로움), **👿 진상**(13%, 인내심 짧음·3배로 어지럽힘·💢 소리 지르면 진정시켜야 함·가끔 반값 요구).
- 날씨 (`WEATHER`): 맑음(10초에 3명꼴) / 흐림(10초에 1~2명) / 비·눈(12~18초에 1명). 매일 무작위, 다음 날 예보 표시.
- 손님은 나갈 때 **사물함에서 옷을 갈아입고**(`doDress`) 나감. 표정: `neutral / happy / angry / sad` (`setExpr`).

## 조작

| 입력 | 동작 |
|---|---|
| WASD / 방향키 | 내 캐릭터 이동 (카메라 기준). Shift 달리기 |
| E / Space | 근처 상호작용 (손님 안내·시설 보내기·우유·편의점·집·차·침대 등) |
| R / F | 근처 시설 청소 / 탕 온도 조절 |
| H | 운전 중 경적 |
| 마우스 드래그 | 360° 회전 + 위아래 각도 / 우클릭·Shift+드래그 이동 / 휠 확대 |
| Z·C / T·G / +·- | 키보드 회전 / 높이 / 확대 |
| 모바일 | 왼쪽 아래 조이스틱(끝까지 밀면 달리기), 오른쪽 E·🧹·🌡️ 버튼, 한 손가락 회전·두 손가락 확대/이동 |

## 관리자 패널

- 영업 중이나 밤에 HUD의 **♨ 행복탕** 간판을 2.5초 안에 5번 클릭 → 비밀번호 입력.
- 비밀번호: **`bath2026`** (코드에는 FNV-1a 해시 `ADMIN_HASH = '65fe8b02'`만 저장). 바꾸려면 새 비밀번호의 FNV-1a 32비트 hex를 계산해 `ADMIN_HASH`에 넣음.
- 통과하면 `localStorage['hbt_admin'] = '1'` → 그 기기에서만 🛠️ 버튼이 보임. 새 사용자에게는 안 보임.
- 서버가 없는 정적 페이지라 완벽한 보안은 아님 (코드 분석으로 우회 가능).
- 기능: 돈, 평판, 날씨, 손님 소환(일반/VIP/진상), 업그레이드 최대, 청소, 게임 속도(`ADMIN.speed`), 즉시 마감, 밤 전환, 날짜 변경, 인내심 무한.

## 코드 구조 (`index.html` 안의 `<script>` 순서)

1. **게임 데이터**: `FAC`, `WEATHER`, `WISH`, `PEOPLE`(손님 외모 14종), `UPG`(상점 업그레이드)
2. **공간 배치**: `SEATS`(대기 의자 8자리), `SLOTS`(시설별 자리 좌표·포즈), `entryPts()`(시설 출입 경유점), 통로 `AISLE_H`/`AISLE_C`, 문 `DOOR_Z`, 출구 `EXIT`
3. **상태**: 전역 `S`(게임 상태), `F`(UI 요소 참조), `selected`(선택된 손님), `T`(누적 시간)
4. **3D 헬퍼**: `M()`(재질 캐시), `mesh/box/cyl/sph/plane/group`, `canvasTex/tileTex/woodTex/textTex`(캔버스 텍스처), `tagPick()`(클릭 판정용 `userData.pick`)
5. **건물**: `buildWorld()` → `buildShowers/buildTub/buildSauna/buildScrub/buildChanging/buildSteam/buildOutside`
   - 바깥: `buildOutside()`(도로·가게·아파트 단지·굴뚝), `buildConvenience()`(편의점 외관), `buildApartment()`(행복아파트+집 내부), `buildPremium()`(프리미엄 타워+펜트하우스), `buildStoreInterior()`(편의점 내부)
6. **캐릭터**: `makePerson(spec)` → `{g, hip, legL, legR, armL, armR, head, face, body}` / `setPose(p, 'walk'|'sit'|'soak'|'lie'|'stand')` / `setExpr()` / `putOn()`(옷 입기)
7. **이동**: `route(from, fromK, to, toK)`(통로·문을 거치는 경로), `sendTo(c, anchor, k)`, `moveCustomer()`
8. **라벨**: 3D 좌표를 화면에 투영하는 `placeEl()`; 시설 이름표 `PLATE_POS`, 손님 말풍선 `.tag`(`data-m`: `q` 대기 / `s` 이용 중 / `r` 반응)
9. **게임 로직**: `spawn(forceType)`, `clickCustomer`, `clickFacility`, `finishService`, `depart`, `angryLeave`, `tick(dt)`(영업 중 로직), `animate(dt)`(애니메이션), `render3D(dt)`
10. **하루 흐름**: `startDay` → (시간 끝) `startClosing` → `endDay`(정산) → `goNight` → 침대에서 `sleep()` → 아침 `showShop()` → 다음 `startDay`. 손님 모드는 `endDayCustomer()`.
11. **플레이어**: `PLAYER`(`x, z, inHome, inStore, inPrem, using, jacuzzi`), `updatePlayer`, `collide(P, R, extra, room)`, `interactMain`, `nearestTargets`
12. **운전**: `DRIVE`, `buildMyCar`, `enterCar/exitCar/updateCar`, `CAR_BLOCK`
13. **편의점 내부**: `SHELVES`, `OWNER_ITEMS`/`CUST_ITEMS`, `S.basket`, `addToBasket`, `payBasket`
14. **역할**: `startRole`, `applyRoleLook`, `useFacility/finishMe`(손님이 직접 이용), `autoManage`
15. **프리미엄**: `PREM_PRICE = 1,000,000`, `premDoor`, `enterPrem/exitPrem`, `soakJacuzzi`, 숙면 보너스(`S.sleptPrem`)
16. **관리자**: `ADMIN`, `openAdmin`, `adminAction`
17. **날씨 효과**: `buildWeatherFx`, `updateWeather`(밤 조명·가로등·창문 불빛도 여기서 처리)
18. **루프**: `loop()` → `tick` / `animate` / `updatePlayer` / `render3D` → `renderer.render` → `renderLabels` / `renderPanel`

## 좌표계와 장소

- 단위: 대략 1 = 1m. 바닥 y=0. 캐릭터 정면 = +z (회전 0).
- **목욕탕**: x -10~10. 욕탕 홀 z -7.3~1.45, 탈의실 z 1.45~7. 뒤·왼쪽 벽만 있고 지붕 없음(위에서 보는 단면 구조).
- **앞 도로**: z 10~17. 내 차 처음 위치 (3, 12.2).
- **편의점 외관** (-15, 5), 문 앞 (-15, 8.4). **행복아파트** (24, 2), 문 앞 (24, 5.5). **프리미엄 타워** (30.5, -6), 문 앞 (30.5, -2.3).
- **실내 방은 멀리 따로 만들어 순간이동**: 집 `HX = 70`, 편의점 내부 `SX = 100`, 펜트하우스 `PX = 130`. 방마다 `collide()`의 `room` 인자(`'home' | 'store' | 'prem'`)로 경계와 장애물(`HOME_OBST` 등)을 바꿈.
- 장애물: `OBST`(바깥, `[x0, z0, x1, z1]` 사각형). 새 건물을 추가하면 `OBST.push()` 해야 함.

## 새 기능 추가 방법

- **새 시설 / 장소 이름표**: `PLATE_POS`에 좌표 추가 → `buildUI()`와 `renderLabels()`의 키 목록 두 곳에 추가 → `clickFacility()`에 처리 추가.
- **새 상호작용**: `interactMain()`(E키 동작)과 `updatePlayer()`의 힌트 문구(`h = ...`) 두 곳을 같은 순서로 수정. 우선순위: 운전 > 시설 이용 중 > 편의점 안 > 프리미엄 > 집 > 차 > 역할별 동작.
- **새 실내 공간**: 멀리 떨어진 좌표에 방을 만들고, `PLAYER.inXxx` 플래그, 들어가기/나가기 함수, `collide()`의 room 분기를 추가.
- **새 손님 종류**: `spawn()`의 type 결정 부분, `wishText`/`TYPE_ICON`, 요금·평판 가중치(`finishService`, `depart`, `angryLeave`).
- **밤에 바뀌는 연출**: `updateWeather()` 안의 `W.night`(0~1) 값을 사용.
- **상점 업그레이드**: `UPG` 배열에 추가하고 `buy()`에서 효과 처리.

## 작업 규칙

- UI 문구, 주석, 안내는 모두 **한국어**. 귀여운 레트로 동네 목욕탕 분위기 유지.
- 한 파일 구조 유지 (`index.html`). 외부 이미지는 쓰지 않고, 모든 그림은 Three.js 도형과 캔버스 텍스처로 만듦.
- 아티팩트 환경 제약: `alert/confirm/prompt` 금지 → 직접 만든 모달(`showModal`) 사용. `localStorage`는 try/catch로 감쌈.
- 키 입력은 `e.code`(`KeyW` 등)로 판별 → 한글 입력 상태에서도 동작.
- 수정 후 확인: `<script>` 내용만 뽑아 `node --check`로 문법 검사.
  ```bash
  python3 -c "import re;s=open('index.html').read();open('/tmp/g.js','w').write(re.findall(r'<script>(.*?)</script>',s,re.S)[-1])" && node --check /tmp/g.js
  ```
  작업 환경에서 cdnjs 접속이 막혀 있으면 3D 화면은 직접 띄워 볼 수 없음 → 사용자에게 확인을 부탁할 것.
- 실존 브랜드나 상표 디자인을 흉내 내지 않음 (편의점은 "행복 편의점", 우유병은 일반 노란 병).

## 변경 이력 (요약)

1. 2D 카드형 목욕탕 경영 게임 → 일러스트 내부
2. Three.js 3D 건물·캐릭터, 손님 이동·포즈
3. 바깥 동네(도로·가게·아파트·굴뚝), 손님 10초에 1~2명
4. 손님 표정(기쁨·화남), 탈의실 옷 갈아입기
5. 날씨 시스템, 3D 편의점
6. WASD 캐릭터 조종, 모바일 조이스틱, 자유 시점
7. 내 차 운전, VIP·진상 손님
8. 행복아파트(우리 집)·밤·잠자기 → 다음 날 아침
9. 관리자 패널(비밀번호)
10. 사장/손님 모드 선택, 편의점 안에서 직접 골라 사기
11. 프리미엄 타워(펜트하우스, 자쿠지, 숙면 보너스)
