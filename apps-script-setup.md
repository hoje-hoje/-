# 구글 드라이브 자동 업로드 설정 (제작자 전용, 최초 1회)

기여자들이 `index.html`에서 "드라이브로 보내기"를 누르면, 로그인 없이도
바로 **제작자의 구글 드라이브**에 데이터가 저장되도록 만드는 설정입니다.
아래 과정은 데이터를 받을 사람(제작자) 한 명만 하면 됩니다.

## 1. 드라이브에 저장 폴더 만들기
1. 구글 드라이브에서 새 폴더를 만들고 이름을 예: `벚꽃결투_카드제출함` 으로 지정
2. 폴더를 열고 주소창의 URL에서 `folders/` 뒤의 문자열(폴더 ID)을 복사해둠

## 2. Apps Script 프로젝트 만들기
1. https://script.google.com 접속 → "새 프로젝트"
2. 아래 코드를 전체 붙여넣기 (기존 코드 지우고 교체)

```javascript
// 1번에서 복사한 폴더 ID를 아래에 붙여넣으세요
const FOLDER_ID = '여기에_폴더_ID_붙여넣기';
const DELETED_LOG_FILENAME = '_deleted.json'; // 모든 기기가 공유하는 삭제 기록 파일

function doPost(e) {
  try {
    const folder = DriveApp.getFolderById(FOLDER_ID);
    const body = JSON.parse(e.postData.contents);
    const incoming = body.data; // {goddesses, cards, deletedCardIds, deletedGoddessIds}

    // 0) 이번에 온 삭제 기록을, 드라이브에 저장된 공유 삭제 기록과 합침 (다른 기기가 예전에 지운 것도 항상 같이 적용됨)
    const deletedLog = mergeDeletedLog(folder, incoming.deletedCardIds || [], incoming.deletedGoddessIds || []);
    const deletedCardIds = deletedLog.cardIds;
    const deletedGoddessIds = deletedLog.goddessIds;

    // 1) 삭제된 여신은 그 여신 파일 자체를 드라이브에서 지움
    if (deletedGoddessIds.length) {
      const allFiles = folder.getFiles();
      while (allFiles.hasNext()) {
        const f = allFiles.next();
        if (f.getName().slice(-5) !== '.json' || f.getName() === DELETED_LOG_FILENAME) continue;
        try {
          const rec = JSON.parse(f.getBlob().getDataAsString());
          if (rec.goddess && deletedGoddessIds.indexOf(rec.goddess.id) !== -1) {
            f.setTrashed(true);
          }
        } catch (err) {
          // 형식이 이상한 파일은 건너뜀
        }
      }
    }

    incoming.goddesses.forEach(function (god) {
      if (deletedGoddessIds.indexOf(god.id) !== -1) return; // 삭제된 여신은 다시 안 만듦
      const filename = sanitizeName(god.name) + '.json';
      const godCards = incoming.cards.filter(function (c) {
        return c.goddessId === god.id && deletedCardIds.indexOf(c.id) === -1;
      });

      const files = folder.getFilesByName(filename);
      let record;
      if (files.hasNext()) {
        const file = files.next();
        const existing = JSON.parse(file.getBlob().getDataAsString());
        record = mergeGoddessRecord(existing, god, godCards, deletedCardIds);
        file.setContent(JSON.stringify(record, null, 2));
      } else {
        record = { goddess: god, cards: godCards };
        folder.createFile(filename, JSON.stringify(record, null, 2), MimeType.PLAIN_TEXT);
      }
    });

    return ContentService.createTextOutput(JSON.stringify({ ok: true }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ ok: false, error: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// 공유 삭제 기록 파일(_deleted.json)을 읽어서 이번 요청의 삭제 목록과 합친 뒤 다시 저장
function mergeDeletedLog(folder, newCardIds, newGoddessIds) {
  let log = { cardIds: [], goddessIds: [] };
  const files = folder.getFilesByName(DELETED_LOG_FILENAME);
  let file = null;
  if (files.hasNext()) {
    file = files.next();
    try { log = JSON.parse(file.getBlob().getDataAsString()); } catch (err) {}
  }
  log.cardIds = uniqueMerge(log.cardIds || [], newCardIds);
  log.goddessIds = uniqueMerge(log.goddessIds || [], newGoddessIds);

  if (file) {
    file.setContent(JSON.stringify(log));
  } else {
    folder.createFile(DELETED_LOG_FILENAME, JSON.stringify(log), MimeType.PLAIN_TEXT);
  }
  return log;
}
function uniqueMerge(a, b) {
  const set = {};
  a.concat(b).forEach(function (id) { set[id] = true; });
  return Object.keys(set);
}

// 파일명에는 날짜나 올린 사람 이름, 카드 개수를 넣지 않고, 여신 이름만 사용
function sanitizeName(name) {
  return (name || '이름없음').replace(/[\\/:*?"<>|]/g, '').trim();
}

// 같은 여신 파일이 이미 있으면, 카드 id 기준으로 합침(같은 id는 최신 걸로 덮어씀)
// deletedCardIds에 있는 카드는 기존 데이터에 남아있어도 이번에 실제로 제거함
function mergeGoddessRecord(existing, god, incomingCards, deletedCardIds) {
  const cardMap = {};
  (existing.cards || []).forEach(function (c) {
    if (deletedCardIds.indexOf(c.id) === -1) cardMap[c.id] = c;
  });
  incomingCards.forEach(function (c) { cardMap[c.id] = c; });
  return {
    goddess: god, // 이름/이미지는 가장 최근에 보낸 걸로 갱신됨
    cards: Object.keys(cardMap).map(function (k) { return cardMap[k]; })
  };
}

// index.html이 "드라이브에서 가져오기"를 누르거나 자동으로 새로고침할 때 호출됨
// 폴더 안의 모든 여신 파일을 모아서 하나로 합쳐 돌려주고, 공유 삭제 기록도 같이 내려보냄
function doGet(e) {
  try {
    const folder = DriveApp.getFolderById(FOLDER_ID);
    const files = folder.getFiles();
    const goddesses = [];
    const cards = [];
    let deletedLog = { cardIds: [], goddessIds: [] };
    while (files.hasNext()) {
      const file = files.next();
      if (file.getName().slice(-5) !== '.json') continue;
      if (file.getName() === DELETED_LOG_FILENAME) {
        try { deletedLog = JSON.parse(file.getBlob().getDataAsString()); } catch (err) {}
        continue;
      }
      try {
        const record = JSON.parse(file.getBlob().getDataAsString());
        if (record.goddess) goddesses.push(record.goddess);
        if (record.cards) cards.push.apply(cards, record.cards);
      } catch (err) {
        // 형식이 이상한 파일은 건너뜀
      }
    }
    return ContentService.createTextOutput(JSON.stringify({
      goddesses: goddesses,
      cards: cards,
      deletedCardIds: deletedLog.cardIds || [],
      deletedGoddessIds: deletedLog.goddessIds || []
    })).setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ error: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

**동작 방식**
- 드라이브 폴더 안에 여신마다 `여신이름.json` 파일이 하나씩 생김 (날짜·올린 사람 이름·카드 개수는 파일명에 안 들어감)
- 카드를 추가·수정할 때마다 이 파일이 계속 갱신됨(같은 파일에 카드가 id 기준으로 자동 합쳐짐)
- **누구든 카드를 올리면**, index.html이 페이지를 열 때 + 30초마다 자동으로 드라이브를 다시 읽어와서 **다른 사람이 올린 내용도 화면에 자동으로 반영됨** (새로고침 안 해도 됨)
- 주의 1: 카드나 여신을 **삭제**하면, 삭제 목록(어떤 id를 지웠는지)이 같이 전송돼서 드라이브 쪽 파일에서도 실제로 지워져요. 자동 전송(2.5초 후)이 끝나기 전에 드라이브 새로고침이 먼저 일어나면 아주 잠깐 다시 보일 수는 있지만, 곧 사라져요
- 주의 2: 여신 **이름을 바꾸면** 새 이름으로 새 파일이 생겨요(예전 이름 파일은 그대로 남음) — 이름은 되도록 확정한 뒤에 다 같이 입력하는 걸 추천해요

3. 프로젝트 이름을 예: `벚꽃결투 카드 수집기` 로 저장 (디스크 아이콘 또는 Ctrl/Cmd+S)

## 3. 웹앱으로 배포하기
1. 우측 상단 "배포" → "새 배포"
2. 유형 선택(톱니바퀴 아이콘) → "웹앱" 선택
3. 설정:
   - 실행 계정: **나(본인)**
   - 액세스 권한: **모든 사용자** (Anyone) — 기여자가 로그인 없이 보낼 수 있게 하려면 필수
4. "배포" 클릭 → 처음 실행 시 권한 승인 창이 뜨면 본인 계정으로 승인
5. 배포가 끝나면 나오는 **웹앱 URL**(`https://script.google.com/macros/s/xxxxx/exec` 형태)을 복사

