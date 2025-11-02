# Chapter 08: トラブルシューティング

**📖 この章の目的**

エラーが発生したとき、**自分で原因を特定し解決できる**ようにします。

**🎯 この章で判断すべきこと**
```
□ 現在発生している問題は何か？
□ 問題の原因はどこにあるか？（Ollama / LiteLLM / Aider / 設定）
□ 自分で解決できるか、サポートが必要か？
```

**💡 最も多い問題 TOP 3**

```
1位: 🔌 LiteLLMに接続できない（50%）
    原因: LiteLLMが起動していない
    解決: sudo systemctl start litellm

2位: 🤖 モデルが見つからない（30%）
    原因: モデルがインストールされていない
    解決: ollama pull qwen3-coder:30b-a3b-q8_0

3位: 🐌 応答が遅い（15%）
    原因: GPU加速が無効
    解決: ROCm環境変数を設定
```

**🔍 問題の切り分けフローチャート**

```
エラー発生
    ↓
┌───────────────────────────────┐
│ 接続エラー？                  │
│ (Connection refused, timeout) │
└───────────────────────────────┘
    ↓ YES → 8.1節へ（接続エラー）
    ↓ NO
┌───────────────────────────────┐
│ 認証エラー？                  │
│ (Unauthorized, Invalid key)   │
└───────────────────────────────┘
    ↓ YES → 8.1.2節へ（認証エラー）
    ↓ NO
┌───────────────────────────────┐
│ モデルエラー？                │
│ (Model not found)             │
└───────────────────────────────┘
    ↓ YES → 8.1.3節へ（モデルエラー）
    ↓ NO
┌───────────────────────────────┐
│ パフォーマンス問題？          │
│ (遅い、Out of memory)         │
└───────────────────────────────┘
    ↓ YES → 8.2節へ（パフォーマンス）
    ↓ NO
┌───────────────────────────────┐
│ コード生成品質の問題？        │
└───────────────────────────────┘
    ↓ YES → 8.3節へ（品質問題）
```

---

## 8.1 よくある問題と解決方法

### 8.1.1 接続エラー

**🤔 あなたに該当する？**
```
□ "Connection refused"エラーが出る
□ Aiderが"Could not connect"と言う
□ curl http://localhost:8000/health が失敗する
```
→ 1つでもチェック: **この節を読むべき**

**問題1: "Could not connect to LiteLLM proxy"**

**💡 何が起きているか？**
AiderがLiteLLMプロキシに接続しようとしているが、LiteLLMが起動していない。

```bash
# 症状
$ aider
Error: Could not connect to http://localhost:8000

# 原因確認
$ curl http://localhost:8000/health
curl: (7) Failed to connect to localhost port 8000: Connection refused
```

**🔧 解決方法（まず試すこと）**

```bash
# ステップ1: LiteLLMが起動しているか確認
$ ps aux | grep litellm
# 何も表示されない → 起動していない

# ステップ2: LiteLLMを起動
$ cd ~/litellm
$ source venv/bin/activate
$ litellm --config config.yaml --port 8000 --host 0.0.0.0

# または systemdサービスとして
$ sudo systemctl start litellm
$ sudo systemctl status litellm

# ステップ3: 動作確認
$ curl http://localhost:8000/health
# {"status": "healthy"} と表示されればOK
```

**✅ 正常に起動している状態**
```bash
$ sudo systemctl status litellm
● litellm.service - LiteLLM Proxy
   Active: active (running) since ...

$ curl http://localhost:8000/health
{"status": "healthy"}
```

**⚠️ それでも解決しない場合**
```bash
# ポートが使用されているか確認
$ sudo lsof -i :8000
# 別のプロセスが使っている場合は、そのプロセスを停止

# ログを確認
$ sudo journalctl -u litellm -n 50
# エラーメッセージを確認
```

**問題2: "Ollama service not responding"**

