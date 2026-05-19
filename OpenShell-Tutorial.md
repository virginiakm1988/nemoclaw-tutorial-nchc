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

下面是完整流程。**預設用 OpenClaw 當 agent + NVIDIA Endpoints 當 inference provider** — 這是最沒門檻的組合，去 <https://build.nvidia.com> 註冊就有免費 API key，不需要 Anthropic / OpenAI 帳號。如果你想用其他組合（Ollama / Claude），看 §5 變化版。

### Step 1 — 建一個 sandbox，把 OpenClaw agent 裝進去

先在 host 上設好 NVIDIA API key（從 <https://build.nvidia.com> 取得）：

```bash
export NVIDIA_API_KEY=nvapi-...
```

建立 sandbox：

```bash
openshell sandbox create budget-agent --from openclaw -- openclaw
```

- `--from openclaw`：用預載 OpenClaw 的 base image
- `-- openclaw`：sandbox 啟動時直接跑 `openclaw` CLI（agent 本體）

建好之後 OpenShell 會把你連進 sandbox shell（prompt 變 `sandbox$`）。OpenClaw 第一次啟動會問你 inference provider — 選 **NVIDIA Endpoints** 並貼上 API key 即可。

> NemoClaw 就是 OpenClaw + OpenShell 的整合包。這份 tutorial 直接用底層的 OpenClaw + OpenShell，會更清楚兩層在做什麼。

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

兩種跟 OpenClaw 對話的方式，挑一個用：

**(a) 互動模式**（推薦新手）：

```bash
# sandbox$
openclaw
```

進到 REPL 後輸入 prompt：

```
請分析 ~/finance/transactions.csv：
1. 把消費分類（餐飲 / 居住 / 通勤 / 訂閱 / 購物 / 其他）
2. 算出每類別 4 月總花費 + 佔比
3. 把結果寫成 ~/finance/reports/2026-04.md
```

**(b) One-shot 模式**（適合腳本化）：

```bash
openclaw agent --agent main -m "請分析 ~/finance/transactions.csv：分類花費、算佔比、寫到 ~/finance/reports/2026-04.md" --session-id apr-report
```

OpenClaw 會自己用 Python（sandbox 內建 Python 3.14）讀 CSV、跑 pandas / 純 stdlib 算數、然後 `write` 出報表檔。完成後可以開來看：

```bash
cat ~/finance/reports/2026-04.md
```

✅ **第一個 milestone**：你已經有一個會分析消費的 agent。

### Step 4 — 進階對話：要省錢建議

繼續在 OpenClaw REPL 裡問（或用 `openclaw agent ... -m "..."` 接一條 prompt）：

```
看一下上面的報表，幫我找出 3 個可以省錢的點，並算如果照建議做，
我這個月可以多存多少。把建議追加到 reports/2026-04.md 最後面。
```

或者讓它做趨勢預測（如果你之後加更多月份的 CSV）：

```
請讀 ~/finance/transactions.csv 還有任何 transactions-*.csv 檔案，
畫出近三個月消費趨勢的 ASCII bar chart，存到 reports/trend.md
```

> 💡 OpenClaw 的「persona」可以在 `~/.openclaw/SOUL.md` 改 — 例如要它「講話像精算師」或「給建議要直接不要委婉」，下次對話就會帶這個風格。

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
    - ~/.openclaw/**  # agent 自己的 memory / skills
    - /usr/**         # 系統 binary 必要
  write:
    - ~/finance/reports/**
    - ~/.openclaw/memory/**

network:
  egress:
    # 只允許 NVIDIA Endpoints（給 OpenClaw 推理用）
    - host: integrate.api.nvidia.com
      method: ["POST"]
    # 其他全部 deny

inference:
  provider: nvidia
  model: meta/llama-3.1-70b-instruct
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

# (5) 連 NVIDIA Endpoints → 允許（透過 openclaw）
openclaw agent --agent main -m "say hi in 5 words" --session-id policy-test
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

## 5. Tutorial 變化版（換 agent / 換 inference provider）

預設組合是 **OpenClaw + NVIDIA Endpoints**。其他幾種對應到 Step 1：

| 想用… | API key 來源 | 怎麼改 Step 1 |
|------|--------------|---------------|
| **OpenClaw + NVIDIA Endpoints**（預設） | <https://build.nvidia.com>（免費） | `openshell sandbox create budget-agent --from openclaw -- openclaw` |
| **OpenClaw + 本地 Ollama** | 不需 API key（要有 GPU 或夠快的 CPU） | 先 `ollama pull llama3.1` 然後 `openshell sandbox create budget-agent --from ollama -- openclaw` |
| **OpenClaw + OpenAI** | OpenAI 帳號 | `export OPENAI_API_KEY=...` 再建 sandbox，OpenClaw onboarding 選 OpenAI |
| **Claude Code（付費版）** | Anthropic 帳號 | `export ANTHROPIC_API_KEY=sk-ant-...` 然後 `openshell sandbox create budget-agent -- claude` — 互動方式改用 `claude` 而不是 `openclaw` |

Step 2-4 流程一樣，只是「在 sandbox 內怎麼跟 agent 對話」的指令名字會變（OpenClaw → `openclaw`，Claude Code → `claude`）。

> **NCHC bootcamp 推薦**：用預設組合（NVIDIA Endpoints）。如果你的 VM 有 GPU 想完全離線，第二個（Ollama）也很適合。

---

## 6. Troubleshooting

| 問題 | 解法 |
|------|------|
| `openshell: command not found` | 重開 shell 或 `source ~/.bashrc`；確認 `~/.local/bin` 在 `PATH` |
| `unresolvable CDI devices nvidia.com/gpu=all` | 不需要 GPU 就加 `--no-gpu`；需要的話照 NemoClaw tutorial §9.2 裝 nvidia-container-toolkit |
| OpenClaw 在 sandbox 內說「no inference provider」 | 在 host 先 `export NVIDIA_API_KEY=nvapi-...` 再建 sandbox；或在 sandbox 內跑 `openclaw config inference` 重設 |
| 改用 Claude Code 時 `ANTHROPIC_API_KEY not set` | host 先 `export ANTHROPIC_API_KEY=sk-ant-...`；或用 `openshell provider create --type anthropic --from-existing` |
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