## 4. index.html에 연결하기
1. `index.html`을 열고 우측 상단 "⚙ 드라이브 설정" 클릭
2. 3번에서 복사한 웹앱 URL을 붙여넣고 저장
3. 기여자들에게도 이 `index.html` 파일(또는 깃허브 페이지 링크)과 함께
   **이 웹앱 URL을 어떻게 알려줄지**만 정하면 됩니다:
   - 방법 A: `index.html` 안에 URL을 미리 박아서 배포 (코드에서 `localStorage.getItem('scriptUrl')` 대신 고정값 사용)
   - 방법 B(권장, 현재 구조): 기여자마다 처음 한 번 "⚙ 드라이브 설정"에서 URL을 붙여넣게 안내

## 참고
- 코드를 수정해서 배포를 업데이트할 때는 "배포" → "배포 관리" → 연필 아이콘 → "새 버전"으로 다시 배포해야
  변경사항이 실제 웹앱 URL에 반영됩니다.
- 폴더 안에는 여신마다 `여신이름.json` 파일이 하나씩 쌓이고 계속 최신 상태로 자동 병합되므로,
  나중에 이 파일들을 그대로 읽어서 깃허브의 덱빌더 사이트 데이터로 쓰면 됩니다(따로 병합 스크립트가 필요 없음).