**💡 何が起きているか？**
LiteLLMが正常でも、その後ろのOllamaが起動していない。

```bash
# 症状
$ curl http://localhost:11434/api/tags
curl: (7) Failed to connect

# 解決方法
$ sudo systemctl status ollama
# Active: inactive (dead) → 起動していない

$ sudo systemctl start ollama
$ sudo systemctl status ollama
# Active: active (running) → OK

# GPU認識確認
$ rocm-smi
# GPU 0が表示され、使用率が表示されればOK
```

**✅ 正常に起動している状態**
```bash
$ ollama ps
NAME                         ID        SIZE    UNTIL
qwen3-coder:30b-a3b-q8_0     abc123    34 GB   5 minutes from now
```

### 8.1.2 認証エラー

**🤔 あなたに該当する？**
```
□ "Unauthorized"エラーが出る
□ "Invalid API key"と言われる
□ 環境変数 OPENAI_API_KEY が空
```
→ 1つでもチェック: **この節を読むべき**

**問題: "Unauthorized" または "Invalid API key"**

**💡 何が起きているか？**
API keyが設定されていないか、config.yamlのmaster_keyと一致していない。

```bash
# 原因確認
$ echo $OPENAI_API_KEY
# 空または間違ったキー

# 解決方法
$ export OPENAI_API_KEY="sk-local-dev-1234"

# ~/.bashrcに追加（永続化）
$ echo 'export OPENAI_API_KEY="sk-local-dev-1234"' >> ~/.bashrc
$ source ~/.bashrc

# config.yamlのmaster_keyと一致しているか確認
$ grep master_key ~/litellm/config.yaml
```

**✅ 正しい設定**
```bash
# ~/.bashrc または ~/.zshrc
export OPENAI_API_KEY="sk-local-dev-1234"

# config.yaml
general_settings:
  master_key: "sk-local-dev-1234"  # ← 一致している
```

### 8.1.3 モデルエラー

**🤔 あなたに該当する？**
```
□ "Model not found"エラーが出る
□ ollama listでモデルが表示されない
□ モデルをpullしていない
```
→ 1つでもチェック: **この節を読むべき**

**問題: "Model not found"**

**💡 何が起きているか？**
LiteLLMが要求するモデルがOllamaにインストールされていない。

```bash
# 症状
Error: Model 'qwen3-coder:30b-a3b-q8_0' not found

# 原因確認
$ ollama list
# qwen3-coder:30b-a3b-q8_0が表示されない

# 解決方法
$ ollama pull qwen3-coder:30b-a3b-q8_0

# LiteLLM config.yamlの確認
$ cat ~/litellm/config.yaml | grep qwen3-coder
```

**問題: "Context length exceeded"**

**💡 何が起きているか？**
処理しようとしているコードが、モデルのコンテキストウィンドウサイズを超えている。

**🤔 あなたに該当する？**
```
□ 大規模ファイル（1000行以上）を処理しようとしている
□ 多数のファイルを同時に /add している
□ 会話履歴が長い
```

```bash
# 症状
Error: This model's maximum context length is 262144 tokens

# 💡 まず試すこと（優先順）

# 方法1: 不要なファイルを除外
> /drop unnecessary_file.py
> /drop old_code.py

# 方法2: 会話履歴をクリア
> /clear

# 方法3: map-tokensを削減
$ nano ~/.aider.conf.yml
map-tokens: 4096  # 8192から削減
max-chat-history-tokens: 8192  # 16384から削減

# 方法4: ファイルを分割して処理
> /add module_part1.py
# 処理完了後
> /drop module_part1.py
> /add module_part2.py
```

**✅ 予防策**
```bash
# 大規模プロジェクトでは最初から設定を調整
# ~/.aider.conf.yml
map-tokens: 4096      # デフォルト: 8192
max-chat-history-tokens: 8192  # デフォルト: 16384
```

---

