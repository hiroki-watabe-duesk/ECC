---
name: perl-testing
description: Test2::V0、Test::More、prove ランナー、モック、Devel::Cover によるカバレッジ、TDD 手法を使用した Perl テストパターン。
origin: ECC
---

# Perl テストパターン

Test2::V0、Test::More、prove、TDD 手法を使用した Perl アプリケーションの包括的なテスト戦略。

## アクティブにするタイミング

- 新しい Perl コードを書くとき（TDD に従う: レッド、グリーン、リファクタリング）
- Perl モジュールまたはアプリケーションのテストスイートを設計するとき
- Perl テストカバレッジをレビューするとき
- Perl テストインフラをセットアップするとき
- Test::More から Test2::V0 へテストを移行するとき
- 失敗している Perl テストをデバッグするとき

## TDD ワークフロー

常に RED-GREEN-REFACTOR サイクルに従う。

```perl
# Step 1: RED — Write a failing test
# t/unit/calculator.t
use v5.36;
use Test2::V0;

use lib 'lib';
use Calculator;

subtest 'addition' => sub {
    my $calc = Calculator->new;
    is($calc->add(2, 3), 5, 'adds two numbers');
    is($calc->add(-1, 1), 0, 'handles negatives');
};

done_testing;

# Step 2: GREEN — Write minimal implementation
# lib/Calculator.pm
package Calculator;
use v5.36;
use Moo;

sub add($self, $a, $b) {
    return $a + $b;
}

1;

# Step 3: REFACTOR — Improve while tests stay green
# Run: prove -lv t/unit/calculator.t
```

## Test::More の基礎

標準的な Perl テストモジュール — 広く使われており、コアに同梱されている。

### 基本的なアサーション

```perl
use v5.36;
use Test::More;

# Plan upfront or use done_testing
# plan tests => 5;  # Fixed plan (optional)

# Equality
is($result, 42, 'returns correct value');
isnt($result, 0, 'not zero');

# Boolean
ok($user->is_active, 'user is active');
ok(!$user->is_banned, 'user is not banned');

# Deep comparison
is_deeply(
    $got,
    { name => 'Alice', roles => ['admin'] },
    'returns expected structure'
);

# Pattern matching
like($error, qr/not found/i, 'error mentions not found');
unlike($output, qr/password/, 'output hides password');

# Type check
isa_ok($obj, 'MyApp::User');
can_ok($obj, 'save', 'delete');

done_testing;
```

### SKIP と TODO

```perl
use v5.36;
use Test::More;

# Skip tests conditionally
SKIP: {
    skip 'No database configured', 2 unless $ENV{TEST_DB};

    my $db = connect_db();
    ok($db->ping, 'database is reachable');
    is($db->version, '15', 'correct PostgreSQL version');
}

# Mark expected failures
TODO: {
    local $TODO = 'Caching not yet implemented';
    is($cache->get('key'), 'value', 'cache returns value');
}

done_testing;
```

## Test2::V0 モダンフレームワーク

Test2::V0 は Test::More の現代的な代替品 — より豊富なアサーション、より良い診断、拡張可能。

### なぜ Test2 を使うのか？

- ハッシュ/配列ビルダーによる優れた深い比較
- 失敗時のより良い診断出力
- よりクリーンなスコープを持つサブテスト
- Test2::Tools::* プラグインによる拡張性
- Test::More テストとの後方互換性

### ビルダーによる深い比較

```perl
use v5.36;
use Test2::V0;

# Hash builder — check partial structure
is(
    $user->to_hash,
    hash {
        field name  => 'Alice';
        field email => match(qr/\@example\.com$/);
        field age   => validator(sub { $_ >= 18 });
        # Ignore other fields
        etc();
    },
    'user has expected fields'
);

# Array builder
is(
    $result,
    array {
        item 'first';
        item match(qr/^second/);
        item DNE();  # Does Not Exist — verify no extra items
    },
    'result matches expected list'
);

# Bag — order-independent comparison
is(
    $tags,
    bag {
        item 'perl';
        item 'testing';
        item 'tdd';
    },
    'has all required tags regardless of order'
);
```

### サブテスト

