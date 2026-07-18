# Firestore 무단 전체 열람 차단 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `events` 컬렉션에 걸린 `allow read: if true;`를 `get`(단건, 전체 허용)과 `list`(다건 쿼리, 소유자 본인만 허용)로 분리하고, 공유 뷰가 `list` 없이 동작하도록 Firestore 문서 ID를 `event.id`에서 `event.shareId`로 바꾼다.

**Architecture:** `firestore.rules.txt` 규칙 변경 + `index.html`의 Firestore `.doc(...)` 호출 4곳(쓰기 2곳, 삭제 1곳, 읽기 1곳)의 키를 `event.id` → `event.shareId`로 교체. 로컬 라우팅(`#/event/:id`), `photoStore` 키, `event.id` 필드 자체는 전혀 건드리지 않는다 — 오직 "Firestore가 문서를 어떤 값으로 찾느냐"만 바뀐다.

**Tech Stack:** 변경 없음. 자동화 테스트는 브라우저 콘솔에서 도는 `runRegressionTests`(순수 JS, 실제 Firestore 대신 fake 객체로 목킹)뿐이므로 그 fake도 같이 갱신한다.

## Global Constraints

- `firestore.rules.txt`는 자동 배포되지 않는다 — 사용자가 Firebase 콘솔에 직접 붙여넣어야 실제로 적용된다. 이 사실을 최종 verification 단계에서 다시 안내한다.
- 확인 결과 현재 Firebase 프로젝트(chongmu-603fb)에는 지켜야 할 기존 데이터가 없으므로, 문서 ID 스킴 변경에 대한 별도 마이그레이션 로직은 만들지 않는다.
- `event.id`(로컬 라우팅, `photoStore` 키, 회귀 테스트의 로컬 이벤트 매칭)는 이번 변경의 대상이 아니다. `event.shareId`만 Firestore 문서 ID로 새로 쓰인다.
- 이 저장소에는 Node 테스트 러너가 없다 — 유일한 자동 검증은 브라우저에서 `CONFIG.DEV_MODE=true`(또는 콘솔에서 `runRegressionTests()` 직접 호출)로 도는 순수 JS 회귀 테스트다.

---

### Task 1: Firestore 규칙 분리 + 문서 ID를 shareId로 전환 + 회귀 테스트 갱신

**Files:**
- Modify: `firestore.rules.txt` (전체 교체)
- Modify: `index.html` (`scheduleCloudSave`, `store.remove`, `syncFromCloud`, `renderShareView`, 회귀 테스트의 `makeFakeFirestore` 블록)

**Interfaces:**
- Consumes: 기존 `event.shareId`(20자 hex, `store.save`가 항상 부여), `event.id`(내부용, 무변경)
- Produces: 없음(내부 구현 변경, 새 공개 함수 없음)

- [ ] **Step 1: `firestore.rules.txt` 전체 교체**

Find (파일 전체):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /events/{eventId} {
      allow read: if true;
      allow create: if request.auth != null && request.auth.uid == request.resource.data.ownerUid;
      allow update, delete: if request.auth != null && request.auth.uid == resource.data.ownerUid;
    }
  }
}
```

Replace with:
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

- [ ] **Step 2: `scheduleCloudSave`가 `event.shareId`를 문서 ID로 쓰도록 변경**

Find (`index.html`):
```javascript
  function scheduleCloudSave(event) {
    clearTimeout(cloudSaveTimers[event.id]);
    cloudSaveTimers[event.id] = setTimeout(() => {
      if (!currentUser) return; // 디바운스 대기 중 로그아웃된 경우: payload 조립 자체를 건너뜀
      const payload = { ...event, ownerUid: currentUser.uid };
      firestore.collection('events').doc(event.id).set(payload, { merge: true })
        .catch(err => console.warn('클라우드 저장 실패(로컬엔 이미 반영됨):', err));
    }, 800);
  }
