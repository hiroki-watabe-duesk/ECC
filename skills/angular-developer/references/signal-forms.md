# Signal Forms

Signal Formsは、対象とするAngularバージョンがサポートしている場合、新規フォームの推奨実装方法です。Angular Signalsを使用して、リアクティブで型安全かつモデル駆動なフォーム状態管理を提供します。

Signal Formsを使用する際は、フィールドの値や型に `null` を使用しないでください。

## インポート

`@angular/forms/signals` から以下をインポートできます。

```ts
import {
  form,
  FormField,
  submit,
  // フィールド状態のルール
  disabled,
  hidden,
  readonly,
  debounce,
  // スキーマヘルパー
  applyWhen,
  applyEach,
  schema,
  // カスタムバリデーション
  validate,
  validateHttp,
  validateStandardSchema,
  // メタデータ
  metadata,
} from '@angular/forms/signals';
```

## フォームの作成

Signalモデルとともに `form()` 関数を使用します。フォームの構造はモデルから直接導出されます。

```ts
import {Component, signal} from '@angular/core';
import {form, FormField} from '@angular/forms/signals';

@Component({
  // ...
  imports: [FormField],
})
export class Example {
  // 1. 初期値を持つモデルを定義する（undefinedは避ける）
  userModel = signal({
    name: '', // 重要: 初期値に null や undefined を絶対に使用しない
    email: '',
    age: 0, // 数値には 0 を使用し、null は使わない
    address: {
      street: '',
      city: '',
    },
    hobbies: [] as string[], // 配列には [] を使用し、null は使わない
  });

  // 間違い — これを行わないこと:
  // badModel = signal({
  //   name: null,      // エラー: '' を使用すること
  //   age: null,       // エラー: 0 を使用すること
  //   items: null      // エラー: [] を使用すること
  // });

  // 2. フォームを作成する
  userForm = form(this.userModel);
}
```

## バリデーション

`@angular/forms/signals` からバリデーターをインポートします。

```ts
import {required, email, min, max, minLength, maxLength, pattern} from '@angular/forms/signals';
```

`form()` に渡すschema関数の中で使用します。

```ts
userForm = form(this.userModel, (schemaPath) => {
  // required（必須）
  required(schemaPath.name, {message: 'Name is required'});

  // 条件付きrequired
  required(schemaPath.name, {
    when({valueOf}) {
      return valueOf(schemaPath.age) > 10;
    },
  });
  // when は required にのみ使用可能
  // これを行わないこと: pattern(p.name, /xxx/, {when /* エラー */)

  // email（メール形式）
  email(schemaPath.email, {message: 'Invalid email'});

  // 数値のMin/Max
  min(schemaPath.age, 18);
  max(schemaPath.age, 100);

  // 文字列・配列のMinLength/MaxLength
  minLength(schemaPath.password, 8);
  maxLength(schemaPath.description, 500);

  // Pattern（正規表現）
  pattern(schemaPath.zipCode, /^\d{5}$/);
});
```

## FieldState と FormField：親となる要件

**FormField**（構造）と **FieldState**（実際のデータ/Signal）の違いを理解することが重要です。

**ルール**: 状態Signalにアクセスするには、フィールドを関数として**呼び出す**必要があります（valid、touched、dirty、hidden など）。

```ts
// f は FormField（構造的）
const f = form(signal({cat: {name: 'pirojok-the-cat', age: 5}}));

f.cat.name; // FormField: ここからフラグを取得できません！
f.cat.name.touched(); // エラー: touched() は FormField に存在しない

f.cat.name(); // FieldState: 呼び出すことでSignalにアクセスできる
f.cat.name().touched(); // 有効: Signalにアクセスしている
f.cat().name.touched(); // エラー: f.cat() は状態であり、子要素を持たない！
```

テンプレートでも同様:

```html
<!-- 間違い: 'hidden' プロパティは 'FormField' 型に存在しない -->
@if (bookingForm.hotelDetails.hidden()) { ... }

<!-- 正しい: 先に呼び出す -->
@if (bookingForm.hotelDetails().hidden()) { ... }
```

## Disabled / Readonly / Hidden

スキーマ内のルールを使用してフィールドの状態を制御します。

