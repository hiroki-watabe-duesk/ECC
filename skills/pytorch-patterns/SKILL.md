---
name: pytorch-patterns
description: PyTorchのディープラーニングパターンとベストプラクティス。堅牢で効率的かつ再現性のあるトレーニングパイプライン・モデルアーキテクチャ・データ読み込みの構築に対応。
origin: ECC
---

# PyTorch開発パターン

堅牢で効率的かつ再現性のあるディープラーニングアプリケーション構築のためのイディオマティックなPyTorchパターンとベストプラクティス。

## 有効化タイミング

- 新しいPyTorchモデルまたはトレーニングスクリプトを書く場合
- ディープラーニングコードのレビュー
- トレーニングループまたはデータパイプラインのデバッグ
- GPUメモリ使用量またはトレーニング速度の最適化
- 再現可能な実験のセットアップ

## 基本原則

### 1. デバイス非依存のコード

デバイスをハードコードせず、CPUとGPUの両方で動作するコードを常に書いてください。

```python
# 良い例: デバイス非依存
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = MyModel().to(device)
data = data.to(device)

# 悪い例: ハードコードされたデバイス
model = MyModel().cuda()  # GPUがなければクラッシュ
data = data.cuda()
```

### 2. 再現性を最優先

再現可能な結果のためにすべての乱数シードを設定します。

```python
# 良い例: 完全な再現性セットアップ
def set_seed(seed: int = 42) -> None:
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    np.random.seed(seed)
    random.seed(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

# 悪い例: シード制御なし
model = MyModel()  # 毎回異なる重み
```

### 3. 明示的なシェイプ管理

テンソルのシェイプを常に文書化し検証します。

```python
# 良い例: シェイプアノテーション付きフォワードパス
def forward(self, x: torch.Tensor) -> torch.Tensor:
    # x: (batch_size, channels, height, width)
    x = self.conv1(x)    # -> (batch_size, 32, H, W)
    x = self.pool(x)     # -> (batch_size, 32, H//2, W//2)
    x = x.view(x.size(0), -1)  # -> (batch_size, 32*H//2*W//2)
    return self.fc(x)    # -> (batch_size, num_classes)

# 悪い例: シェイプ追跡なし
def forward(self, x):
    x = self.conv1(x)
    x = self.pool(x)
    x = x.view(x.size(0), -1)  # このサイズは何？
    return self.fc(x)           # これは動作する？
```

## モデルアーキテクチャパターン

### クリーンなnn.Module構造

```python
# 良い例: 整理されたモジュール
class ImageClassifier(nn.Module):
    def __init__(self, num_classes: int, dropout: float = 0.5) -> None:
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
        )
        self.classifier = nn.Sequential(
            nn.Dropout(dropout),
            nn.Linear(64 * 16 * 16, num_classes),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.features(x)
        x = x.view(x.size(0), -1)
        return self.classifier(x)

# 悪い例: forwardにすべてを詰め込む
class ImageClassifier(nn.Module):
    def __init__(self):
        super().__init__()

    def forward(self, x):
        x = F.conv2d(x, weight=self.make_weight())  # 毎回重みを作成！
        return x
```

### 適切な重みの初期化

```python
# 良い例: 明示的な初期化
def _init_weights(self, module: nn.Module) -> None:
    if isinstance(module, nn.Linear):
        nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")
        if module.bias is not None:
            nn.init.zeros_(module.bias)
    elif isinstance(module, nn.Conv2d):
        nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")
    elif isinstance(module, nn.BatchNorm2d):
        nn.init.ones_(module.weight)
        nn.init.zeros_(module.bias)

model = MyModel()
model.apply(model._init_weights)
```

## トレーニングループパターン

### 標準的なトレーニングループ