```perl
use v5.36;
use Test2::V0;

subtest 'User creation' => sub {
    my $user = User->new(name => 'Alice', email => 'alice@example.com');
    ok($user, 'user object created');
    is($user->name, 'Alice', 'name is set');
    is($user->email, 'alice@example.com', 'email is set');
};

subtest 'User validation' => sub {
    my $warnings = warns {
        User->new(name => '', email => 'bad');
    };
    ok($warnings, 'warns on invalid data');
};

done_testing;
```

### Test2 による例外テスト

```perl
use v5.36;
use Test2::V0;

# Test that code dies
like(
    dies { divide(10, 0) },
    qr/Division by zero/,
    'dies on division by zero'
);

# Test that code lives
ok(lives { divide(10, 2) }, 'division succeeds') or note($@);

# Combined pattern
subtest 'error handling' => sub {
    ok(lives { parse_config('valid.json') }, 'valid config parses');
    like(
        dies { parse_config('missing.json') },
        qr/Cannot open/,
        'missing file dies with message'
    );
};

done_testing;
```

## テスト構成と prove

### ディレクトリ構造

```text
t/
├── 00-load.t              # Verify modules compile
├── 01-basic.t             # Core functionality
├── unit/
│   ├── config.t           # Unit tests by module
│   ├── user.t
│   └── util.t
├── integration/
│   ├── database.t
│   └── api.t
├── lib/
│   └── TestHelper.pm      # Shared test utilities
└── fixtures/
    ├── config.json        # Test data files
    └── users.csv
```

### prove コマンド

```bash
# Run all tests
prove -l t/

# Verbose output
prove -lv t/

# Run specific test
prove -lv t/unit/user.t

# Recursive search
prove -lr t/

# Parallel execution (8 jobs)
prove -lr -j8 t/

# Run only failing tests from last run
prove -l --state=failed t/

# Colored output with timer
prove -l --color --timer t/

# TAP output for CI
prove -l --formatter TAP::Formatter::JUnit t/ > results.xml
```

### .proverc 設定

```text
-l
--color
--timer
-r
-j4
--state=save
```

## フィクスチャとセットアップ/ティアダウン

### サブテストの分離

```perl
use v5.36;
use Test2::V0;
use File::Temp qw(tempdir);
use Path::Tiny;

subtest 'file processing' => sub {
    # Setup
    my $dir = tempdir(CLEANUP => 1);
    my $file = path($dir, 'input.txt');
    $file->spew_utf8("line1\nline2\nline3\n");

    # Test
    my $result = process_file("$file");
    is($result->{line_count}, 3, 'counts lines');

    # Teardown happens automatically (CLEANUP => 1)
};
```

### 共有テストヘルパー

再利用可能なヘルパーは `t/lib/TestHelper.pm` に置き、`use lib 't/lib'` で読み込む。`Exporter` を介して `create_test_db()`、`create_temp_dir()`、`fixture_path()` などのファクトリー関数をエクスポートする。

## モック

### Test::MockModule

```perl
use v5.36;
use Test2::V0;
use Test::MockModule;

subtest 'mock external API' => sub {
    my $mock = Test::MockModule->new('MyApp::API');

    # Good: Mock returns controlled data
    $mock->mock(fetch_user => sub ($self, $id) {
        return { id => $id, name => 'Mock User', email => 'mock@test.com' };
    });

    my $api = MyApp::API->new;
    my $user = $api->fetch_user(42);
    is($user->{name}, 'Mock User', 'returns mocked user');

    # Verify call count
    my $call_count = 0;
    $mock->mock(fetch_user => sub { $call_count++; return {} });
    $api->fetch_user(1);
    $api->fetch_user(2);
    is($call_count, 2, 'fetch_user called twice');

    # Mock is automatically restored when $mock goes out of scope
};

# Bad: Monkey-patching without restoration
# *MyApp::API::fetch_user = sub { ... };  # NEVER — leaks across tests
```

軽量なモックオブジェクトには `Test::MockObject` を使用して、`->mock()` で注入可能なテストダブルを作成し、`->called_ok()` で呼び出しを確認する。

## Devel::Cover によるカバレッジ

### カバレッジの実行

```bash
# Basic coverage report
cover -test

# Or step by step
perl -MDevel::Cover -Ilib t/unit/user.t
cover

# HTML report
cover -report html
open cover_db/coverage.html

# Specific thresholds
cover -test -report text | grep 'Total'

# CI-friendly: fail under threshold
cover -test && cover -report text -select '^lib/' \
  | perl -ne 'if (/Total.*?(\d+\.\d+)/) { exit 1 if $1 < 80 }'
```