```ts
import {disabled, readonly, hidden} from '@angular/forms/signals';

userForm = form(this.userModel, (schemaPath) => {
  // 条件付きdisabled
  disabled(schemaPath.password, ({valueOf}) => !valueOf(schemaPath.createAccount));

  // 条件付きhidden（モデルからは削除されず、hiddenとしてマークされるだけ）
  hidden(schemaPath.shippingAddress, ({valueOf}) => valueOf(schemaPath.sameAsBilling));

  // readonly
  readonly(schemaPath.username);
});
```

## バインディング

`FormField` をインポートして `[formField]` ディレクティブを使用します。

```ts
import {FormField} from '@angular/forms/signals';
```

`disabled`、`hidden`、`readonly`、`name` などの状態の全プロパティは自動的にバインドされます。
`name` フィールドを手動でバインドしないでください。

**重要: 禁止属性**
`[formField]` 使用時は、テンプレートで以下の属性を設定してはなりません（静的・バインド問わず）:

- `min`、`max`（代わりにスキーマ内のバリデーターを使用する）
- `value`、`[value]`、`[attr.value]`（`[formField]` によって処理済み）
- `[attr.min]`、`[attr.max]`
- `[disabled]`、`[readonly]`（`[formField]` によって処理済み）

これを行わないこと: `<input min="1" [formField]>` または `<input [value]="val" [formField]>`。

```html
<!-- Input -->
<input [formField]="userForm.name" />

<!-- Checkbox -->
<input type="checkbox" [formField]="userForm.isAdmin" />

<!-- Select -->
<select [formField]="userForm.country">
  <option value="us">US</option>
</select>

<!-- userForm.name は nullable にできない。inputはnullを受け付けないため -->
<input [formField]="userForm.name" />
```

## リアクティブフォーム

`@angular/forms` から `FormControl`、`FormGroup`、`FormArray`、`FormBuilder` を**インポートしないでください**。Signal Formsがこれらの概念を完全に置き換えます。
Signal Formsにはbuilderがありません。

## 状態へのアクセス

フォーム内の各フィールドは状態を返す関数です。

```ts
// フィールドを呼び出してアクセスする
const emailState = this.userForm.email();

// 値（WritableSignal）
const value = this.userForm().value();

// バリデーション状態（Signals）
const isValid = this.userForm().valid();
const isInvalid = this.userForm().invalid();
const errors = this.userForm().errors(); // エラーの配列
const isPending = this.userForm().pending(); // 非同期バリデーション待ち

// インタラクション状態（Signals）
const isTouched = this.userForm().touched();
const isDirty = this.userForm().dirty();

// 有効性状態（Signals）
const isDisabled = this.userForm().disabled();
const isHidden = this.userForm().hidden();
const isReadonly = this.userForm().readonly();
```

重要：状態を取得するには必ずフィールドを呼び出すこと。

```ts
form().invalid()
form.field().dirty()
form.field.subfield().touched()
form.a.b.c.d().value()
form.address.ssn().pending()
form().reset()

// 唯一の例外はlength:
form.children.length
form.length // 注意: 括弧なし！
form.client.addresses.length  // "()" なし

@for (income of form.addresses; track $index) {/**/}
```

## 送信

`submit()` 関数を使用します。アクションを実行する前に、全フィールドにtouchedを自動的に付与します。

**重要**: `submit()` へのコールバックは `async` でなければならず、Promiseを返す必要があります。

```ts
import { submit } from '@angular/forms/signals';

// 正しい — asyncコールバック
onSubmit() {
  submit(this.userForm, async () => {
    // フォームが有効な場合のみ実行される
    await this.apiService.save(this.userModel());
    console.log('Saved!');
  });
}

// 間違い — asyncキーワードが欠けている
onSubmit() {
  submit(this.userForm, () => {  // エラー: asyncでなければならない
    console.log('Saved!');
  });
}
```

## エラーの処理

`field().errors()` はValidationErrorの配列を返します。

```ts
interface ValidationError {
  readonly kind: string;
  readonly message?: string;
}
```

バリデーターからnullを返さないでください。
エラーがない場合はundefinedを返します。

### コンテキスト

