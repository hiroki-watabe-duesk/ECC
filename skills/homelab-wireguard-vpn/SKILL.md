---
name: homelab-wireguard-vpn
description: WireGuard VPNサーバーのセットアップ、ピア設定、鍵生成、スプリットトンネリングとフルトンネルルーティングの比較、モバイルおよびノートPCクライアントからホームネットワークへのリモートアクセス。
origin: community
---

# Homelab WireGuard VPN

WireGuardは高速でモダンなVPNプロトコルである。ホームネットワークへのリモートアクセスに適した選択肢で、OpenVPNより設定が簡単で、ほとんどの代替手段より高速である。

以下の設定例はよくある構成を示している。システムに適用する前に、特にiptablesの転送ルールと鍵ファイルのパーミッションに関する各コマンドを確認し、メンテナンスウィンドウ内で変更を行うこと。

## 使用するタイミング

- Raspberry Pi、Linuxホスト、pfSense、またはルーターにWireGuardサーバーをセットアップするとき
- WireGuardの鍵ペアを生成してピア設定ファイルを作成するとき
- スマートフォンやノートPCからホームネットワークへのリモートアクセスを設定するとき
- スプリットトンネリング（ホームトラフィックのみルーティング）とフルトンネル（全トラフィックをルーティング）の違いを説明するとき
- 接続が確立しないWireGuard接続をトラブルシューティングするとき
- 複数クライアントのピア設定生成を自動化するとき

## WireGuardの仕組み

```
スマートフォン（WireGuardクライアント）
    │
    │  暗号化されたUDPトンネル（ポート51820）
    │
ホームルーター（WireGuardサーバー — パブリックIPまたはDDNSが必要）
    │
    ホームネットワーク（192.168.1.0/24、NAS、Piなど）

各デバイスが鍵ペア（公開鍵 + 秘密鍵）を持つ。
サーバーは各クライアントの公開鍵を知っている。
クライアントはサーバーの公開鍵 + エンドポイント（IP:ポート）を知っている。
中央サーバーや認証機関なしでエンドツーエンド暗号化が行われる。
```

## サーバーセットアップ（Linux）

```bash
# WireGuardをインストール
sudo apt update && sudo apt install wireguard -y

# サーバーの鍵ペアを生成 — 最初からプライベートパーミッションでファイルを作成
sudo mkdir -p /etc/wireguard
sudo sh -c 'umask 077; wg genkey > /etc/wireguard/server_private.key'
sudo sh -c 'wg pubkey < /etc/wireguard/server_private.key > /etc/wireguard/server_public.key'

# サーバー設定を書き込む — 実際の秘密鍵の値に置き換える
# 秘密鍵はバージョン管理に保存したり共有したりしないこと
sudo tee /etc/wireguard/wg0.conf << 'EOF'
[Interface]
Address = 10.8.0.1/24              # VPNサブネット — サーバーは .1 を取得
ListenPort = 51820
PrivateKey = <paste_server_private_key_here>

# スコープを絞った転送ルール：VPNトラフィックの入出力を許可するが、FORWARDを全許可しない
PostUp   = iptables -A FORWARD -i wg0 -o eth0 -j ACCEPT
PostUp   = iptables -A FORWARD -i eth0 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostUp   = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -o eth0 -j ACCEPT
PostDown = iptables -D FORWARD -i eth0 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# スマートフォン — 実際のスマートフォンの公開鍵に置き換える
PublicKey = <phone_public_key>
AllowedIPs = 10.8.0.2/32

[Peer]
# ノートPC — 実際のノートPCの公開鍵に置き換える
PublicKey = <laptop_public_key>
AllowedIPs = 10.8.0.3/32
EOF
sudo chmod 600 /etc/wireguard/wg0.conf

# eth0 を実際のアウトバウンドインターフェース名に置き換える
# 確認方法: ip route show default

# IPフォワーディングを有効化（サーバー経由のトラフィックルーティングに必要）
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system

# WireGuardを起動し、起動時に自動開始するよう設定
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

## クライアント設定

```bash
# 各クライアントデバイス用に固有の鍵ペアを生成する
# クライアント上で実行するか、サーバー上で生成して秘密鍵を安全に転送する — 平文での転送は不可
umask 077
wg genkey | tee phone_private.key | wg pubkey > phone_public.key

