---
name: network-config-validation
description: ルーターおよびスイッチ設定のデプロイ前チェック。危険なコマンド、重複アドレス、サブネットの重複、参照の陳腐化、管理プレーンのリスク、IOS スタイルのセキュリティ衛生を含みます。
origin: community
---

# ネットワーク設定バリデーション

このスキルを使用して、変更ウィンドウの前、または自動化実行が本番デバイスに触れる前にネットワーク設定をレビューします。

## いつ使うか

- デプロイ前に Cisco IOS または IOS-XE スタイルのスニペットをレビューするとき。
- スクリプトやテンプレートから生成された設定を監査するとき。
- 危険なコマンド、重複 IP アドレス、またはサブネットの重複を探すとき。
- ACL、ルートマップ、プレフィックスリスト、またはライン設定が参照されているが定義されていない場合を確認するとき。
- ネットワーク自動化のための軽量なプリフライトスクリプトを構築するとき。

## 仕組み

設定バリデーションを完全なパーサーではなく、層状の証拠として扱います。正規表現チェックはプリフライトの警告に有用ですが、最終承認にはネットワークエンジニアが意図、プラットフォームの構文、ロールバック手順をレビューする必要があります。

以下の順序でバリデーションを行います。

1. 破壊的なコマンド。
2. 認証情報と管理プレーンの露出。
3. 重複アドレスとサブネットの重複。
4. ACL、ルートマップ、プレフィックスリスト、インターフェースへの陳腐化した参照。
5. NTP、タイムスタンプ、リモートロギング、バナーなどの運用衛生。

## 危険なコマンドの検出

```python
import re

DANGEROUS_PATTERNS: list[tuple[re.Pattern[str], str]] = [
    (re.compile(r"\breload\b", re.I), "reload causes downtime"),
    (re.compile(r"\berase\s+(startup|nvram|flash)", re.I), "erases persistent storage"),
    (re.compile(r"\bformat\b", re.I), "formats a device filesystem"),
    (re.compile(r"\bno\s+router\s+(bgp|ospf|eigrp)\b", re.I), "removes a routing process"),
    (re.compile(r"\bno\s+interface\s+\S+", re.I), "removes interface configuration"),
    (re.compile(r"\baaa\s+new-model\b", re.I), "changes authentication behavior"),
    (re.compile(r"\bcrypto\s+key\s+(zeroize|generate)\b", re.I), "changes device SSH keys"),
]

def find_dangerous_commands(lines: list[str]) -> list[dict[str, str | int]]:
    findings = []
    for line_number, line in enumerate(lines, start=1):
        stripped = line.strip()
        for pattern, reason in DANGEROUS_PATTERNS:
            if pattern.search(stripped):
                findings.append({
                    "line": line_number,
                    "command": stripped,
                    "reason": reason,
                })
    return findings
```

## 重複 IP とサブネットの重複

```python
import ipaddress
import re
from collections import Counter

IP_ADDRESS_RE = re.compile(
    r"^\s*ip address\s+"
    r"(?P<ip>\d{1,3}(?:\.\d{1,3}){3})\s+"
    r"(?P<mask>\d{1,3}(?:\.\d{1,3}){3})\b",
    re.I | re.M,
)

def extract_interfaces(config: str) -> list[dict[str, str]]:
    results = []
    current = None
    for line in config.splitlines():
        if line.startswith("interface "):
            current = line.split(maxsplit=1)[1]
            continue
        match = IP_ADDRESS_RE.match(line)
        if current and match:
            ip = match.group("ip")
            mask = match.group("mask")
            network = ipaddress.ip_interface(f"{ip}/{mask}").network
            results.append({"interface": current, "ip": ip, "network": str(network)})
    return results

def find_duplicate_ips(config: str) -> list[str]:
    ips = [entry["ip"] for entry in extract_interfaces(config)]
    counts = Counter(ips)
    return sorted(ip for ip, count in counts.items() if count > 1)

def find_subnet_overlaps(config: str) -> list[tuple[str, str]]:
    networks = [ipaddress.ip_network(entry["network"]) for entry in extract_interfaces(config)]
    overlaps = []
    for index, left in enumerate(networks):
        for right in networks[index + 1:]:
            if left.overlaps(right):
                overlaps.append((str(left), str(right)))
    return overlaps
```

