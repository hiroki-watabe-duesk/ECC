---
name: click-path-audit
description: "ユーザーが操作するすべてのボタン/タッチポイントをその完全な状態変更シーケンスでトレースし、個々の関数は動作するが互いに打ち消し合ったり、最終状態が誤っていたり、UIが矛盾した状態になるバグを発見する。使用時: 系統的なデバッグでバグが見つからないのにユーザーがボタン不具合を報告する場合、または共有ステートストアに触れる大規模リファクタリング後。"
origin: community
---

# /click-path-audit — 動作フロー監査

静的なコード読み取りでは見逃すバグを発見します: ステートの相互作用の副作用、連続呼び出し間の競合状態、ハンドラーが互いに無音でアンドゥし合う問題。

## この問題が解決すること

従来のデバッグが確認すること:
- 関数は存在するか？（配線の欠落）
- クラッシュするか？（ランタイムエラー）
- 正しい型を返すか？（データフロー）

しかし確認**しない**こと:
- **最終的なUIの状態はボタンラベルが約束するものと一致しているか？**
- **関数Bが関数Aの変更を無音でアンドゥしていないか？**
- **共有ステート（Zustand/Redux/context）が意図した操作をキャンセルする副作用を持っていないか？**

実際の例: 「新規メール」ボタンが `setComposeMode(true)` を呼び出し、次に `selectThread(null)` を呼び出していました。両者は個別に動作していました。しかし `selectThread` には `composeMode: false` をリセットする副作用があり、ボタンは何もしませんでした。系統的なデバッグで54件のバグが発見されましたが、このバグは見逃されていました。

---

## 動作原理

対象領域のすべてのインタラクティブなタッチポイントについて:

```
1. ハンドラーを特定する（onClick, onSubmit, onChange等）
2. ハンドラー内のすべての関数呼び出しを順番にトレースする
3. 各関数呼び出しについて:
   a. どのステートをREADするか？
   b. どのステートをWRITEするか？
   c. 共有ステートへのSIDE EFFECTがあるか？
   d. 副作用でステートをリセット/クリアするか？
4. 後の呼び出しが前の呼び出しのステート変更をUNDOするか確認する
5. 最終ステートがボタンラベルからユーザーが期待するものと一致するか確認する
6. 競合状態はあるか（非同期呼び出しが誤った順序で解決する）を確認する
```

---

## 実行ステップ

### ステップ1: ステートストアをマップする

タッチポイントを監査する前に、すべてのステートストアアクションの副作用マップを構築します:

```
スコープ内の各Zustandストア / Reactコンテキストについて:
  各アクション/セッターについて:
    - どのフィールドをセットするか？
    - 副作用として他のフィールドをRESETするか？
    - ドキュメント化: actionName → {sets: [...], resets: [...]}
```

これが重要なリファレンスです。「新規メール」バグは `selectThread` が `composeMode` をリセットすることを知らなければ見えませんでした。

**出力フォーマット:**
```
STORE: emailStore
  setComposeMode(bool) → sets: {composeMode}
  selectThread(thread|null) → sets: {selectedThread, selectedThreadId, messages, drafts, selectedDraft, summary} RESETS: {composeMode: false, composeData: null, redraftOpen: false}
  setDraftGenerating(bool) → sets: {draftGenerating}
  ...

危険なリセット（所有していないステートをクリアするアクション）:
  selectThread → setComposeMode所有のcomposeModeをリセット
  reset → すべてをリセット
```

### ステップ2: 各タッチポイントを監査する

対象領域の各ボタン/トグル/フォーム送信について:

```
タッチポイント: [ボタンラベル] in [Component:行番号]
  ハンドラー: onClick → {
    呼び出し1: functionA() → sets {X: true}
    呼び出し2: functionB() → sets {Y: null} RESETS {X: false}  ← 競合
  }
  期待値: ユーザーがボタンラベルから期待すること
  実際: functionBがリセットしたためXはfalse
  判定: バグ — [説明]
```

**次のバグパターンを確認してください:**

#### パターン1: 連続アンドゥ
```
handler() {
  setState_A(true)     // sets X = true
  setState_B(null)     // side effect: resets X = false
}
// 結果: Xはfalse。最初の呼び出しは無意味だった。
```

#### パターン2: 非同期競合
```
handler() {
  fetchA().then(() => setState({ loading: false }))
  fetchB().then(() => setState({ loading: true }))
}
// 結果: 最終的なloadingの状態はどちらが先に解決するかによる
```

#### パターン3: ステールクロージャ
```
const [count, setCount] = useState(0)
const handler = useCallback(() => {
  setCount(count + 1)  // 古いcountをキャプチャ
  setCount(count + 1)  // 同じ古いcount — 2ではなく1だけ増加する
}, [count])
```