## 8.2 パフォーマンス問題

### 8.2.1 応答が遅い

**🤔 あなたに該当する？**
```
□ 応答に10秒以上かかる
□ GPU使用率が低い（50%未満）
□ rocm-smiでGPUが0%と表示される
```
→ 1つでもチェック: **この節を読むべき**

**問題: 応答に30秒以上かかる**

**💡 何が起きているか？**
GPU加速が無効になっており、CPU処理になっている。

**診断**

```bash
# GPU使用率を確認
$ rocm-smi

# 出力例（問題あり）:
# GPU  Temp   AvgPwr  SCLK     MCLK     Fan   Perf  PwrCap  VRAM%  GPU%
# 0    45.0c  15.0W   800Mhz   1000Mhz  0%    auto  120.0W  15%    0%
#                                                                    ↑ 0%は問題
```

**解決方法**

```bash
# ROCm環境変数を確認
$ sudo systemctl show ollama | grep Environment

# 正しく設定されていない場合
$ sudo nano /etc/systemd/system/ollama.service.d/override.conf

[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="PYTORCH_ROCM_ARCH=gfx1100"
Environment="GPU_MAX_ALLOC_PERCENT=95"

$ sudo systemctl daemon-reload
$ sudo systemctl restart ollama

# GPU使用率を再確認
$ rocm-smi
# GPU% が 80-100% になっていればOK
```

**✅ 正常なパフォーマンス（MS-S1 Max）**
```bash
$ rocm-smi
GPU  Temp  AvgPwr  SCLK    MCLK    Fan  Perf  VRAM%  GPU%
0    65c   80W     2600Mhz 2000Mhz 60%  auto  35%    85%
                                                      ↑ 80-100%が正常
```

**⚠️ 期待される応答時間**
```
簡単なタスク（コメント追加）: 2-4秒
通常タスク（関数修正）: 3-8秒
複雑なタスク（リファクタリング）: 8-15秒

→ これより遅い場合は設定を見直す
```

### 8.2.2 メモリ不足

**🤔 あなたに該当する？**
```
□ "Out of memory"エラーが出る
□ メモリ使用率が90%超
□ 複数の大型モデルを同時に使っている
```
→ 1つでもチェック: **この節を読むべき**

**問題: "Out of memory" エラー**

**💡 何が起きているか？**
モデルやコンテキストがメモリを使い果たしている。

```bash
# メモリ使用状況を確認
$ free -h
              total        used        free
Mem:          128Gi        95Gi        33Gi

# Ollamaのメモリ使用量を確認
$ ollama ps
NAME                         ID              SIZE      UNTIL
qwen3-coder:30b-a3b-q8_0     abc123          34 GB     5 minutes
qwen3-coder:14b              def456          20 GB     5 minutes
# 合計54GB使用中（MS-S1 Maxは96GB VRAM + 128GB総メモリで余裕）
```

**解決方法**

```bash
# 方法1: 不要なモデルをアンロード（MS-S1 Maxでは通常不要）
$ ollama stop qwen3-coder:14b  # 軽量モデルをアンロード

# 方法2: KEEP_ALIVEを短縮
$ sudo nano /etc/systemd/system/ollama.service.d/override.conf
Environment="OLLAMA_KEEP_ALIVE=2m"  # 5mから短縮

$ sudo systemctl daemon-reload
$ sudo systemctl restart ollama

# 方法3: より小さいモデルを使用（MS-S1 Maxでは通常不要）
> /model gpt-3.5-turbo  # qwen3-coder:14b (20GB)
> /model claude-3-haiku-20240307  # qwen3-coder:7b (10GB、最軽量)
```

### 8.2.3 ディスクI/O問題

**問題: モデルロードが遅い**

```bash
# ディスク速度を確認
$ sudo hdparm -Tt /dev/nvme0n1

# SSDでない場合やUSBドライブの場合は移動
$ sudo mv ~/.ollama /mnt/nvme/ollama
$ sudo ln -s /mnt/nvme/ollama ~/.ollama
$ sudo systemctl restart ollama
```

