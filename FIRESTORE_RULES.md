# 재료 관리 앱 — 계정관리(vault) 보안 규칙

계정관리 탭의 보관함(`vault/main`)은 **재료 관리자로 지정된 구글 계정**과 **최고관리자**만 읽고 쓸 수 있어야 합니다.
(보관함 내용은 비밀번호로 암호화돼 있지만, 암호문 자체도 지정되지 않은 사람이 받아 가지 못하게 막는 것입니다.)

재료 관리자 목록은 자라다 People 앱 → 관리 → 설정 → **재료 관리자** 카드에서 최고관리자가 지정하며,
이 프로젝트(`jarada-materials`)의 `config/materialManagers` 문서(`{ emails: [...] }`)에 저장됩니다.

## 적용 방법 (Firebase 콘솔, 프로젝트 `jarada-materials`)

1. Firestore Database → **규칙** 탭을 엽니다.
2. 기존 규칙(`materials`, `requests` 등)은 **그대로 두고**, 아래 `match` 두 개와 함수 세 개를 같은 `match /databases/{database}/documents { … }` 안에 추가합니다.
3. `SUPER_ADMIN_EMAILS` 의 이메일을 **최고관리자의 구글 계정**으로 바꿉니다. (People 앱 최고관리자와 같은 계정이어야 합니다.)
4. **게시**를 누릅니다.

```
    // ---------- 계정관리 / 재료 관리자 ----------
    function superAdminEmails() {
      return ['REPLACE_WITH_SUPER_ADMIN@gmail.com'];   // ← 최고관리자 구글 이메일(소문자)로 교체
    }
    function mailOk() {
      return request.auth != null && request.auth.token.get('email_verified', false) == true;
    }
    function mailLower() {
      return request.auth.token.email.lower();
    }
    function isMatSuper() {
      return mailOk() && mailLower() in superAdminEmails();
    }
    function isMatManager() {
      return mailOk()
        && exists(/databases/$(database)/documents/config/materialManagers)
        && mailLower() in get(/databases/$(database)/documents/config/materialManagers).data.get('emails', []);
    }

    // 재료 관리자 목록: 최고관리자만 변경, 최고관리자·재료 관리자만 조회
    match /config/materialManagers {
      allow read: if isMatSuper() || isMatManager();
      allow write: if isMatSuper()
        && request.resource.data.keys().hasOnly(['emails', 'updatedBy', 'updatedAt'])
        && request.resource.data.emails is list
        && request.resource.data.emails.size() <= 20;
    }

    // 계정관리 보관함(비밀번호로 암호화된 데이터)
    match /vault/{docId} {
      allow read, write: if isMatSuper() || isMatManager();
    }
```

## 동작

- 목록에 없는 계정이 계정관리 탭을 열면 “Firebase 보안 규칙이 계정관리 저장소를 막고 있습니다” 안내가 나옵니다.
- 재료 관리자로 지정돼도 **보관함 비밀번호**를 알아야 내용을 봅니다. 비밀번호는 People 앱에서 보이지 않습니다.
- 재료 관리자에서 빠진 사람은 다음 요청부터 보관함을 읽지 못합니다. 이미 알고 있는 비밀번호는 바뀌지 않으므로, 사람이 바뀌면 계정관리 탭의 **비밀번호 변경**도 함께 하세요.
- 최고관리자가 People 앱 설정에서 목록을 처음 저장하려면, 재료 앱 프로젝트에도 같은 구글 계정으로 한 번 연결(로그인)해야 합니다. `cooldesign81.github.io` 는 이미 승인된 도메인입니다.