`validate()`、`disabled()`、`applyWhen` などのルールに渡される関数はコンテキストオブジェクトを受け取ります。その構造を理解することが**重要**です。

```ts
validate(
  schemaPath.username,
  ({
    value, // Signal<T>: フィールドの現在値を示す書き込み可能なSignal
    fieldTree, // FieldTree<T>: サブフィールド（グループ・配列の場合）
    state, // FieldState<T>: state.valid()、state.dirty() などのフラグにアクセス
    valueOf, // (path) => T: 他のフィールドの値を読み取る（依存関係を追跡）例: valueOf(schemaPath.password)
    stateOf, // (path) => FieldState: 他のフィールドの状態（valid/dirty）にアクセス 例: stateOf(schemaPath.password).valid()
    pathKeys, // Signal<string[]>: ルートからこのフィールドへのパス
  }) => {
    // 間違い: if (touched()) ... (touched はコンテキストにない)
    // 正しい: if (state.touched()) ...

    if (value() === 'admin') {
      return {kind: 'reserved', message: 'Username admin is reserved'};
    }
  },
);
```

### 重要: パスはSignalではない

`form()` コールバック内で、`schemaPath` およびその子要素（例: `schemaPath.user.name`）は **Signal ではなく**、**呼び出し可能でもありません**。

```ts
// 間違い — エラーが発生します:
applyWhen(p.ssn, () => p.ssn().touched(), (ssnField) => { ... });

// 正しい — stateOf() を使用してパスの状態を取得する:
applyWhen(p.ssn, ({ stateOf }) => stateOf(p.ssn).touched(), (ssnField) => { ... });

// 正しい — valueOf() を使用してパスの値を取得する:
applyWhen(p.ssn, ({ valueOf }) => valueOf(p.ssn) !== '', (ssnField) => { ... });
```

### 複数アイテム

- アイテムごとにルールを適用するには `applyEach` を使用する。
- **重要**: `applyEach` のコールバックが受け取る引数は**1つだけ**（アイテムのパス）であり、2つではありません。

```ts
// 正しい — 引数1つ
applyEach(s.items, (item) => {
  required(item.name);
});

// 間違い — インデックスを渡さないこと
applyEach(s.items, (item, index) => {
  // エラー: コールバックの引数は1つ
  required(item.name);
});
```

- テンプレートではアイテムの反復に `@for` を使用する。
- 配列からアイテムを削除するには、データ内の対応するアイテムを配列から取り除くだけです。
- **`select` バインディング**: `<select [formField]="form.country">` へのバインドは可能。optionに `value` 属性があることを確認すること。

### ネストした @for ループ

**重要**: Angularには `$parent` が存在しません。ネストしたループでは、外側のインデックスを変数に格納してください。

```html
<!-- 間違い — $parent は存在しない -->
@for (item of form.items; track $index) { @for (option of item.options; track $index) {
<button (click)="removeOption($parent.$index, $index)">Remove</button>
<!-- エラー -->
} }

<!-- 正しい — let を使用して外側のインデックスを格納する -->
@for (item of form.items; track $index; let outerIndex = $index) { @for (option of item.options;
track $index) {
<button (click)="removeOption(outerIndex, $index)">Remove</button>
} }
```

### フォームボタンの無効化

```html
<button [disabled]="form().invalid() || form().pending()" />
<!-- または -->
<button [disabled]="taxForm.invalid()" />
```

inputに `[disabled]` を使用しないでください。`[formField]` がこれを処理します。
inputに `[readonly]` を使用しないでください。`[formField]` がこれを処理します。
フィールドをdisabledまたはreadonlyにする必要がある場合は、スキーマ内で `disabled()` または `readonly()` ルールを使用してください。

### 非同期バリデーション

非同期には `validate()` を使用せず、`validateAsync()` を使用してください。

**重要**:

1. `params` オプションは、バリデートする値を返す関数でなければなりません。
2. `onError` ハンドラーは**必須**です。省略できません！