```python
# 良い例: ベストプラクティスを備えた完全なトレーニングループ
def train_one_epoch(
    model: nn.Module,
    dataloader: DataLoader,
    optimizer: torch.optim.Optimizer,
    criterion: nn.Module,
    device: torch.device,
    scaler: torch.amp.GradScaler | None = None,
) -> float:
    model.train()  # 常にトレーニングモードを設定
    total_loss = 0.0

    for batch_idx, (data, target) in enumerate(dataloader):
        data, target = data.to(device), target.to(device)

        optimizer.zero_grad(set_to_none=True)  # zero_grad()より効率的

        # 混合精度トレーニング
        with torch.amp.autocast("cuda", enabled=scaler is not None):
            output = model(data)
            loss = criterion(output, target)

        if scaler is not None:
            scaler.scale(loss).backward()
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            scaler.step(optimizer)
            scaler.update()
        else:
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()

        total_loss += loss.item()

    return total_loss / len(dataloader)
```

### バリデーションループ

```python
# 良い例: 適切な評価
@torch.no_grad()  # torch.no_grad()ブロックでラップするより効率的
def evaluate(
    model: nn.Module,
    dataloader: DataLoader,
    criterion: nn.Module,
    device: torch.device,
) -> tuple[float, float]:
    model.eval()  # 常にevalモードを設定 — ドロップアウト無効化、BNのrunning statsを使用
    total_loss = 0.0
    correct = 0
    total = 0

    for data, target in dataloader:
        data, target = data.to(device), target.to(device)
        output = model(data)
        total_loss += criterion(output, target).item()
        correct += (output.argmax(1) == target).sum().item()
        total += target.size(0)

    return total_loss / len(dataloader), correct / total
```

## データパイプラインパターン

### カスタムデータセット

```python
# 良い例: 型ヒント付きのクリーンなデータセット
class ImageDataset(Dataset):
    def __init__(
        self,
        image_dir: str,
        labels: dict[str, int],
        transform: transforms.Compose | None = None,
    ) -> None:
        self.image_paths = list(Path(image_dir).glob("*.jpg"))
        self.labels = labels
        self.transform = transform

    def __len__(self) -> int:
        return len(self.image_paths)

    def __getitem__(self, idx: int) -> tuple[torch.Tensor, int]:
        img = Image.open(self.image_paths[idx]).convert("RGB")
        label = self.labels[self.image_paths[idx].stem]

        if self.transform:
            img = self.transform(img)

        return img, label
```

### 効率的なDataLoader設定

```python
# 良い例: 最適化されたDataLoader
dataloader = DataLoader(
    dataset,
    batch_size=32,
    shuffle=True,            # トレーニング用にシャッフル
    num_workers=4,           # 並列データ読み込み
    pin_memory=True,         # より高速なCPU->GPU転送
    persistent_workers=True, # エポック間でワーカーを維持
    drop_last=True,          # BatchNormのために一貫したバッチサイズ
)

# 悪い例: 遅いデフォルト
dataloader = DataLoader(dataset, batch_size=32)  # num_workers=0、pin_memoryなし
```

### 可変長データのカスタムCollate

```python
# 良い例: collate_fnでシーケンスをパディング
def collate_fn(batch: list[tuple[torch.Tensor, int]]) -> tuple[torch.Tensor, torch.Tensor]:
    sequences, labels = zip(*batch)
    # バッチの最大長にパディング
    padded = nn.utils.rnn.pad_sequence(sequences, batch_first=True, padding_value=0)
    return padded, torch.tensor(labels)

dataloader = DataLoader(dataset, batch_size=32, collate_fn=collate_fn)
```

## チェックポイントパターン

### チェックポイントの保存と読み込み

```python
# 良い例: すべてのトレーニング状態を含む完全なチェックポイント
def save_checkpoint(
    model: nn.Module,
    optimizer: torch.optim.Optimizer,
    epoch: int,
    loss: float,
    path: str,
) -> None:
    torch.save({
        "epoch": epoch,
        "model_state_dict": model.state_dict(),
        "optimizer_state_dict": optimizer.state_dict(),
        "loss": loss,
    }, path)

def load_checkpoint(
    path: str,
    model: nn.Module,
    optimizer: torch.optim.Optimizer | None = None,
) -> dict:
    checkpoint = torch.load(path, map_location="cpu", weights_only=True)
    model.load_state_dict(checkpoint["model_state_dict"])
    if optimizer:
        optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
    return checkpoint

# 悪い例: モデルの重みのみ保存（トレーニングを再開できない）
torch.save(model.state_dict(), "model.pt")
```