**✅ MS-S1 Maxでの正常なメモリ使用**
```bash
$ free -h
              total        used        free
Mem:          128Gi        54Gi        74Gi
# ← 50GB程度の使用は正常（qwen3-coder:30b-a3b-q8_0は34GB）
```

**⚠️ 注意**: MS-S1 Maxは128GB RAMと96GB VRAM割り当てがあるため、通常はメモリ不足にならない。64GB以下のマシンでは軽量モデル推奨。

---

## 8.3 コード生成品質の問題

### 8.3.1 生成コードが期待と異なる

**🤔 あなたに該当する？**
```
□ 指示と異なるコードが生成される
□ プロンプトが曖昧すぎる
□ 毎回異なる結果が返ってくる
```
→ 1つでもチェック: **この節を読むべき**

**問題: プロンプトに沿わないコードが生成される**

**💡 何が起きているか？**
プロンプトが曖昧で、LLMが意図を理解できていない。

**悪い例**

```
> make it better
```

**良い例**

```
> Improve this function by:
> 1. Adding type hints to all parameters and return value
> 2. Adding comprehensive docstring with Args, Returns, and Examples
> 3. Adding input validation for edge cases (None, empty list, negative numbers)
> 4. Using more descriptive variable names
> 5. Optimizing the algorithm for better time complexity
```

### 8.3.2 一貫性のない出力

**問題: 同じ質問で毎回異なる回答**

**解決方法: Temperatureを調整**

```yaml
# config.yaml
model_list:
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen3-coder:30b-a3b-q8_0
      num_ctx: 262144  # 256K context
      temperature: 0.3  # 0.7から削減（より決定論的）
```

または

```
> /model claude-3-5-sonnet-20241022
# Aiderでtemperatureを指定
aider --model claude-3-5-sonnet-20241022 --temperature 0.2
```

### 8.3.3 コードが動作しない

**問題: 生成されたコードにバグがある**

**デバッグ手順**

```bash
# Step 1: コードを実行
> /run python3 generated_code.py

# Step 2: エラーメッセージをフィードバック
> The code fails with this error:
> Traceback (most recent call last):
>   File "generated_code.py", line 10, in <module>
>     result = process_data(None)
>   File "generated_code.py", line 5, in process_data
>     return data.split(',')
> AttributeError: 'NoneType' object has no attribute 'split'
>
> Fix this bug by adding proper input validation

# Step 3: レビューを依頼
> Review the fixed code and suggest additional improvements
```

## 8.4 Git統合の問題

### 8.4.1 コミットエラー

**問題: "Git user not configured"**

```bash
$ git config user.name
# 何も表示されない

# 解決方法
$ git config --global user.name "Your Name"
$ git config --global user.email "[email protected]"
```

**問題: "Uncommitted changes"**

```bash
# Aiderが変更をコミットしない

# 原因確認
$ cat ~/.aider.conf.yml | grep commit
auto-commits: false  # 無効になっている

# 解決方法
$ nano ~/.aider.conf.yml
auto-commits: true
dirty-commits: true

# または手動でコミット
> /commit
Commit message: Add error handling
```

### 8.4.2 大きなファイルの扱い

**問題: "File too large"**

```bash
# 大きなファイル（>10MB）を追加しようとするとエラー

# 解決方法1: .gitignoreに追加
$ echo "large_data.csv" >> .gitignore
$ git add .gitignore
$ git commit -m "Ignore large data files"

# 解決方法2: Git LFSを使用
$ git lfs install
$ git lfs track "*.csv"
$ git add .gitattributes
$ git commit -m "Track CSV files with LFS"
```

## 8.5 診断ツール

### 8.5.1 包括的なヘルスチェック