```ts
import {resource} from '@angular/core';
import {validateAsync} from '@angular/forms/signals';

userForm = form(this.userModel, (s) => {
  validateAsync(s.username, {
    // 1. 関数でなければならない — paramsはコンテキストを受け取り値を返す
    params: ({value}) => value(),

    // 2. リソースを作成する — ファクトリーはSignalを受け取る
    factory: (username) =>
      resource({
        params: username, // resource() では 'params' を使用する
        loader: async ({params: value}) => {
          await new Promise((resolve) => setTimeout(resolve, 1000));
          return value === 'taken';
        },
      }),

    // 3. 成功結果をエラーにマッピングする
    onSuccess: (isTaken) =>
      isTaken ? {kind: 'taken', message: 'Username is already taken'} : undefined,

    // 4. エラーを処理する — これは必須！
    onError: () => ({kind: 'error', message: 'Validation failed'}),
  });
});
```

**間違いの例:**

```ts
// 間違い — params は関数でなければならない
validateAsync(s.username, {
  params: s.username, // エラー: ({ value }) => value() でなければならない
  // ...
});

// 間違い — onError が欠けている（必須！）
validateAsync(s.username, {
  params: ({value}) => value(),
  factory: (username) =>
    resource({
      /* ... */
    }),
  onSuccess: (result) => (result ? {kind: 'error'} : undefined),
  // エラー: 'onError' が欠けているが必須！
});
```

### resource の使用

**重要**: Angularの `resource()` では、入力Signalに `params` を使用します。

```ts
// 正しい
resource({
  params: mySignal,
  loader: async ({params: value}) => {
    /* ... */
  },
});

// 間違い
resource({
  request: mySignal, // エラー: 'params' を使うべき
  loader: async ({request}) => {
    /* ... */
  },
});
```

UIとモデルの同期を遅延させるには `debounce()` を使用します。

```ts
import {debounce} from '@angular/forms/signals';

userForm = form(this.userModel, (s) => {
  // モデルの更新を300ms遅延させる
  debounce(s.username, 300);
});
```

### 条件付きバリデーション

```ts
form(
  data,
  (path) => {
    applyWhen(
      name,
      ({value}) => value() !== 'admin',
      (namePath) => {
        validate(namePath.last /* ... */);
        disable(namePath.last /* ... */);
      },
    );
  },
  {injector: TestBed.inject(Injector)},
);
```

`applyWhen` は第1引数にマッピングされたパスを渡します。
親フィールドが必要な場合は、`applyWhen` に渡すだけです。

```ts
form(
  data,
  (path) => {
    applyWhen(
      cat,
      ({value}) => value().name !== 'admin',
      (catPath) => {
        require(cat.catPath /* ... */);
      },
    );
  },
  {injector: TestBed.inject(Injector)},
);
```

## よくある落とし穴（これを行わないこと）

| エラーシナリオ | 間違い（よくあるミス） | 正しい（正しい方法） |
| :--------------------- | :-------------------------------------------- | :---------------------------------------------------------- |
| **フラグへのアクセス** | `form.field.valid()` | `form.field().valid()` |
| **値へのアクセス** | `form.field.value()` | `form.field().value()` |
| **値のセット** | `form.field.set(x)` | モデルSignalを更新: `this.model.update(...)` |
| **フォームルートのフラグ** | `form.invalid()` | `form().invalid()` |
| **二重呼び出し** | `form.field()()` | `form.field().value()` |
| **ルールのコンテキスト** | `({ touched }) => touched()` | `({ state }) => state.touched()` |
| **パスの呼び出し** | `applyWhen(p.foo, () => p.foo() === 'x')` | `applyWhen(p.foo, ({ valueOf }) => valueOf(p.foo) === 'x')` |
| **applyWhenの引数** | `applyWhen(condition, () => {...})` | `applyWhen(path, condition, schemaFn)` — 3つの引数が必要 |
| **配列のlength** | `form.items().length` | `form.items.length`（構造的） |
| **複数選択配列** | `<select [formField]="form.tags">` (string[]) | 配列フィールドにはcheckboxを使用 |
| **readonly属性** | `<input readonly [formField]>` | スキーマで `readonly()` ルールを使用 |
| **min/max属性** | `<input min="1" max="10">` | スキーマで `min()`、`max()` ルールを使用 |
| **valueバインディング** | `<input [value]="val">` | `[formField]` と `[value]` を同時に使用しない |
| **whenオプション** | `pattern(p.x, /.../, {when: ...})` | `when` は `required()` にのみ機能する |
| **送信コールバック** | `submit(form, () => { ... })` | `submit(form, async () => { ... })` |
| **非同期params** | `params: s.field` | `params: ({ value }) => value()` |
| **非同期onError** | `onError` を省略 | `validateAsync` では `onError` は必須 |
| **resource() API** | `request: signal` | `params: signal` |
| **applyEachの引数** | `applyEach(s.items, (item, index) => ...)` | `applyEach(s.items, (item) => ...)` |
| **ネストした @for** | `$parent.$index` | `let outerIndex = $index` を使用 |
| **FormStateのインポート** | `import { FormState }` | `FormState` は存在しない。`FieldState` を使用 |
| **モデルのnull** | `signal({ name: null })` | `signal({ name: '' })` または `signal({ age: 0 })` |
| **validateの構文** | `validate(s.field, { value } => ...)` | `validate(s.field, ({ value }) => ...)` |
| **チェックボックス配列** | `[formField]="form.tags"` (string[]) | チェックボックスは `boolean` にのみバインドできる |