# クライアント設定ファイル（phone_wg0.conf）:
[Interface]
PrivateKey = <phone_private_key>
Address = 10.8.0.2/32
DNS = 192.168.1.2                  # オプション: トンネル経由のDNSにPi-holeを使用

[Peer]
PublicKey = <server_public_key>
Endpoint = your-home-ip.ddns.net:51820  # パブリックIPまたはDDNSホスト名
AllowedIPs = 192.168.1.0/24            # スプリットトンネル: ホームネットワークのトラフィックのみ
# AllowedIPs = 0.0.0.0/0, ::/0        # フルトンネル: 全トラフィックをVPN経由

PersistentKeepalive = 25              # NATホールを開いたままにする（モバイルクライアントに必要）
```

## スプリットトンネルとフルトンネルの比較

```
# スプリットトンネル: AllowedIPs = 192.168.1.0/24
  ホームネットワーク宛てのトラフィックのみVPNを通過する。
  インターネットトラフィック（YouTube、Spotify）は直接通信 — モバイルでのパフォーマンスが向上する。
  適したケース: 「どこからでもNASやPiにアクセスしたいだけ」という場合。

# フルトンネル: AllowedIPs = 0.0.0.0/0, ::/0
  全トラフィックがホームのインターネット接続を通過する。
  用途: ホームDNS/Pi-holeの広告ブロックを活用する。
  デメリット: ホームのアップロード速度がどこでもボトルネックになる。

# マルチサブネットのスプリットトンネル（ホームラボでの最も一般的なケース）:
  AllowedIPs = 192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24, 10.8.0.0/24
  全VLANをトンネル経由でルーティングし、インターネットは直接接続を維持する。
```

## 鍵生成とピア管理

```python
import subprocess

def generate_keypair() -> tuple[str, str]:
    """WireGuardの鍵ペアを生成する。(秘密鍵, 公開鍵) のタプルを返す。"""
    private = subprocess.check_output(["wg", "genkey"]).decode().strip()
    public = subprocess.run(
        ["wg", "pubkey"], input=private.encode(), capture_output=True
    ).stdout.decode().strip()
    return private, public

def generate_preshared_key() -> str:
    return subprocess.check_output(["wg", "genpsk"]).decode().strip()

def build_client_config(
    client_private_key: str,
    client_vpn_ip: str,       # 例: "10.8.0.3"
    server_public_key: str,
    server_endpoint: str,     # 例: "home.example.com:51820"
    allowed_ips: str = "192.168.1.0/24",
    dns: str = "",
) -> str:
    dns_line = f"DNS = {dns}\n" if dns else ""
    return f"""[Interface]
PrivateKey = {client_private_key}
Address = {client_vpn_ip}/32
{dns_line}
[Peer]
PublicKey = {server_public_key}
Endpoint = {server_endpoint}
AllowedIPs = {allowed_ips}
PersistentKeepalive = 25
"""

def build_server_peer_block(
    client_public_key: str,
    client_vpn_ip: str,
    comment: str = "",
) -> str:
    comment_line = f"# {comment}\n" if comment else ""
    return f"""
{comment_line}[Peer]
PublicKey = {client_public_key}
AllowedIPs = {client_vpn_ip}/32
"""
```

秘密鍵をソース管理に含めないこと。このスクリプトを使用する場合は、鍵材料をモード600のファイルに書き込み、ログやプリントに出力しないこと。

## pfSense / OPNsense WireGuard

```
# pfSense: VPN → WireGuard → Add Tunnel
  Interface Keys: Generate（自動的に鍵ペアを作成）
  Listen Port: 51820
  Interface Address: 10.8.0.1/24

# ピアを追加（クライアントごとに1つ）:
  Public Key: <クライアントの公開鍵>
  Allowed IPs: 10.8.0.2/32

# WireGuardインターフェースを割り当て:
  Interfaces → Assignments → Add（wg0を選択）
  インターフェースを有効化、IPは不要（トンネル設定で設定済み）

# ファイアウォールルール:
  WAN → UDPポート51820のインバウンドを許可（クライアントがサーバーに到達できるように）
  WireGuardインターフェース → アクセスを許可したいLANネットワークへのトラフィックを許可