```

Replace with:
```javascript
  function scheduleCloudSave(event) {
    clearTimeout(cloudSaveTimers[event.id]);
    cloudSaveTimers[event.id] = setTimeout(() => {
      if (!currentUser) return; // 디바운스 대기 중 로그아웃된 경우: payload 조립 자체를 건너뜀
      const payload = { ...event, ownerUid: currentUser.uid };
      // Firestore 문서 ID는 event.id가 아니라 shareId를 쓴다 — list 권한 없이 get만으로
      // 공유 링크가 동작하게 하기 위함(무단 컬렉션 전체 열람 차단 설계 참고).
      firestore.collection('events').doc(event.shareId).set(payload, { merge: true })
        .catch(err => console.warn('클라우드 저장 실패(로컬엔 이미 반영됨):', err));
    }, 800);
  }
```

- [ ] **Step 3: `store.remove`가 지우기 전에 `shareId`를 먼저 확보하도록 변경**

Find:
```javascript
    remove(id) {
      eventsCache = eventsCache.filter(e => e.id !== id);
      this._writeAllLocal();
      if (currentUser && firebaseReady) {
        firestore.collection('events').doc(id).delete().catch(err => console.warn('클라우드 삭제 실패:', err));
      }
    },
```

Replace with:
```javascript
    remove(id) {
      const existing = eventsCache.find(e => e.id === id);
      eventsCache = eventsCache.filter(e => e.id !== id);
      this._writeAllLocal();
      if (currentUser && firebaseReady && existing && existing.shareId) {
        firestore.collection('events').doc(existing.shareId).delete().catch(err => console.warn('클라우드 삭제 실패:', err));
      }
    },
```

- [ ] **Step 4: `syncFromCloud` 마이그레이션 저장이 `shareId`를 문서 ID로 쓰도록 변경**

Find:
```javascript
      for (const ev of localOnly) {
        const payload = { ...ev, ownerUid: user.uid, shareId: ev.shareId || generateShareId() };
        await firestore.collection('events').doc(ev.id).set(payload).catch(err => console.warn('마이그레이션 실패:', ev.id, err));
        cloudEvents.push(payload);
      }
```

Replace with:
```javascript
      for (const ev of localOnly) {
        const payload = { ...ev, ownerUid: user.uid, shareId: ev.shareId || generateShareId() };
        await firestore.collection('events').doc(payload.shareId).set(payload).catch(err => console.warn('마이그레이션 실패:', ev.id, err));
        cloudEvents.push(payload);
      }
```

- [ ] **Step 5: `renderShareView`가 `where` 쿼리 대신 `shareId`로 단건 `get`하도록 변경**

Find:
```javascript
    const snapshot = await firestore.collection('events').where('shareId', '==', params.shareId).limit(1).get()
      .catch(() => null);
    if (!snapshot || snapshot.empty) {
      app.innerHTML = `<main class="screen"><div class="empty-state"><p>모임을 찾을 수 없어요</p></div></main>`;
      return;
    }
    const ev = snapshot.docs[0].data();
```

Replace with:
```javascript
    const docSnap = await firestore.collection('events').doc(params.shareId).get()
      .catch(() => null);
    if (!docSnap || !docSnap.exists) {
      app.innerHTML = `<main class="screen"><div class="empty-state"><p>모임을 찾을 수 없어요</p></div></main>`;
      return;
    }
    const ev = docSnap.data();