```bash
#!/bin/bash
# health_check.sh

echo "=== System Health Check ==="

# 1. Ollama
echo -e "\n[1] Ollama Status:"
systemctl is-active ollama && echo "✓ Running" || echo "✗ Not running"
curl -s http://localhost:11434/api/tags > /dev/null && echo "✓ API responding" || echo "✗ API not responding"

# 2. LiteLLM
echo -e "\n[2] LiteLLM Status:"
curl -s http://localhost:8000/health > /dev/null && echo "✓ Running" || echo "✗ Not running"

# 3. GPU
echo -e "\n[3] GPU Status:"
rocm-smi --showuse | grep "GPU use" || echo "✗ ROCm not available"

# 4. Memory
echo -e "\n[4] Memory:"
free -h | grep Mem

# 5. Models
echo -e "\n[5] Loaded Models:"
ollama ps

# 6. Disk Space
echo -e "\n[6] Disk Space:"
df -h ~ | tail -1

echo -e "\n=== Health Check Complete ==="
```

実行：

```bash
$ chmod +x health_check.sh
$ ./health_check.sh
```

### 8.5.2 詳細なログ収集

```bash
#!/bin/bash
# collect_logs.sh

LOG_DIR="./debug_logs_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$LOG_DIR"

echo "Collecting logs to $LOG_DIR..."

# Ollama logs
sudo journalctl -u ollama -n 100 > "$LOG_DIR/ollama.log"

# LiteLLM logs
if [ -f ~/litellm/litellm.log ]; then
    cp ~/litellm/litellm.log "$LOG_DIR/"
fi

# System info
uname -a > "$LOG_DIR/system_info.txt"
free -h >> "$LOG_DIR/system_info.txt"
df -h >> "$LOG_DIR/system_info.txt"

# ROCm info
rocm-smi > "$LOG_DIR/rocm_info.txt" 2>&1

# Ollama models
ollama list > "$LOG_DIR/ollama_models.txt"

# LiteLLM config
cp ~/litellm/config.yaml "$LOG_DIR/" 2>/dev/null

# Aider config
cp ~/.aider.conf.yml "$LOG_DIR/" 2>/dev/null

echo "Logs collected in $LOG_DIR"
echo "You can share this directory for troubleshooting"
```

## 8.6 パフォーマンスベンチマーク

### 8.6.1 エンドツーエンドベンチマーク

```python
# e2e_benchmark.py
import time
import requests

def benchmark_e2e():
    """エンドツーエンドパフォーマンステスト"""
    url = "http://localhost:8000/v1/chat/completions"
    headers = {
        "Authorization": "Bearer sk-local-dev-1234",
        "Content-Type": "application/json"
    }

    test_cases = [
        "Write a function to reverse a string",
        "Explain what a Python decorator is",
        "Fix this bug: def f(l): return l[10]"
    ]

    results = []

    for i, prompt in enumerate(test_cases, 1):
        print(f"Test {i}/3: {prompt[:50]}...")

        start = time.time()
        response = requests.post(url, headers=headers, json={
            "model": "claude-3-5-sonnet-20241022",
            "messages": [{"role": "user", "content": prompt}]
        })
        end = time.time()

        if response.status_code == 200:
            data = response.json()
            results.append({
                "prompt": prompt,
                "time": end - start,
                "tokens": data['usage']['total_tokens']
            })
            print(f"  ✓ {end - start:.2f}s")
        else:
            print(f"  ✗ Error: {response.status_code}")

    # Summary
    print(f"\n=== Summary ===")
    avg_time = sum(r['time'] for r in results) / len(results)
    print(f"Average response time: {avg_time:.2f}s")
    print(f"Total tokens: {sum(r['tokens'] for r in results)}")

if __name__ == "__main__":
    benchmark_e2e()
```

### 8.6.2 期待される結果（MS-S1 Max）

