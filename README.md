
name :`.NET 8 專案中，Linux 系統管理 架構設計C# 程式的設定檔`

Description=Nightly Linux Server AI Agent Task

"""
node : 用 Linux Systemd Service 與 Timer 來設定每日凌晨 02:00 自動執行 C# 程式的設定檔。

#核心任務規劃（四階段運作）

# 第一步：先將 C# 專案發布為原生單一執行檔

在 C# 專案目錄下執行發布指令

dotnet publish -c Release -r linux-x64 --self-contained -p:PublishSingleFile=true -o agentDiag>發布後，執行檔路徑為：`agentDiagnosticsClientApp`
1. 系統指令執行模組
CommandRunner.cs
透過 `System.Diagnostics.Process` 執行 `journalctl`、`df` 等 Linux 指令並抓取 Output 字串。

3. AI 推論串接模組
OllamaClient.cs
使用 `HttpClient` 將系統狀態傳遞給 `http://localhost:11434/api/chat`，並啟用 `format: "json"`。

4. 安全任務執行器
僅允許執行有註冊過的白名單指令（如 `journalctl --vacuum-time`），避免 AI 生成危險指令（如 `rm -rf`）。

4. 定時啟動服務
 my-agent-cron.service
透過 Linux 內建的 `crontab` 或 Systemd Timer 設定每天凌晨 02:00 自動啟動該 C# Executable。

# 第二步：建立 Systemd Service (服務本體)
這個設定檔負責定義「要執行什麼程式」。

1. 建立服務設定檔：
    /etc/systemd/system/nightly-agent.service

2. 貼入以下需要的內容：
 [Unit]
....

[Service]
...

# 標準輸出與錯誤輸出自動導向 journalctl

[Install]
...

### 第三步：建立 Systemd Timer (排程計時器)

這個設定檔負責定義「什麼時間觸發上面的 Service」。檔名必須與 Service 一致（副檔名改為 `.timer`）。

2. 貼入以下內容：

    ```
    [Unit]
    Description=Run Nightly AI Agent every day at 02:00 AM

    [Timer]
    # 每日凌晨 02:00 觸發
    OnCalendar=*-*-* 02:00:00
    # 隨機延遲 0~5 分鐘，避免與其他固定計時器搶資源
    # 隨機延遲 0~5 分鐘，避免與其他固定計時器搶資源
    RandomizedDelaySec=300

    # 若伺服器在 02:00 處於關機狀態，開機後補執行
    Persistent=true

    [Install]
    WantedBy=timers.target
    ```
### 第四步：載入並啟動 Timer

設定檔建立完成後，執行以下指令讓設定生效：

1. **重新載入 Systemd 設定：**

2. **啟用並啟動 Timer（不需要啟動 Service，Timer 會自動呼叫 Service **

### 第五步：維護與測試指令 (常用工具)

#### 1. 檢查排程狀態與下次執行時間：

```
systemctl list-timers --all | grep nightly-agent
```

_(畫面應會清晰顯示下次執行的倒數時間以及上一次執行的時間)_

#### 2. 手動觸發一次 Service（不安裝 Timer 直接測試 C# 程式能否順利被 Systemd 跑起來）：

```
sudo systemctl start nightly-agent.service
```

#### 3. 查看 C# Agent 執行的詳細 Output / Log：

```
sudo journalctl -u nightly-agent.service -n 50 --no-pager
```
-n 5 將 `journalctl` 調整為只抓最關鍵的 **5 條** 錯誤

排程設定完成    