#### パターン4: ステート遷移の欠落
```
// ボタンには「保存」と書かれているが、ハンドラーは検証するだけで実際には保存しない
// ボタンには「削除」と書かれているが、ハンドラーはAPIを呼び出さずフラグをセットするだけ
// ボタンには「送信」と書かれているが、APIエンドポイントが削除/壊れている
```

#### パターン5: 条件付きデッドパス
```
handler() {
  if (someState) {        // someStateはこの時点で常にfalse
    doTheActualThing()    // 到達しない
  }
}
```

#### パターン6: useEffectの干渉
```
// ボタンがstateX = trueをセットする
// useEffectがstateXを監視してfalseにリセットする
// ユーザーには何も起きていないように見える
```

### ステップ3: レポート

発見された各バグについて:

```
CLICK-PATH-NNN: [severity: CRITICAL/HIGH/MEDIUM/LOW]
  タッチポイント: [ボタンラベル] in [file:line]
  パターン: [連続アンドゥ / 非同期競合 / ステールクロージャ / 遷移欠落 / デッドパス / useEffect干渉]
  ハンドラー: [関数名またはインライン]
  トレース:
    1. [呼び出し] → sets {field: value}
    2. [呼び出し] → RESETS {field: value}  ← 競合
  期待値: [ユーザーが期待すること]
  実際: [実際に起きること]
  修正: [具体的な修正方法]
```

---

## スコープ制御

この監査はコストが高いです。適切にスコープを設定してください:

- **アプリ全体の監査:** ローンチ時や大規模リファクタリング後に使用。ページごとに並列エージェントを起動する。
- **単一ページ監査:** 新しいページを作成した後や、ユーザーが壊れたボタンを報告した後に使用。
- **ストア重点監査:** Zustandストアを変更した後に使用 — 変更されたアクションの全消費者を監査する。

### アプリ全体のおすすめエージェント分割:

```
Agent 1: すべてのステートストアをマップ（ステップ1）— 他のすべてのエージェントの共有コンテキスト
Agent 2: ダッシュボード（Tasks, Notes, Journal, Ideas）
Agent 3: チャット（DanteChatColumn, JustChatPage）
Agent 4: メール（ThreadList, DraftArea, EmailsPage）
Agent 5: プロジェクト（ProjectsPage, ProjectOverviewTab, NewProjectWizard）
Agent 6: CRM（全サブタブ）
Agent 7: プロフィール、設定、Vault、通知
Agent 8: マネジメントスイート（全ページ）
```

Agent 1は最初に完了しなければなりません。その出力は他のすべてのエージェントへの入力です。

---

## 使用すべき時

- 系統的なデバッグで「バグなし」と判定されたにもかかわらず、ユーザーがUIの不具合を報告する場合
- 任意のZustandストアアクションを変更した後（全呼び出し元を確認する）
- 共有ステートに触れるリファクタリングの後
- リリース前の重要なユーザーフローで
- ボタンが「何もしない」場合 — これがそのためのツールです

## 使用すべきでない時

- APIレベルのバグ（誤ったレスポンス形状、欠落したエンドポイント）— systematic-debuggingを使用
- スタイル/レイアウトの問題 — 視覚的な検査
- パフォーマンスの問題 — プロファイリングツール

---

## 他のスキルとの連携

- `/superpowers:systematic-debugging` の後に実行する（他の54種類のバグを発見する）
- `/superpowers:verification-before-completion` の前に実行する（修正が機能することを検証する）
- `/superpowers:test-driven-development` に繋げる — ここで発見された各バグはテストを持つべき

---

## 例: このスキルを生み出したバグ

**ThreadList.tsx「New Email」ボタン:**
```
onClick={() => {
  useEmailStore.getState().setComposeMode(true)   // ✓ composeMode = true をセット
  useEmailStore.getState().selectThread(null)      // ✗ composeMode = false にリセット
}}
```

ストア定義:
```
selectThread: (thread) => set({
  selectedThread: thread,
  selectedThreadId: thread?.id ?? null,
  messages: [],
  drafts: [],
  selectedDraft: null,
  summary: null,
  composeMode: false,     // ← この無音リセットがボタンを殺した
  composeData: null,
  redraftOpen: false,
})
```

**系統的なデバッグが見逃した理由:**
- ボタンにはonClickハンドラーがある（デッドではない）
- 両方の関数が存在する（配線の欠落なし）
- どちらの関数もクラッシュしない（ランタイムエラーなし）
- データ型は正しい（型不一致なし）

**クリックパス監査が発見した理由:**
- ステップ1が `selectThread` は `composeMode` をリセットするとマップする
- ステップ2がハンドラーをトレースする: 呼び出し1がtrueをセット、呼び出し2がfalseにリセット
- 判定: 連続アンドゥ — 最終ステートがボタンの意図と矛盾する
