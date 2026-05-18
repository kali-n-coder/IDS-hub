# IDS Hub と投資ゲームのアカウント連携

IDS Hubのアカウントと投資ゲームのアカウントを、同じメールアドレスで紐づけるための実装メモです。

## 方針

IDS Hubをアカウントの入口として扱い、投資ゲーム側ではFirebase Authenticationのアカウントをそのまま使います。両方のメールアドレスが一致していれば、同じユーザーとして扱います。

パスワードはHubから投資ゲームへ渡しません。

## Hub側でやること

Hubから投資ゲームを開くとき、ログイン中ユーザーの情報をURLパラメータで渡します。

```text
https://game-manager.github.io/stock_sim/?hub=ids&hubEmail=user@example.com&hubName=IDS%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC&hubReturnUrl=https%3A%2F%2Fkali-n-coder.github.io%2FIDS-hub%2F
```

同時に、Hub側の `users/{email}` に連携状態を保存します。

```js
{
  linkedApps: {
    stockGame: {
      linked: true,
      linkedEmail: "user@example.com",
      displayName: "IDSユーザー",
      lastOpenedAt: serverTimestamp()
    }
  }
}
```

## 投資ゲーム側でやること

投資ゲーム側では、ページ読み込み時にURLパラメータを読みます。

```js
const params = new URLSearchParams(location.search);
const hub = params.get("hub");
const hubEmail = params.get("hubEmail");
const hubName = params.get("hubName");
const hubReturnUrl = params.get("hubReturnUrl");

if (hub === "ids" && hubEmail) {
  document.getElementById("auth-email").value = hubEmail;
  const usernameInput = document.getElementById("auth-username");
  if (usernameInput && hubName) {
    usernameInput.value = hubName;
  }
}
```

投資ゲーム側のFirebase Authenticationに同じメールアドレスのユーザーが存在する場合は、そのままログインしてもらいます。存在しない場合は、新規登録フォームに同じメールアドレスを入れた状態で登録してもらいます。

## 連携済み判定

投資ゲーム側でログイン後、Firebase Authの `currentUser.email` とURLから渡された `hubEmail` が一致すれば、IDS Hub連携済みとして扱えます。

```js
function isLinkedWithIdsHub(currentUser) {
  const params = new URLSearchParams(location.search);
  const hubEmail = params.get("hubEmail");
  return Boolean(
    params.get("hub") === "ids" &&
    hubEmail &&
    currentUser?.email &&
    currentUser.email.toLowerCase() === hubEmail.toLowerCase()
  );
}
```

投資ゲーム側のRealtime Databaseにも残すなら、ユーザーデータに次のような情報を保存します。

```js
{
  idsHub: {
    linked: true,
    email: currentUser.email,
    linkedAt: Date.now()
  }
}
```

## やらないこと

- HubのパスワードをURLで渡さない
- Hubの `passwordBase64` を投資ゲーム側へコピーしない
- 投資ゲーム側のFirebase Auth UIDをHubのユーザーIDとして上書きしない

## 将来やるなら

より強い連携にする場合は、Cloud Functionsなどで短時間だけ有効な連携トークンを発行し、投資ゲーム側がそのトークンを検証する方式にします。今の静的HTML中心の構成では、まずメールアドレス一致による連携が現実的です。
