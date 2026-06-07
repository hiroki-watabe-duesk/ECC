---
name: healthcare-eval-harness
description: 医療アプリケーションのデプロイメントに向けた患者安全評価ハーネス。CDSS精度、PHI露出、臨床ワークフロー整合性、統合コンプライアンスの自動テストスイート。安全性の失敗時はデプロイをブロックします。
origin: Health1 Super Speciality Hospitals — contributed by Dr. Keyur Patel
version: "1.0.0"
---

# Healthcare Eval Harness — 患者安全検証

医療アプリケーションのデプロイメントに向けた自動検証システム。CRITICAL 判定の失敗が1件でもあるとデプロイはブロックされます。患者の安全は絶対です。

> **注意:** サンプルでは Jest をリファレンステストランナーとして使用しています。お使いのフレームワーク（Vitest、pytest、PHPUnit 等）に合わせてコマンドを適宜変更してください。テストカテゴリと合格閾値はフレームワークに依存しません。

## いつ使うか

- EMR/EHR アプリケーションのデプロイ前
- CDSS ロジック（薬物相互作用、用量バリデーション、スコアリング）を変更した後
- 患者データに触れるデータベーススキーマを変更した後
- 認証やアクセス制御を変更した後
- 医療アプリの CI/CD パイプライン設定時
- 臨床モジュールのマージコンフリクト解消後

## 仕組み

評価ハーネスは5つのテストカテゴリを順番に実行します。最初の3つ（CDSS 精度、PHI 露出、データ整合性）は CRITICAL ゲートで、合格率 100% が必要です。1件でも失敗するとデプロイがブロックされます。残り2つ（臨床ワークフロー、統合）は HIGH ゲートで、95% 以上の合格率が必要です。

各カテゴリは Jest のテストパスパターンに対応します。CI パイプラインは CRITICAL ゲートに `--bail`（最初の失敗で停止）を指定して実行し、`--coverage --coverageThreshold` でカバレッジ閾値を強制します。

### 評価カテゴリ

**1. CDSS 精度（CRITICAL — 100% 必須）**

すべての臨床判断支援ロジックをテストします。薬物相互作用ペア（双方向）、用量バリデーションルール、公開された仕様に対する臨床スコアリング、偽陰性なし、サイレントな失敗なし。

```bash
npx jest --testPathPattern='tests/cdss' --bail --ci --coverage
```

**2. PHI 露出（CRITICAL — 100% 必須）**

保護された医療情報の漏洩をテストします。API エラーレスポンス、コンソール出力、URL パラメータ、ブラウザストレージ、施設間分離、未認証アクセス、サービスロールキーの不在。

```bash
npx jest --testPathPattern='tests/security/phi' --bail --ci
```

**3. データ整合性（CRITICAL — 100% 必須）**

臨床データの安全性をテストします。ロックされたエンカウンター、監査証跡エントリ、カスケード削除保護、同時編集処理、孤立レコードなし。

```bash
npx jest --testPathPattern='tests/data-integrity' --bail --ci
```

**4. 臨床ワークフロー（HIGH — 95% 以上必須）**

エンドツーエンドのフローをテストします。エンカウンターライフサイクル、テンプレートレンダリング、薬剤セット、薬剤・診断検索、処方箋 PDF、レッドフラグアラート。

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$tmp_json" || true
total=$(jq '.numTotalTests // 0' "$tmp_json")
passed=$(jq '.numPassedTests // 0' "$tmp_json")
if [ "$total" -eq 0 ]; then
  echo "No clinical tests found" >&2
  exit 1
fi
rate=$(echo "scale=2; $passed * 100 / $total" | bc)
echo "Clinical pass rate: ${rate}% ($passed/$total)"
```

**5. 統合コンプライアンス（HIGH — 95% 以上必須）**

外部システムをテストします。HL7 メッセージ解析（v2.x）、FHIR バリデーション、検査結果マッピング、不正なメッセージ処理。

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/integration' --ci --json --outputFile="$tmp_json" || true
total=$(jq '.numTotalTests // 0' "$tmp_json")
passed=$(jq '.numPassedTests // 0' "$tmp_json")
if [ "$total" -eq 0 ]; then
  echo "No integration tests found" >&2
  exit 1
fi
rate=$(echo "scale=2; $passed * 100 / $total" | bc)
echo "Integration pass rate: ${rate}% ($passed/$total)"
```

### 合否マトリクス