```

## DDNS（ダイナミックDNS）ホームサーバー用

ほとんどのホームインターネット接続は動的IPを持つ。DDNSを使用することで、IP変更後もVPNエンドポイントに到達できる。

```bash
# オプション1: Cloudflare DDNS — 認証情報をシークレットファイルに保存し、インラインに書かない
# envファイルを使用したdocker-composeのエントリ:
  ddns-updater:
    image: qmcgaw/ddns-updater
    env_file: ./ddns.env   # zone_idとtokenをここに保存し、composeには書かない
    restart: unless-stopped

# ddns.env（chmod 600、gitにコミットしない）:
#   SETTINGS_CLOUDFLARE_ZONE_ID=your_zone_id
#   SETTINGS_CLOUDFLARE_TOKEN=your_api_token

# オプション2: DuckDNS（無料、シンプル）
  duckdns.orgでサインアップ → トークンとサブドメインを取得（myhome.duckdns.org）
  トークンを /etc/ddns.env（モード600）に保存し、rootが所有する小さなスクリプトを使用:

  # /usr/local/bin/update-duckdns
  #!/bin/sh
  set -eu
  . /etc/ddns.env
  curl --fail --silent --show-error --max-time 10 \
    --get "https://www.duckdns.org/update" \
    --data-urlencode "domains=myhome" \
    --data-urlencode "token=${DUCKDNS_TOKEN}" \
    --data-urlencode "ip="

  # cronジョブ:
  */5 * * * * /usr/local/bin/update-duckdns >/dev/null 2>&1
```

## トラブルシューティング

```bash
# WireGuardのステータスと最後のハンドシェイクを確認
sudo wg show

# "latest handshake" が表示されないか非常に古い場合、トンネルが接続されていない。
# 確認事項:
# 1. ルーター/ファイアウォールでUDPポート51820が開いているか？
sudo ufw status  # またはpfSense/UniFiのファイアウォールルールを確認

# 2. クライアント設定のサーバー公開鍵が正しいか？
sudo wg show wg0 public-key   # クライアント設定の内容と比較する

# 3. サーバーでIPフォワーディングが有効になっているか？
cat /proc/sys/net/ipv4/ip_forward  # 1であるべき

# 4. クライアントのAllowedIPsが到達しようとしているIPをカバーしているか？
# AllowedIPs = 192.168.1.0/24 で 192.168.3.5 に到達しようとしても、ルーティングされない。

# WireGuardエラーのカーネルログを確認
dmesg | grep wireguard

# WireGuardを再起動
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

## アンチパターン

```
# 悪い例: 秘密鍵をバージョン管理に保存したり共有したりする
# 秘密鍵はパスワードと同等 — gitにコミットしないこと

# 悪い例: モバイルで影響を考慮せずにAllowedIPs = 0.0.0.0/0を使用する
# フルトンネルは全モバイルトラフィックをホームのアップロード経由にする — 通常は遅い

# 悪い例: モバイルクライアントにPersistentKeepaliveを設定しない
# NAT下のモバイルクライアントは設定がなければアイドルトンネルを切断する

# 悪い例: ファイアウォールでポート51820を開けるがサーバーのIPフォワーディングを忘れる
# トンネルは接続するがトラフィックがルーティングされない — デバッグが困難

# 悪い例: 複数のクライアントデバイスで鍵ペアを共有する
# 各デバイスは固有の鍵ペアを持たなければならない — 鍵の共有はセキュリティモデルを破壊する

# 悪い例: 広範な「FORWARD ACCEPT」iptablesルールを使用する
# 転送ルールはwg0インターフェースと方向のみに絞ること
```

## ベストプラクティス

- クライアントデバイスごとに固有の鍵ペアを生成する — 鍵を再利用しない
- モバイルにはスプリットトンネリング（`AllowedIPs = <ホームサブネット>`）を使用する
- 全モバイルクライアントに `PersistentKeepalive = 25` を設定する
- ISPが動的IPを割り当てる場合はDDNSを使用し、認証情報はenvファイルに保存してインラインに書かない
- スコープを絞ったiptables転送ルール（wg0のみのインバウンド）を使用し、FORWARDの全許可は避ける
- クライアント設定の `DNS =` にPi-holeのIPを追加してVPN経由の広告ブロックを有効にする
- サーバーの鍵ペアを定期的にローテーションして全クライアント設定を更新する

## 関連スキル

- homelab-network-setup
- homelab-vlan-segmentation
- homelab-pihole-dns