## 管理プレーンのチェック

VTY ブロックをセクションごとに解析して、アクセスクラスのチェックが無関係な行にまたがらないようにします。

```python
import re

def iter_blocks(config: str, starts_with: str) -> list[str]:
    blocks = []
    current: list[str] = []
    for line in config.splitlines():
        if line.startswith(starts_with):
            if current:
                blocks.append("\n".join(current))
            current = [line]
            continue
        if current:
            if line and not line.startswith(" "):
                blocks.append("\n".join(current))
                current = []
            else:
                current.append(line)
    if current:
        blocks.append("\n".join(current))
    return blocks

def check_vty_blocks(config: str) -> list[str]:
    issues = []
    for block in iter_blocks(config, "line vty"):
        if re.search(r"transport\s+input\s+.*telnet", block, re.I):
            issues.append("VTY allows Telnet; require SSH only.")
        if not re.search(r"\baccess-class\s+\S+\s+in\b", block, re.I):
            issues.append("VTY block has no inbound access-class source restriction.")
        if not re.search(r"\bexec-timeout\s+\d+\s+\d+\b", block, re.I):
            issues.append("VTY block has no explicit exec-timeout.")
    return issues
```

## セキュリティ衛生チェック

```python
SECURITY_PATTERNS = [
    (re.compile(r"\bsnmp-server community\s+(public|private)\b", re.I),
     "default SNMP community configured"),
    (re.compile(r"\bsnmp-server community\s+\S+", re.I),
     "SNMPv2 community string configured; prefer SNMPv3 authPriv"),
    (re.compile(r"\bip ssh version 1\b", re.I),
     "SSH version 1 enabled"),
    (re.compile(r"\benable password\b", re.I),
     "enable password is present; use enable secret"),
    (re.compile(r"\busername\s+\S+\s+password\b", re.I),
     "local username uses password instead of secret"),
]

BEST_PRACTICE_PATTERNS = [
    (re.compile(r"\bntp server\b", re.I), "NTP server"),
    (re.compile(r"\bservice timestamps\b", re.I), "log timestamps"),
    (re.compile(r"\blogging\s+\S+", re.I), "logging destination or buffer"),
    (re.compile(r"\bsnmp-server group\s+\S+\s+v3\s+priv\b", re.I), "SNMPv3 authPriv group"),
    (re.compile(r"\bbanner\s+(login|motd)\b", re.I), "login banner"),
]

def check_security(config: str) -> list[str]:
    return [message for pattern, message in SECURITY_PATTERNS if pattern.search(config)]

def check_missing_hygiene(config: str) -> list[str]:
    return [
        f"Missing {description}"
        for pattern, description in BEST_PRACTICE_PATTERNS
        if not pattern.search(config)
    ]
```

## 使用例

### 変更ウィンドウのプリフライト

1. 貼り付けるスニペットに対して危険なコマンドチェックを実行する。
2. 完全な候補設定に対して重複 IP とサブネットの重複チェックを実行する。
3. 参照されているすべての ACL、ルートマップ、プレフィックスリストが存在することを確認する。
4. 管理プレーンの変更を行う前に、ロールバックコマンドと帯域外アクセスを確認する。

### 自動化プリフライト

生成された設定を Netmiko、NAPALM、Ansible、またはベンダー API の自動化がプッシュする前に、バリデーションをブロッキングゲートとして使用します。危険なコマンドと認証情報では失敗し、変更スコープ外のベストプラクティスのギャップでは警告します。

## アンチパターン

- 正規表現バリデーションをデバイスパーサーとして扱う。
- ドライランの差分なしに生成された設定を適用する。
- SNMPv2 コミュニティ文字列を監視要件として推奨する。
- 無関係なセクションをまたぐ正規表現で VTY ブロックをチェックする。
- カウンター/ログを読む代わりに ACL を無効にしてファイアウォールの動作をテストする。

## 関連情報

- Agent: `network-config-reviewer`
- Agent: `network-troubleshooter`
- Skill: `network-interface-health`