```
Test 1/3: Write a function to reverse a string...
  ✓ 3.42s
Test 2/3: Explain what a Python decorator is...
  ✓ 5.78s
Test 3/3: Fix this bug: def f(l): return l[10]...
  ✓ 4.21s

=== Summary ===
Average response time: 4.47s
Total tokens: 387
```

**正常範囲**: 3-8秒
**要調査**: 10秒以上

## 8.7 サポートリソース

### 8.7.1 コミュニティ

- **Ollama GitHub**: https://github.com/ollama/ollama
- **LiteLLM GitHub**: https://github.com/BerriAI/litellm
- **Aider GitHub**: https://github.com/paul-gauthier/aider

### 8.7.2 デバッグ情報の提供

問題を報告する際に含めるべき情報：

```bash
# システム情報
uname -a
lsb_release -a

# バージョン情報
ollama --version
litellm --version
aider --version

# GPU情報
rocm-smi
rocminfo | grep "Name:"

# ログ
sudo journalctl -u ollama -n 50
tail -50 ~/litellm/litellm.log

# 設定
cat ~/litellm/config.yaml
cat ~/.aider.conf.yml
```

---

## 8.8 まとめ

本章では、よくある問題のトラブルシューティング方法を学びました。

**重要なチェックポイント**
✅ サービスが起動しているか
✅ 環境変数が正しく設定されているか
✅ GPUが認識されているか
✅ メモリに余裕があるか
✅ プロンプトが明確か

**デバッグの基本手順**
```
1. エラーメッセージを確認
   → どのコンポーネントのエラーか特定

2. ログを確認
   → journalctl -u ollama / journalctl -u litellm

3. 設定を確認
   → config.yaml, .aider.conf.yml, 環境変数

4. サービスを再起動
   → systemctl restart ollama litellm

5. ベンチマークで検証
   → rocm-smi, free -h, ollama ps
```

**❓ よくある質問（FAQ）**

**Q: すべてのチェックをしたが問題が解決しない**
A: `health_check.sh`と`collect_logs.sh`を実行し、ログを収集してGitHub Issueで報告してください。

**Q: エラーメッセージが英語でわからない**
A: エラーメッセージをAider/LLMに貼り付けて「このエラーの意味と解決方法を教えて」と聞きましょう。

**Q: どのログを確認すればいい？**
A:
- Ollama: `sudo journalctl -u ollama -n 50`
- LiteLLM: `tail -50 ~/litellm/litellm.log`
- Aider: ターミナルの出力

**Q: 本番環境で問題が起きたら？**
A:
1. まずサービスを再起動
2. ログを収集
3. バックアップから復旧（Chapter 09参照）

**Q: パフォーマンスが突然悪化した**
A:
- GPUドライバ（ROCm）が更新された可能性
- 環境変数を再確認
- Ollamaを再インストール

**💡 トラブルシューティングのコツ**
1. **焦らず切り分ける**: フローチャート（本章冒頭）を使う
2. **ログを読む習慣**: エラーは必ずログに記録されている
3. **段階的に戻す**: 最後に変更した設定を元に戻す
4. **記録を取る**: 問題と解決方法をメモ（次回に活かす）
5. **コミュニティを活用**: GitHub Issueで質問する

**📊 問題解決までの平均時間**
```
接続エラー: 2-5分（サービス起動）
認証エラー: 1-3分（環境変数設定）
モデルエラー: 5-15分（モデルダウンロード）
パフォーマンス問題: 10-30分（設定調整）
品質問題: 3-10分（プロンプト改善）
```

**次のステップ**
最終章では、高度な設定とベストプラクティスを学びます。

**確認チェックリスト**
- [ ] health_check.shが実行できる
- [ ] ログを収集できる
- [ ] パフォーマンスを測定できる
- [ ] 基本的な問題を解決できる
- [ ] 問題切り分けフローチャートを理解している

すべてチェックできたら、Chapter 09へ進みましょう！