## パフォーマンス最適化

### 混合精度トレーニング

```python
# 良い例: GradScalerを使ったAMP
scaler = torch.amp.GradScaler("cuda")
for data, target in dataloader:
    with torch.amp.autocast("cuda"):
        output = model(data)
        loss = criterion(output, target)
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
    optimizer.zero_grad(set_to_none=True)
```

### 大型モデルのグラジェントチェックポイント

```python
# 良い例: メモリのために計算をトレード
from torch.utils.checkpoint import checkpoint

class LargeModel(nn.Module):
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # メモリを節約するためにバックワード時にアクティベーションを再計算
        x = checkpoint(self.block1, x, use_reentrant=False)
        x = checkpoint(self.block2, x, use_reentrant=False)
        return self.head(x)
```

### 速度向上のためのtorch.compile

```python
# 良い例: より高速な実行のためにモデルをコンパイル（PyTorch 2.0+）
model = MyModel().to(device)
model = torch.compile(model, mode="reduce-overhead")

# モード: "default"（安全）、"reduce-overhead"（高速）、"max-autotune"（最高速）
```

## クイックリファレンス: PyTorchイディオム

| イディオム | 説明 |
|-------|-------------|
| `model.train()` / `model.eval()` | トレーニング/評価前に必ずモードを設定する |
| `torch.no_grad()` | 推論時にグラジェントを無効化する |
| `optimizer.zero_grad(set_to_none=True)` | より効率的なグラジェントのクリア |
| `.to(device)` | デバイス非依存のテンソル/モデル配置 |
| `torch.amp.autocast` | 2倍速のための混合精度 |
| `pin_memory=True` | より高速なCPU→GPUデータ転送 |
| `torch.compile` | 速度向上のためのJITコンパイル（2.0+） |
| `weights_only=True` | セキュアなモデル読み込み |
| `torch.manual_seed` | 再現可能な実験 |
| `gradient_checkpointing` | メモリのために計算をトレード |

## 避けるべきアンチパターン

```python
# 悪い例: バリデーション中のmodel.eval()忘れ
model.train()
with torch.no_grad():
    output = model(val_data)  # ドロップアウトがまだ有効！BatchNormがバッチ統計を使用！

# 良い例: 常にevalモードを設定
model.eval()
with torch.no_grad():
    output = model(val_data)

# 悪い例: autogradを破壊するインプレース演算
x = F.relu(x, inplace=True)  # グラジェント計算を破壊する可能性
x += residual                  # インプレース加算はautogradグラフを破壊する

# 良い例: アウトオブプレース演算
x = F.relu(x)
x = x + residual

# 悪い例: トレーニングループ内で繰り返しデータをGPUに移動する
for data, target in dataloader:
    model = model.cuda()  # 毎イテレーションモデルを移動！

# 良い例: ループ前に一度だけモデルを移動する
model = model.to(device)
for data, target in dataloader:
    data, target = data.to(device), target.to(device)

# 悪い例: バックワード前に.item()を使用する
loss = criterion(output, target).item()  # グラフから切り離す！
loss.backward()  # エラー: .item()を通してバックプロップできない

# 良い例: ロギングのためだけに.item()を呼ぶ
loss = criterion(output, target)
loss.backward()
print(f"Loss: {loss.item():.4f}")  # バックワード後の.item()は問題なし

# 悪い例: torch.saveの不適切な使用
torch.save(model, "model.pt")  # モデル全体を保存（脆弱で移植性なし）

# 良い例: state_dictを保存する
torch.save(model.state_dict(), "model.pt")
```

__覚えておいてください__: PyTorchコードはデバイス非依存・再現可能・メモリを意識したものであるべきです。迷った場合は `torch.profiler` でプロファイリングし、`torch.cuda.memory_summary()` でGPUメモリを確認してください。
