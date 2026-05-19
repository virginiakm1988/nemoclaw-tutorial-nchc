# OpenShell 快速上手：5 步驟做一個記帳助理 Agent

> 目標：跟著這份文件走完，你會用 OpenShell 建好一個自己的 **記帳助理 agent** — 它讀本地的 `transactions.csv`、算分類花費、給省錢建議，全程不需要連外網。
>
> 官方文件：<https://docs.nvidia.com/openshell/> · GitHub：<https://github.com/NVIDIA/OpenShell>

---

## 1. OpenShell 是什麼？

OpenShell 是 NVIDIA 開源的 **AI agent sandbox runtime**。簡單講就是：

> 給 AI agent 一個「沙盒」，限制它能讀哪些檔案、能連哪些網域、能跑什麼系統呼叫 — 但不犧牲它的工作能力。

它跟我們前面那份 NemoClaw 教學的關係：

```
┌─────────────────────────┐
│  NemoClaw CLI           │  ← 高階部署工具（OpenClaw + OpenShell 整包）
├─────────────────────────┤
│  OpenClaw / Claude Code │  ← agent 本體（會推理、會用工具）
├─────────────────────────┤
│  OpenShell  ◀────────── │  ← 本文重點：sandbox runtime
└─────────────────────────┘
```

NemoClaw 把整套打包；如果你只想要 sandbox 本身、自己挑要跑哪種 agent，就直接用 OpenShell。

### 它解決什麼問題？

讓 agent 在隔離環境裡操作，靠四層 policy 防呆：

| 層級 | 預設行為 |
|------|----------|
| Filesystem | 只能讀寫被允許的路徑（Landlock） |
| Network | 預設拒絕所有對外連線，policy 白名單才放行 |
| Process | 阻擋 privilege escalation 與危險 syscall（Seccomp） |
| Inference | 把 model API 呼叫導去你指定的 endpoint |

---

## 2. 環境需求 & 安裝

| 項目 | 需求 |
|------|------|
| OS | Linux（推薦 Ubuntu 22.04+）或 macOS |
| Docker | Engine 24+ 在跑 |
| Disk | ≥ 5 GB |

**安裝（一行）**：

```bash
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
```

驗證：

```bash
openshell --version
openshell --help
```

> 如果你之前已經跑過 NemoClaw onboarding，OpenShell 多半已經被裝過了。`openshell --version` 有出版本就 OK。

---

## 3. 動手做：記帳助理 Agent

下面是完整流程。預設 inference provider 是 Claude（需要 `ANTHROPIC_API_KEY`，或換成 NVIDIA Endpoints / OpenAI / Ollama 都行）。

### Step 1 — 建一個 sandbox，把 Claude Code 裝進去

```bash
export ANTHROPIC_API_KEY=sk-ant-...     # 或設定別的 provider
openshell sandbox create budget-agent -- claude
```

`-- claude` 表示這個 sandbox 預載 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 當 agent 本體。建好之後 OpenShell 會直接把你連進 sandbox shell（看到 prompt 變 `sandbox$`）。

> 想換 agent：`--from openclaw -- openclaw`、`--from ollama -- ollama run llama3` 都行。Tutorial 後面用 `claude`。

### Step 2 — 在 sandbox 內準備測試資料

進去之後，建立工作目錄與一份範例 transactions：

```bash
# 在 sandbox 內 (sandbox$ prompt)
mkdir -p ~/finance && cd ~/finance

cat > transactions.csv <<'EOF'
date,amount,merchant,note
2026-04-02,-3200,房東,4月房租
2026-04-03,-185,全聯,日用品
2026-04-05,-95,星巴克,咖啡
2026-04-07,-1250,台電,電費
2026-04-09,-420,Uber Eats,午餐
2026-04-10,55000,公司,薪資
2026-04-11,-680,家樂福,週末採購
2026-04-13,-95,星巴克,咖啡
2026-04-15,-260,Spotify,訂閱
2026-04-16,-380,Uber,通勤
2026-04-18,-1480,誠品,書 + 文具
2026-04-19,-95,星巴克,咖啡
2026-04-21,-560,7-11,雜物
2026-04-23,-2200,IKEA,家具
2026-04-25,-95,星巴克,咖啡
2026-04-27,-890,五分埔,衣服
2026-04-28,-340,計程車,加班
2026-04-30,-120,全家,早餐
EOF
```

### Step 3 — 讓 agent 做第一份報表

最簡單的玩法：直接跟 `claude` 對話。

```bash
# sandbox$
claude
```

進到 Claude Code 互動介面後輸入：

```
請分析 ~/finance/transactions.csv：
1. 把消費分類（餐飲 / 居住 / 通勤 / 訂閱 / 購物 / 其他）
2. 算出每類別 4 月總花費 + 佔比
3. 把結果寫成 ~/finance/reports/2026-04.md
```

Claude Code 會自己用 Python（sandbox 內建 Python 3.14）讀 CSV、跑 pandas / 純 stdlib 算數、然後 `write` 出報表檔。完成後可以開來看：

```bash
cat ~/finance/reports/2026-04.md
```

✅ **第一個 milestone**：你已經有一個會分析消費的 agent。

### Step 4 — 進階對話：要省錢建議

繼續在 Claude Code 裡問：

```
看一下上面的報表，幫我找出 3 個可以省錢的點，並算如果照建議做，
我這個月可以多存多少。把建議追加到 reports/2026-04.md 最後面。
```

或者讓它做趨勢預測（如果你之後加更多月份的 CSV）：