```

- [ ] **Step 6: 회귀 테스트의 fake Firestore를 shareId 키 기준으로 갱신**

Find:
```javascript
    // ---- syncFromCloud: 마이그레이션은 로컬 전용 이벤트만 업로드, 기존 클라우드 데이터는 절대 덮어쓰지 않음 ----
    function makeFakeFirestore(initialDocs) {
      const docs = new Map(initialDocs.map(d => [d.id, d]));
      function queryFrom(list) {
        return {
          where(field, op, value) { return queryFrom(list.filter(d => d[field] === value)); },
          limit(n) { return queryFrom(list.slice(0, n)); },
          get: () => Promise.resolve({ docs: list.map(d => ({ data: () => d })), empty: list.length === 0 }),
        };
      }
      return {
        collection() {
          return Object.assign(queryFrom(Array.from(docs.values())), {
            doc(id) {
              return {
                set: (payload) => { docs.set(id, { ...payload, id }); return Promise.resolve(); },
                delete: () => { docs.delete(id); return Promise.resolve(); },
              };
            },
          });
        },
        _docs: docs,
      };
    }

    const cloudOnlyEvent = { id: 'cloud-only-1', name: '클라우드 전용', ownerUid: 'test-uid', shareId: 'existing-share-id-0001', members: [], items: [] };
    const localOnlyEvent = createEvent('로컬 전용', '2026-01-01');
    addMember(localOnlyEvent, '총무');
    // id가 클라우드 문서와 충돌하는 로컬 객체 — cloudIds 필터가 깨지면(전체를 업로드해버리면) 이 케이스가 클라우드를 덮어써야 정상적으로 감지된다.
    const localCollisionEvent = { id: 'cloud-only-1', name: '로컬에서 조작된 버전', shareId: 'local-share-id-0001', members: [], items: [] };

    const savedFirestore = firestore, savedCurrentUser = currentUser, savedFirebaseReady = firebaseReady;
    const savedStorageRaw = localStorage.getItem(CONFIG.STORAGE_KEY); // 실제 로컬 저장 데이터를 보존(테스트가 store._writeAllLocal을 통해 이 키를 덮어씀)
    const fakeFs = makeFakeFirestore([cloudOnlyEvent]);
    firestore = fakeFs;
    firebaseReady = true;
    eventsCache.push(localOnlyEvent, localCollisionEvent);
    localStorage.removeItem(CONFIG.MIGRATION_FLAG_PREFIX + 'test-uid');

    await syncFromCloud({ uid: 'test-uid' });

    assert('syncFromCloud: 로컬 전용 이벤트가 클라우드로 업로드됨', fakeFs._docs.has(localOnlyEvent.id));
    assert('syncFromCloud: id가 겹치는 기존 클라우드 이벤트는 로컬 값으로 덮어쓰이지 않고 그대로 유지됨', fakeFs._docs.get('cloud-only-1').name === '클라우드 전용');
    assert('syncFromCloud: 병합 후 캐시에 두 이벤트 모두 포함', store.get('cloud-only-1') !== null && store.get(localOnlyEvent.id) !== null);
    assert('syncFromCloud: 마이그레이션 플래그 기록됨', localStorage.getItem(CONFIG.MIGRATION_FLAG_PREFIX + 'test-uid') === '1');

    fakeFs._docs.delete(localOnlyEvent.id); // 클라우드에서만 제거해 재업로드 여부를 관찰
    eventsCache = [localOnlyEvent];
    await syncFromCloud({ uid: 'test-uid' });
    assert('syncFromCloud: 마이그레이션 플래그가 있으면 재업로드하지 않음', !fakeFs._docs.has(localOnlyEvent.id));