| カテゴリ | 閾値 | 失敗時 |
|----------|-----------|------------|
| CDSS 精度 | 100% | **デプロイをブロック** |
| PHI 露出 | 100% | **デプロイをブロック** |
| データ整合性 | 100% | **デプロイをブロック** |
| 臨床ワークフロー | 95% 以上 | 警告、レビューを経て許可 |
| 統合 | 95% 以上 | 警告、レビューを経て許可 |

### CI/CD 統合

```yaml
name: Healthcare Safety Gate
on: [push, pull_request]

jobs:
  safety-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci

      # CRITICAL ゲート — 100% 必須、最初の失敗で停止
      - name: CDSS Accuracy
        run: npx jest --testPathPattern='tests/cdss' --bail --ci --coverage --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80}}'

      - name: PHI Exposure Check
        run: npx jest --testPathPattern='tests/security/phi' --bail --ci

      - name: Data Integrity
        run: npx jest --testPathPattern='tests/data-integrity' --bail --ci

      # HIGH ゲート — 95% 以上必須、カスタム閾値チェック
      # HIGH ゲート — 95% 以上必須
      - name: Clinical Workflows
        run: |
          TMP_JSON=$(mktemp)
          npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$TMP_JSON" || true
          TOTAL=$(jq '.numTotalTests // 0' "$TMP_JSON")
          PASSED=$(jq '.numPassedTests // 0' "$TMP_JSON")
          if [ "$TOTAL" -eq 0 ]; then
            echo "::error::No clinical tests found"; exit 1
          fi
          RATE=$(echo "scale=2; $PASSED * 100 / $TOTAL" | bc)
          echo "Pass rate: ${RATE}% ($PASSED/$TOTAL)"
          if (( $(echo "$RATE < 95" | bc -l) )); then
            echo "::warning::Clinical pass rate ${RATE}% below 95%"
          fi

      - name: Integration Compliance
        run: |
          TMP_JSON=$(mktemp)
          npx jest --testPathPattern='tests/integration' --ci --json --outputFile="$TMP_JSON" || true
          TOTAL=$(jq '.numTotalTests // 0' "$TMP_JSON")
          PASSED=$(jq '.numPassedTests // 0' "$TMP_JSON")
          if [ "$TOTAL" -eq 0 ]; then
            echo "::error::No integration tests found"; exit 1
          fi
          RATE=$(echo "scale=2; $PASSED * 100 / $TOTAL" | bc)
          echo "Pass rate: ${RATE}% ($PASSED/$TOTAL)"
          if (( $(echo "$RATE < 95" | bc -l) )); then
            echo "::warning::Integration pass rate ${RATE}% below 95%"
          fi
```

### アンチパターン

- 「前回通過したから」という理由で CDSS テストをスキップする
- CRITICAL 閾値を 100% 未満に設定する
- CRITICAL テストスイートで `--no-bail` を使用する
- 統合テストで CDSS エンジンをモック化する（実際のロジックをテストしなければならない）
- 安全ゲートがレッドの状態でデプロイを許可する
- CDSS スイートで `--coverage` なしでテストを実行する

## 使用例

### 例 1: すべての CRITICAL ゲートをローカルで実行する

```bash
npx jest --testPathPattern='tests/cdss' --bail --ci --coverage && \
npx jest --testPathPattern='tests/security/phi' --bail --ci && \
npx jest --testPathPattern='tests/data-integrity' --bail --ci
```

### 例 2: HIGH ゲートの合格率を確認する

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$tmp_json" || true
jq '{
  passed: (.numPassedTests // 0),
  total: (.numTotalTests // 0),
  rate: (if (.numTotalTests // 0) == 0 then 0 else ((.numPassedTests // 0) / (.numTotalTests // 1) * 100) end)
}' "$tmp_json"
# 期待値: { "passed": 21, "total": 22, "rate": 95.45 }
```

### 例 3: 評価レポート

```
## Healthcare Eval: 2026-03-27 [commit abc1234]

### Patient Safety: PASS

| Category | Tests | Pass | Fail | Status |
|----------|-------|------|------|--------|
| CDSS Accuracy | 39 | 39 | 0 | PASS |
| PHI Exposure | 8 | 8 | 0 | PASS |
| Data Integrity | 12 | 12 | 0 | PASS |
| Clinical Workflow | 22 | 21 | 1 | 95.5% PASS |
| Integration | 6 | 6 | 0 | PASS |

### Coverage: 84% (target: 80%+)
### Verdict: SAFE TO DEPLOY
```