```
請讀 ~/finance/transactions.csv 還有任何 transactions-*.csv 檔案，
畫出近三個月消費趨勢的 ASCII bar chart，存到 reports/trend.md
```

✅ **第二個 milestone**：agent 從「會算」升級到「會建議」。

### Step 5 — 用 OpenShell policy 把 agent 鎖在沙盒內

到目前為止 agent 已經很 useful，但你可能會擔心：「它會不會偷讀我其他資料夾？會不會偷上傳到雲端？」

OpenShell 的核心價值就是這裡。離開 Claude Code（`/quit`）、退出 sandbox shell（`exit`），回到 host：

```bash
# host
cat > ~/budget-policy.yaml <<'EOF'
filesystem:
  read:
    - ~/finance/**
    - /usr/**         # 系統 binary 必要
  write:
    - ~/finance/reports/**

network:
  egress:
    # 只允許 Anthropic API（給 Claude 推理用）
    - host: api.anthropic.com
      method: ["POST"]
    # 其他全部 deny

inference:
  provider: anthropic
  model: claude-haiku-4-5
EOF

openshell policy set budget-agent --policy ~/budget-policy.yaml --wait
```

驗證 policy 有生效。重新連進 sandbox：

```bash
openshell sandbox connect budget-agent
```

試 4 件事：

```bash
# (1) 讀 ~/finance/  → 允許
cat ~/finance/transactions.csv | head -3

# (2) 讀 /etc/passwd → 拒絕
cat /etc/passwd
#  → Permission denied (Landlock)

# (3) 寫到 /tmp → 拒絕
echo hi > /tmp/test
#  → Permission denied

# (4) 連任意外網 → 拒絕
curl -sS https://example.com
#  → 403 from proxy (policy_denied)

# (5) 連 Anthropic API → 允許（透過 claude 工具）
claude -p "say hi in 5 words"
```

✅ **第三個 milestone**：你的記帳助理只能做它該做的事，其他全部 deny。

---

## 4. 常用指令速查

```bash
# 生命週期
openshell sandbox create <name> -- <agent>   # 建立 + 啟動
openshell sandbox list                       # 列出所有 sandbox
openshell sandbox connect <name>             # SSH 進去
openshell sandbox stop <name>                # 停止
openshell sandbox destroy <name>             # 刪掉

# 觀察
openshell logs <name> --tail                 # 串流 log
openshell term                               # TUI dashboard（多 sandbox 監看）

# Policy
openshell policy set <name> --policy file.yaml --wait
openshell policy get <name>                  # 看當前 policy

# Inference
openshell inference set --provider anthropic --model claude-haiku-4-5
openshell provider create --type anthropic --from-existing  # 從環境變數抓 key

# 進階：跑別的 agent
openshell sandbox create my-coder --from openclaw -- openclaw
openshell sandbox create local-llm --from ollama -- ollama run llama3
```

---

## 5. Tutorial 變化版（不想用 Claude？）

| 想用… | 怎麼改 Step 1 |
|------|---------------|
| **OpenClaw** | `openshell sandbox create budget-agent --from openclaw -- openclaw agent --agent main` |
| **本地 Ollama** | `openshell sandbox create budget-agent --from ollama -- ollama run llama3.1` |
| **NVIDIA Endpoints** | 先 `openshell inference set --provider nvidia --model meta/llama-3.1-70b-instruct`，再用 `claude` 或 `openclaw` 都會走 NVIDIA |

Step 2-4 完全相同，只是「在 sandbox 內怎麼跟 agent 對話」會稍微不一樣（OpenClaw 用 `openclaw -m "..."`，Ollama 直接 prompt）。

---

## 6. Troubleshooting

| 問題 | 解法 |
|------|------|
| `openshell: command not found` | 重開 shell 或 `source ~/.bashrc`；確認 `~/.local/bin` 在 `PATH` |
| `unresolvable CDI devices nvidia.com/gpu=all` | 不需要 GPU 就加 `--no-gpu`；需要的話照 NemoClaw tutorial §9.2 裝 nvidia-container-toolkit |
| Claude 在 sandbox 內說「`ANTHROPIC_API_KEY` not set」 | 在 host 先 export，OpenShell provider 會自動帶進 sandbox；或用 `openshell provider create --type anthropic --from-existing` |
| 進 sandbox 後執行 curl 卡住 | 預設 network default-deny，要 `openshell policy set` 加 host 白名單 |
| 寫檔失敗 `Permission denied` | 檢查 policy 的 `filesystem.write` 有沒有列到目標路徑 |

---

## 7. 下一步

做完這份 tutorial 後可以試：

1. **多 agent**：再開一個 sandbox 叫 `inbox-agent`，給它 emails CSV，做收件匣分類。比較兩個 agent 的 policy 差異
2. **加 channels**：把 Telegram bot 接到 budget-agent（每月 1 號自動發報表）— 看 OpenClaw 的 channel docs
3. **GPU 推理**：用 `--from ollama-gpu` 跑大一點的 model 做更細的 categorization
4. **打包成 NemoClaw blueprint**：把這整套 sandbox + policy 用 NemoClaw `nemoclaw deploy <blueprint>` 一鍵還原（給同事 demo 用）

更多範例：<https://github.com/NVIDIA/OpenShell/tree/main/examples>

---

完成後，你應該已經能：

- 用 OpenShell 建 sandbox、塞 agent 進去、互動
- 寫一份 YAML policy 同時控制 filesystem + network + inference
- 驗證 sandbox 真的把不該允許的操作擋掉
- 把 NemoClaw / OpenClaw / OpenShell 三層的角色講清楚

Happy sandboxing! 🛡️