```

Replace with:
```javascript
    // ---- syncFromCloud: 마이그레이션은 로컬 전용 이벤트만 업로드, 기존 클라우드 데이터는 절대 덮어쓰지 않음 ----
    // Firestore 문서 ID는 event.id가 아니라 event.shareId를 쓴다(무단 컬렉션 전체 열람 차단 설계 참고).
    function makeFakeFirestore(initialDocs) {
      const docs = new Map(initialDocs.map(d => [d.shareId, d]));
      function queryFrom(list) {
        return {
          where(field, op, value) { return queryFrom(list.filter(d => d[field] === value)); },
          limit(n) { return queryFrom(list.slice(0, n)); },
          get: () => Promise.resolve({ docs: list.map(d => ({ data: () => d })), empty: list.length === 0 }),
        };
      }
      return {
        collection() {
          return Object.assign(queryFrom(Array.from(docs.values())), {
            doc(shareId) {
              return {
                get: () => Promise.resolve(docs.has(shareId) ? { exists: true, data: () => docs.get(shareId) } : { exists: false }),
                set: (payload) => { docs.set(shareId, payload); return Promise.resolve(); },
                delete: () => { docs.delete(shareId); return Promise.resolve(); },
              };
            },
          });
        },
        _docs: docs,
      };
    }

    const cloudOnlyEvent = { id: 'cloud-only-1', name: '클라우드 전용', ownerUid: 'test-uid', shareId: 'existing-share-id-0001', members: [], items: [] };
    const localOnlyEvent = createEvent('로컬 전용', '2026-01-01');
    addMember(localOnlyEvent, '총무');
    // id가 클라우드 문서와 충돌하는 로컬 객체 — cloudIds 필터가 깨지면(전체를 업로드해버리면) 이 케이스가 클라우드를 덮어써야 정상적으로 감지된다.
    const localCollisionEvent = { id: 'cloud-only-1', name: '로컬에서 조작된 버전', shareId: 'local-share-id-0001', members: [], items: [] };

    const savedFirestore = firestore, savedCurrentUser = currentUser, savedFirebaseReady = firebaseReady;
    const savedStorageRaw = localStorage.getItem(CONFIG.STORAGE_KEY); // 실제 로컬 저장 데이터를 보존(테스트가 store._writeAllLocal을 통해 이 키를 덮어씀)
    const fakeFs = makeFakeFirestore([cloudOnlyEvent]);
    firestore = fakeFs;
    firebaseReady = true;
    eventsCache.push(localOnlyEvent, localCollisionEvent);
    localStorage.removeItem(CONFIG.MIGRATION_FLAG_PREFIX + 'test-uid');

    await syncFromCloud({ uid: 'test-uid' });

    const uploadedLocalOnly = store.get(localOnlyEvent.id);
    assert('syncFromCloud: 로컬 전용 이벤트가 클라우드로 업로드됨', uploadedLocalOnly !== null && fakeFs._docs.has(uploadedLocalOnly.shareId));
    assert('syncFromCloud: id가 겹치는 기존 클라우드 이벤트는 로컬 값으로 덮어쓰이지 않고 그대로 유지됨', fakeFs._docs.get('existing-share-id-0001').name === '클라우드 전용');
    assert('syncFromCloud: 병합 후 캐시에 두 이벤트 모두 포함', store.get('cloud-only-1') !== null && store.get(localOnlyEvent.id) !== null);
    assert('syncFromCloud: 마이그레이션 플래그 기록됨', localStorage.getItem(CONFIG.MIGRATION_FLAG_PREFIX + 'test-uid') === '1');

    fakeFs._docs.delete(uploadedLocalOnly.shareId); // 클라우드에서만 제거해 재업로드 여부를 관찰
    eventsCache = [uploadedLocalOnly];
    await syncFromCloud({ uid: 'test-uid' });
    assert('syncFromCloud: 마이그레이션 플래그가 있으면 재업로드하지 않음', !fakeFs._docs.has(uploadedLocalOnly.shareId));
```

- [ ] **Step 7: 회귀 테스트 실행 (브라우저)**

이 저장소엔 Node 테스트 러너가 없으므로 브라우저에서 확인한다:
1. `index.html`에서 `CONFIG.DEV_MODE: false,`를 잠깐 `true`로 바꾼 뒤 브라우저로 열고 콘솔에 모든 `assert` 통과 로그(실패 시 `console.error`로 표시됨)가 뜨는지 확인한다. 특히 `syncFromCloud:`로 시작하는 5개 assert가 모두 통과해야 한다.
2. 확인 후 `CONFIG.DEV_MODE`는 반드시 `false`로 되돌린다(커밋에 `true`가 들어가면 안 됨).

- [ ] **Step 8: 실제 Firebase 콘솔 반영 안내**

이 단계는 코드 변경이 아니라 사용자 액션이다. 커밋 후 다음을 사용자에게 안내한다:
> `firestore.rules.txt`의 새 내용을 Firebase 콘솔 → Firestore Database → 규칙 탭에 복사해 붙여넣고 게시해야 실제로 적용됩니다. 코드만 바꿔서는 지금 운영 중인 프로젝트의 규칙이 자동으로 바뀌지 않습니다.

- [ ] **Step 9: 커밋**

```bash
git add index.html firestore.rules.txt
git commit -m "$(cat <<'EOF'
Firestore list 권한을 소유자로 제한 (컬렉션 전체 열람 차단)

get/list을 분리해 공유 링크 조회(get)는 계속 허용하되, 컬렉션 전체
쿼리(list)는 로그인한 소유자 본인 데이터로만 제한한다. 문서 ID를
event.id 대신 event.shareId로 써서 공유 뷰가 list 없이 get만으로
동작하도록 바꿨다.

EOF
)"
```
