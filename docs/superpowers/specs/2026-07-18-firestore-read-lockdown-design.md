# Firestore 무단 전체 열람 차단 설계

## 배경

`firestore.rules.txt`의 현재 규칙은 `events` 컬렉션에 `allow read: if true;`를 걸어두고 있다. Firestore 보안 규칙은 `get`(문서 ID로 단건 조회)과 `list`(쿼리로 여러 건 조회)를 구분하지 않고 `read` 하나로 합쳐서 열면 두 가지 모두 허용된다. 그 결과 앱 UI는 `renderShareView`에서 `where('shareId','==',...)` 쿼리로 공유된 모임 하나만 찾도록 되어 있지만, 규칙 자체는 그 쿼리 형태를 강제하지 않으므로 Firebase JS SDK를 직접 다루는 누구나 `firestore.collection('events').get()`으로 **동기화된 모든 사용자의 모든 모임(계좌번호·예금주명·멤버 이름·항목 내역 포함)을 컬렉션째로 열람**할 수 있다. 앱이 인기를 얻어 사용자가 늘수록 이 구멍이 실제로 악용될 위험도 커지므로, 트래픽이 늘기 전에 닫는다.

## 해결 방향

`get`과 `list`을 분리한다.
- **`get`은 계속 전체 허용**: 문서 ID를 정확히 아는 사람만 단건 조회 가능. 문서 ID를 `event.shareId`(20자 hex, 80비트 무작위값, 사실상 추측 불가)로 삼으면 "링크를 아는 사람만 읽기"라는 기존 의도가 규칙 수준에서 실제로 강제된다.
- **`list`은 로그인한 소유자 본인 것만 허용**: `resource.data.ownerUid == request.auth.uid`. "다른 기기에서 로그인해 내 모임 전체 복원"(`syncFromCloud`의 `where('ownerUid','==',uid)` 쿼리)은 계속 동작하지만, 컬렉션 전체 덤프는 규칙 수준에서 막힌다.

이 설계를 적용하려면 공유 뷰(`renderShareView`)가 지금처럼 `shareId` 필드로 **쿼리(list)** 하는 대신, Firestore 문서 ID 자체를 `shareId`로 삼아 **단건 조회(get)** 하도록 바꿔야 한다. 즉 Firestore에 이벤트를 쓸 때 문서 ID로 `event.id`(내부용, 로컬 라우팅·photoStore 키에 계속 쓰임) 대신 `event.shareId`를 사용한다.

**현재 Firebase 프로젝트(chongmu-603fb)에는 지켜야 할 기존 데이터가 없음을 확인했다** — 문서 ID 스킴이 바뀌어도 별도 마이그레이션 로직은 필요 없다.

## 변경 범위

### 1. `firestore.rules.txt`

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /events/{eventId} {
      allow get: if true;
      allow list: if request.auth != null && request.auth.uid == resource.data.ownerUid;
      allow create: if request.auth != null && request.auth.uid == request.resource.data.ownerUid;
      allow update, delete: if request.auth != null && request.auth.uid == resource.data.ownerUid;
    }
  }
}
```

`create`/`update`/`delete` 규칙은 기존과 동일 — 문서 ID가 무엇이든 `ownerUid` 검사는 그대로 유효하다.

이 파일은 지금까지도 자동 배포되지 않고 프로젝트 소유자가 Firebase 콘솔에 직접 붙여넣는 방식이었다(CLAUDE.md에 이미 문서화됨) — 이번에도 동일하게, 코드 수정 후 사용자가 콘솔에 새 규칙을 다시 붙여넣어야 실제로 적용된다.

### 2. `index.html` — Firestore 문서 ID를 `event.id` → `event.shareId`로 변경

영향받는 지점 3곳 (모두 `.doc(...)` 호출부만 바뀌고, `event.id` 필드 자체나 로컬 라우팅/photoStore 키는 전혀 건드리지 않는다):

- `scheduleCloudSave(event)`: `firestore.collection('events').doc(event.id)` → `.doc(event.shareId)`
- `store.remove(id)`: 현재는 로컬 캐시에서 지운 뒤 같은 `id`로 Firestore 문서를 삭제한다. `shareId`는 캐시에서 지우기 전에 미리 읽어둬야 하므로, 삭제 대상 이벤트를 필터링 전에 먼저 찾아 그 `shareId`로 Firestore 문서를 삭제하도록 바꾼다.
- `syncFromCloud`의 마이그레이션 저장(`firestore.collection('events').doc(ev.id).set(payload)`): `.doc(ev.id)` → `.doc(payload.shareId)` (해당 라인은 이미 바로 위에서 `payload.shareId = ev.shareId || generateShareId()`를 보장한다)

영향받지 않는 지점(그대로 유지):
- `syncFromCloud`의 크로스 디바이스 복원 쿼리(`where('ownerUid','==',user.uid).get()`)는 문서 내용(`d.data()`)만 읽고 문서 ID는 쓰지 않으므로 무변경.
- 로컬 캐시 매칭(`cloudIds = new Set(cloudEvents.map(e => e.id))`)도 문서 내용의 `id` 필드를 쓰는 것이지 Firestore 문서 ID가 아니므로 무변경.
- `renderShareView`: `firestore.collection('events').where('shareId','==',params.shareId).limit(1).get()` → `firestore.collection('events').doc(params.shareId).get()`으로 변경. 문서가 없으면 기존과 동일하게 "모임을 찾을 수 없어요" 화면을 보여준다(스냅샷의 `exists` 필드로 판별).

## 테스트/검증

자동화된 Firestore 규칙 테스트는 이 프로젝트에 없다(에뮬레이터 미사용). 수동 검증:
1. 로그인 후 새 모임을 만들고 저장 → 브라우저 개발자 도구에서 `firestore.collection('events').doc(<shareId>).get()`으로 조회되는지, `firestore.collection('events').get()`(list, 로그아웃 상태)은 권한 오류로 막히는지 확인.
2. 공유 링크(`#/view/{shareId}`)로 비로그인 상태에서 정상적으로 정산 결과가 보이는지 확인.
3. 다른 기기(또는 시크릿 창)에서 같은 계정으로 로그인해 기존 모임이 정상 복원되는지 확인.
4. 모임 삭제 후 Firestore 콘솔에서 해당 문서(`shareId`로 저장된)가 실제로 지워졌는지 확인.
5. 코드 수정과 별개로, **사용자가 Firebase 콘솔에 새 규칙 텍스트를 직접 붙여넣어야** 실제 반영됨을 최종 확인 시 안내한다.