### インテグレーションテスト

データベーステストにはインメモリ SQLite を使用し、API テストには HTTP::Tiny をモックする。

```perl
use v5.36;
use Test2::V0;
use DBI;

subtest 'database integration' => sub {
    my $dbh = DBI->connect('dbi:SQLite:dbname=:memory:', '', '', {
        RaiseError => 1,
    });
    $dbh->do('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');

    $dbh->prepare('INSERT INTO users (name) VALUES (?)')->execute('Alice');
    my $row = $dbh->selectrow_hashref('SELECT * FROM users WHERE name = ?', undef, 'Alice');
    is($row->{name}, 'Alice', 'inserted and retrieved user');
};

done_testing;
```

## ベストプラクティス

### すべきこと

- **TDD に従う**: 実装前にテストを書く（レッド・グリーン・リファクタリング）
- **Test2::V0 を使う**: モダンなアサーション、より良い診断
- **サブテストを使う**: 関連するアサーションをグループ化し、状態を分離する
- **外部依存関係をモックする**: ネットワーク、データベース、ファイルシステム
- **`prove -l` を使う**: `@INC` に常に lib/ を含める
- **テストに明確な名前を付ける**: `'user login with invalid password fails'`
- **エッジケースをテストする**: 空文字列、undef、ゼロ、境界値
- **80% 以上のカバレッジを目指す**: ビジネスロジックのパスに集中する
- **テストを速く保つ**: I/O をモックし、インメモリデータベースを使用する

### してはいけないこと

- **実装をテストしない**: 内部ではなく動作と出力をテストする
- **サブテスト間で状態を共有しない**: 各サブテストは独立すべき
- **`done_testing` を省略しない**: 全ての計画されたテストが実行されたことを保証する
- **過剰なモックをしない**: テスト対象コードではなく境界のみをモックする
- **新規プロジェクトに `Test::More` を使わない**: Test2::V0 を優先する
- **テストの失敗を無視しない**: マージ前に全てのテストがパスしなければならない
- **CPAN モジュールをテストしない**: ライブラリが正しく動作することを信頼する
- **壊れやすいテストを書かない**: 過度に具体的な文字列マッチングを避ける

## クイックリファレンス

| タスク | コマンド / パターン |
|---|---|
| 全テストを実行する | `prove -lr t/` |
| 1つのテストを冗長モードで実行する | `prove -lv t/unit/user.t` |
| 並列テスト実行 | `prove -lr -j8 t/` |
| カバレッジレポート | `cover -test && cover -report html` |
| 等値テスト | `is($got, $expected, 'label')` |
| 深い比較 | `is($got, hash { field k => 'v'; etc() }, 'label')` |
| 例外テスト | `like(dies { ... }, qr/msg/, 'label')` |
| 例外なしテスト | `ok(lives { ... }, 'label')` |
| メソッドをモックする | `Test::MockModule->new('Pkg')->mock(m => sub { ... })` |
| テストをスキップする | `SKIP: { skip 'reason', $count unless $cond; ... }` |
| TODO テスト | `TODO: { local $TODO = 'reason'; ... }` |

## よくある落とし穴

### `done_testing` を忘れる

```perl
# Bad: Test file runs but doesn't verify all tests executed
use Test2::V0;
is(1, 1, 'works');
# Missing done_testing — silent bugs if test code is skipped

# Good: Always end with done_testing
use Test2::V0;
is(1, 1, 'works');
done_testing;
```

### `-l` フラグの欠如

```bash
# Bad: Modules in lib/ not found
prove t/unit/user.t
# Can't locate MyApp/User.pm in @INC

# Good: Include lib/ in @INC
prove -l t/unit/user.t
```

### 過剰なモック

テスト対象コードではなく*依存関係*をモックする。テストがモックに指示した内容を返すことだけを検証するなら、何もテストしていない。

### テストの汚染

サブテスト内では `our` ではなく `my` 変数を使用する — テスト間で状態が漏れるのを防ぐ。

**覚えておくこと**: テストはセーフティネット。速く、集中的で、独立したものに保つ。新規プロジェクトには Test2::V0 を、実行には prove を、説明責任には Devel::Cover を使用する。