## 大規模フォームの例

### `src/app/app.ts`

```ts
import {Component, signal, ChangeDetectionStrategy} from '@angular/core';
import {
  form,
  FormField,
  submit,
  required,
  email,
  min,
  hidden,
  applyEach,
  validate,
} from '@angular/forms/signals';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [FormField],
  templateUrl: './app.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class App {
  model = signal({
    personalInfo: {
      firstName: '',
      lastName: '',
      email: '',
      age: 0,
    },
    tripDetails: {
      destination: 'Mars',
      launchDate: '',
    },
    package: {
      tier: 'economy',
      extras: [] as string[],
    },
    companions: [] as Array<{name: string; relation: string}>,
  });

  bookingForm = form(this.model, (s) => {
    required(s.personalInfo.firstName, {message: 'First name is required'});
    required(s.personalInfo.lastName, {message: 'Last name is required'});
    required(s.personalInfo.email, {message: 'Email is required'});
    email(s.personalInfo.email, {message: 'Invalid email address'});
    required(s.personalInfo.age, {message: 'Age is required'});
    min(s.personalInfo.age, 18, {message: 'Must be at least 18'});

    required(s.tripDetails.destination);
    required(s.tripDetails.launchDate);
    validate(s.tripDetails.launchDate, ({value}) => {
      const date = new Date(value());
      if (isNaN(date.getTime())) return undefined;
      const today = new Date();
      if (date < today) {
        return {kind: 'pastData', message: 'Launch date must be in the future'};
      }
      return undefined;
    });

    // valueOf は他のフィールドの値をルール内でアクセスするために使用する
    hidden(s.package.extras, ({valueOf}) => valueOf(s.package.tier) === 'economy');

    applyEach(s.companions, (companion) => {
      required(companion.name, {message: 'Companion name required'});
      required(companion.relation, {message: 'Relation required'});
    });
  });

  addCompanion() {
    this.model.update((m) => ({
      ...m,
      companions: [...m.companions, {name: '', relation: ''}],
    }));
  }

  removeCompanion(index: number) {
    this.model.update((m) => ({
      ...m,
      companions: m.companions.filter((_, i) => i !== index),
    }));
  }

  onSubmit() {
    // 重要: 送信コールバックは必ずasyncにすること
    submit(this.bookingForm, async () => {
      console.log('Booking Confirmed:', this.model());
      // 非同期処理が必要な場合:
      // await this.apiService.save(this.model());
    });
  }
}
```

### `src/app/app.html`

```html
<form (submit)="onSubmit(); $event.preventDefault()">
  <h1>Interstellar Booking</h1>

  <section>
    <h2>Personal Info</h2>

    <label>
      First Name
      <input [formField]="bookingForm.personalInfo.firstName" />
      @if (bookingForm.personalInfo.firstName().touched() &&
      bookingForm.personalInfo.firstName().errors().length) {
      <span>{{ bookingForm.personalInfo.firstName().errors()[0].message }}</span>
      }
    </label>

    <label>
      Last Name
      <input [formField]="bookingForm.personalInfo.lastName" />
      @if (bookingForm.personalInfo.lastName().touched() &&
      bookingForm.personalInfo.lastName().errors().length) {
      <span>{{ bookingForm.personalInfo.lastName().errors()[0].message }}</span>
      }
    </label>

    <label>
      Email
      <input type="email" [formField]="bookingForm.personalInfo.email" />
      @if (bookingForm.personalInfo.email().touched() &&
      bookingForm.personalInfo.email().errors().length) {
      <span>{{ bookingForm.personalInfo.email().errors()[0].message }}</span>
      }
    </label>

    <label>
      Age
      <input type="number" [formField]="bookingForm.personalInfo.age" />
      @if (bookingForm.personalInfo.age().touched() &&
      bookingForm.personalInfo.age().errors().length) {
      <span>{{ bookingForm.personalInfo.age().errors()[0].message }}</span>
      }
    </label>
  </section>

  <section>
    <h2>Trip Details</h2>

    <label>
      Destination
      <select [formField]="bookingForm.tripDetails.destination">
        <option value="Mars">Mars</option>
        <option value="Moon">Moon</option>
        <option value="Titan">Titan</option>
      </select>
    </label>

    <label>
      Launch Date
      <input type="date" [formField]="bookingForm.tripDetails.launchDate" />
      @if (bookingForm.tripDetails.launchDate().touched() &&
      bookingForm.tripDetails.launchDate().errors().length) {
      <span>{{ bookingForm.tripDetails.launchDate().errors()[0].message }}</span>
      }
    </label>
  </section>

  <section>
    <h2>Package</h2>

    <label>
      <input type="radio" value="economy" [formField]="bookingForm.package.tier" />
      Economy
    </label>
    <label>
      <input type="radio" value="business" [formField]="bookingForm.package.tier" />
      Business
    </label>
    <label>
      <input type="radio" value="first" [formField]="bookingForm.package.tier" />
      First Class
    </label>

    @if (!bookingForm.package.extras().hidden()) {
    <div>
      <h3>Extras</h3>
      <!-- 配列の複数選択にはselect multipleを使用すること -->
      <select multiple [formField]="bookingForm.package.extras">
        <option value="wifi">WiFi</option>
        <option value="gym">Gym</option>
      </select>
    </div>
    }
  </section>

  <section>
    <h2>Companions</h2>
    <button type="button" (click)="addCompanion()">Add Companion</button>

    @for (companion of bookingForm.companions; track $index) {
    <div>
      <input [formField]="companion.name" placeholder="Name" />
      @if (companion.name().touched() && companion.name().errors().length) {
      <span>{{ companion.name().errors()[0].message }}</span>
      }

      <input [formField]="companion.relation" placeholder="Relation" />
      @if (companion.relation().touched() && companion.relation().errors().length) {
      <span>{{ companion.relation().errors()[0].message }}</span>
      }

      <button type="button" (click)="removeCompanion($index)">Remove</button>
    </div>
    }
  </section>

  <button [disabled]="bookingForm().invalid()">Submit</button>
</form>
```

## ビルドエラーからの回復

ビルドエラーが発生した場合、以下がよくある修正方法です。

### `Property 'value' does not exist on type 'FieldTree'`

**問題**: 最初に呼び出さずにフィールドから直接 `.value()` にアクセスしている。

```ts
// 間違い
const val = this.form.field.value();
// 正しい
const val = this.form.field().value();
```

### `Property 'set' does not exist on type 'FieldTree'`

**問題**: フォームツリーに値をセットしようとしている。Signal Formsはモデル駆動です。

```ts
// 間違い
this.form.address.street.set('Main St');
// 正しい — 代わりにモデルSignalを更新する
this.model.update((m) => ({...m, address: {...m.address, street: 'Main St'}}));
```

### `Type 'string[]' is not assignable to type 'string'`

**問題**: 配列フィールドを単一値の `<select>` に `[formField]` でバインドしている。

```html
<!-- 間違い — assignees は string[] だが、selectはstringを期待する -->
<select [formField]="form.assignees">
  ...
</select>

<!-- 正しい — 配列フィールドには select multiple を使用する -->
<select multiple [formField]="form.assignees">
  <option value="us">US</option>
</select>
```
