# GitHub 免費版防繞過 CI 排版檢查完整建置教程（強制 4 空格 / 禁用 Tab）

本教程完整記錄如何在 **GitHub 個人免費版（GitHub Free）** 環境下，不架設任何外部伺服器，僅透過 GitHub 原生功能建立一套**「只檢查 PR 異動檔案」**、**「強制 4 空格縮排（禁用 Tab）」**且**「無法透過修改或刪除 CI 腳本繞過」**的自動化防線。

---

## 一、 前置條件與平台限制

在開始建置前，請確認你的儲存庫權限類型：

| 功能項目 | 公開專案（Public Repo） | 私人專案（Private Repo） |
| :--- | :--- | :--- |
| **GitHub Actions 自動檢查**（`pull_request_target` + `git diff`） | ✅ **完全免費**（無分鐘數限制） | ✅ **完全免費**（每月 2,000 分鐘額度，純文字檢查約跑 5 秒） |
| **PR 頁面顯示綠燈 / 紅叉警告** | ✅ **完全支援** | ✅ **完全支援** |
| **`CODEOWNERS` 鎖死 `.github/` 修改權** | ✅ **完全免費支援** | ❌ 需升級 GitHub Pro（$4/月） |
| **Rulesets 分支保護（失敗時強制反灰鎖死 Merge 按鈕）** | ✅ **完全免費支援** | ❌ 需升級 GitHub Pro（$4/月） |

> **結論**：若要免費啟用「檢查失敗強制拒絕合併」，儲存庫必須設為 **Public**；若為 Private，系統仍會執行檢查並顯示紅叉警告，但不會強制鎖死合併按鈕。

---

## 二、 核心原理與五道防繞過機制

一般教學使用 `on: pull_request` 存在一個重大漏洞：**GitHub 預設會執行「PR 分支內」的 Workflow 設定檔**。只要協作者在發 PR 時順手把 `.github/workflows/` 刪掉或改爛，就能直接取得綠燈繞過檢查。

本架構透過以下 **五道防線** 徹底堵死所有繞過路徑：

1. **裁判基準隔離（`on: pull_request_target`）**：
   強制 GitHub **只讀取並執行 `main` 分支裡**的 CI 腳本。即使發 PR 的人把整個 `.github/` 資料夾都刪除，系統依然會以 `main` 的規則進行審查。
2. **受測對象鎖定（`ref: ${{ github.event.pull_request.head.sha }}`）**：
   搭配 `pull_request_target` 使用，強制將發 PR 者的最新程式碼（`head`）抓進來給 `main` 的規則檢查，實現「規則」與「受測程式碼」分離。
3. **歷史舊檔豁免（`git diff --diff-filter=ACMR`）**：
   僅針對本次 PR 新增（A）、複製（C）、修改（M）、重命名（R）的檔案進行掃描，完全略過 `main` 既有的歷史舊檔案，避免無關的 PR 被歷史遺留問題卡死。
4. **規則檔所有權鎖（`.github/CODEOWNERS`）**：
   搭配 Rulesets 的 `Require review from Code Owners`，確保任何動到 `.github/` 目錄的 PR，沒有經過指定管理員 Approve 絕對無法合併進 `main`（防止他人趁機修改或刪除 CI 規則）。
5. **合併大門鎖死（GitHub Rulesets）**：
   禁止直接 `git push` 進 `main`、將 `check-tabs` 設為必過條件，且 `Bypass list` 保持空白（連管理員也受限），檢查失敗時伺服器端徹底鎖死 Merge 按鈕。

---

## 三、 詳細實作步驟（Windows + Fork / CMD 環境）

### 階段 1：建立 GitHub 公開倉庫並 Clone 到本地

1. 登入 GitHub 網頁版，點擊右上角 **`+`** → **New repository**。
2. **Repository name**：輸入倉庫名稱（例如 `ci-format-test`）。
3. **Visibility**：務必選擇 **Public**。
4. 勾選 **Add a README file**（自動建立預設的 `main` 分支）。
5. 點擊 **Create repository**，並複製倉庫網址，透過 **Fork**（或其他 Git 客戶端）將專案 Clone 到本地資料夾（例如 `C:\Users\你的帳號\Documents\Code\ci-format-test`）。

---

### 階段 2：確認本地 Git 提交者身分（避免 Commit 報錯）

若這台電腦尚未設定過 Git 身分，提交時會出現 `Author identity unknown` 錯誤；此外，若電腦僅安裝 Fork 而未單獨安裝全域 Git，在一般 Windows CMD 輸入 `git` 會出現 `'git' 不是內部或外部命令`。請用以下任一方式完成一次設定：

* **方法 A：直接在 Fork 介面設定（最推薦）**
  1. 打開 **Fork**，點擊左上角選單 **File** → **Preferences**（快捷鍵 `Ctrl + ,`）。
  2. 切換到 **Git** 分頁。
  3. 在 **Full Name** 填入你的名稱，在 **Email** 填入你綁定 GitHub 的 Email（例如 `eat123apple@gmail.com`），點擊確定儲存。
* **方法 B：透過 Fork 內建終端機下指令**
  1. 點擊 Fork 右上角工具列的 **Open in Console**（此終端機會自動載入 Fork 內建的 Git 路徑）。
  2. 依序輸入以下兩行指令：
     ```cmd
     git config --global user.email "你的GitHub信箱@example.com"
     git config --global user.name "你的名稱"
     ```

---

### 階段 3：在 `main` 分支建立 CI 腳本與權限鎖定檔

確保目前本地端位於 **`main`** 分支，打開專案資料夾（或於專案根目錄開啟 Windows CMD），依序建立以下檔案：

#### 1. 建立 `.github\workflows` 資料夾
在 Windows CMD 中執行（不需加 `-p` 參數）：
```cmd
mkdir .github\workflows
```

#### 2. 建立 CI 檢查腳本 `.github\workflows\check-format.yml`
執行以下指令開啟記事本建立檔案：
```cmd
notepad .github\workflows\check-format.yml
```
將以下完整 YAML 內容貼入記事本並存檔（`Ctrl + S`）：

> **技術細節說明**：腳本採用 `python3 - << 'EOF'` 寫法避免多行字串引號遺漏；且直接掃描 `\t` 字元而非硬性規定行首空格數必須為 4 的倍數，可避免誤殺多行註解（如 `/* ... * ... */`）或長字串換行對齊的正常空格，同時已排除強制必須使用 Tab 的 `Makefile` 與 `.mk` 檔。

```yaml
name: Format Check

on:
  pull_request_target:
    branches: [ main ]

jobs:
  check-tabs:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout PR code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          fetch-depth: 0

      - name: Check for Tab characters (Enforce 4 spaces)
        env:
          BASE_SHA: ${{ github.event.pull_request.base.sha }}
          HEAD_SHA: ${{ github.event.pull_request.head.sha }}
        run: |
          python3 - << 'EOF'
          import subprocess, sys, os

          base_sha = os.environ["BASE_SHA"]
          head_sha = os.environ["HEAD_SHA"]

          cmd = ["git", "diff", "--name-only", "--diff-filter=ACMR", base_sha, head_sha]
          files = subprocess.check_output(cmd).decode("utf-8").splitlines()

          failed = False

          for f in files:
              filename = os.path.basename(f)
              # 排除 .github 設定檔與強制規定必須使用 Tab 的 Makefile
              if f.startswith(".github/") or filename == "Makefile" or f.endswith(".mk") or not os.path.isfile(f):
                  continue
              try:
                  with open(f, "rb") as fp:
                      raw = fp.read()
                  if b"\x00" in raw:
                      continue
                  lines = raw.decode("utf-8").splitlines()

                  for idx, line in enumerate(lines, start=1):
                      if "\t" in line:
                          print(f"❌ [格式錯誤] {f} 第 {idx} 行包含 Tab 字元（請強制使用 4 個空格縮排）")
                          failed = True
              except UnicodeDecodeError:
                  continue

          if failed:
              sys.exit(1)
          print(f"✅ 已檢查 {len(files)} 個異動檔案，無 Tab 字元，縮排全數通過！")
          EOF
```

#### 3. 建立規則檔所有權鎖定檔 `.github\CODEOWNERS`
注意寫入的是**檔案內容**而非終端機指令，且格式有嚴格規定：
* **若使用 Email 指定審核者（不可加 `@`）**：
  ```cmd
  echo /.github/ 你的GitHub信箱@example.com > .github\CODEOWNERS
  ```
* **若使用 GitHub 帳號名稱指定審核者（必須加 `@`）**：
  ```cmd
  echo /.github/ @你的GitHub帳號 > .github\CODEOWNERS
  ```

#### 4. （選用）建立歷史遺留違規檔 `legacy.txt`
若想驗證「舊檔案不會被誤報」，可建立一個 `legacy.txt`，裡面故意按下 **Tab 鍵**縮排並存檔，一同推上 `main` 分支作為對照組。

---

### 階段 4：將初始規則推上 `main` 分支

1. 回到 **Fork** 視窗，在中間上方的 **Unstaged** 區塊點擊 **Stage**，將 `.github/workflows/check-format.yml`、`.github/CODEOWNERS`（以及 `legacy.txt`）移至 Staged 區。
2. 在右下角 **Commit subject** 輸入：`chore: setup anti-bypass CI format check`。
3. 點擊 **Commit**，接著點擊左上角工具列的 **Push**，將規則正式推送到 GitHub 的 `main` 分支。

---

### 階段 5：發起首個測試 PR（向 GitHub 註冊檢查項目）

GitHub 的規則設定介面必須在該 Job **至少執行過一次**後，才搜尋得到它的名稱。

1. 在 Fork 上方工具列點擊 **Branch**，建立並切換到新分支 `test/init-ci`。
2. 在專案資料夾新增一個正常的文字檔 `test.txt`（輸入一行不含 Tab 的文字，例如 `hello world`）並存檔。
3. 回到 Fork，將 `test.txt` 執行 **Stage** → **Commit** → **Push** 推上遠端。
4. 打開 **GitHub 網頁版**專案頁面，點擊黃色提示條的 **Compare & pull request** → **Create pull request**。
5. 等待約 5～10 秒，確認 PR 頁面下方出現綠燈：`Format Check / check-tabs (pull_request_target) — Successful`（**此時先不要按下 Merge pull request**）。

---

### 階段 6：設定 Rulesets 徹底鎖死 `main` 分支

1. 在 GitHub 專案頁面，點擊上方分頁列的 **Settings**。
2. 在左側邊欄找到 **Code, planning, and automation** → 點擊 **Rules** → **Rulesets**。
3. 點擊右上角綠色的 **New ruleset** → 選擇 **New branch ruleset**。
4. 設定基本資訊與對象：
   * **Ruleset Name**：輸入 `main-protect`。
   * **Enforcement status**：點擊下拉選單，將預設的 `Disabled` 切換為 **`Active`**（**必改**，否則規則不會生效）。
   * **Bypass list**：**保持空白**（不加入任何人，確保連你身為管理員也受規則限制，才能驗證鎖死效果；若未來正式上線需要管理員直接合併特權，再將自己加入此清單）。
   * **Target branches**：點擊右側的 **Add target** → 選擇 **Include default branch**（自動套用至 `main`）。
5. 往下捲動至 **Branch rules** 區塊，勾選以下兩項關鍵規則：
   * ☑️ **Require a pull request before merging**
     * 勾選後展開細項，務必勾選 ☑️ **Require review from Code Owners**（鎖死 `.github/` 資料夾修改權）。
   * ☑️ **Require status checks to pass**
     * 點擊展開後的 **Add checks** 按鈕。
     * 在搜尋框輸入 `check-tabs`，點選搜尋結果中的 **`check-tabs`** 加入必過清單。
6. 捲動至頁面最下方，點擊綠色的 **Create** 儲存規則。

---

## 四、 完整驗收與防繞過測試流程

回到剛剛開啟的 PR 頁面（`test/init-ci` 合併至 `main`），你會看到 `Format Check / check-tabs` 右側已出現灰底的 **`Required`** 標籤。接著執行以下測試：

### 測試 1：故意使用 Tab ＋ 企圖刪除 `.github` 繞過檢查
1. 確認 Fork 目前切換在 **`test/init-ci`** 分支。
2. 打開 `test.txt`，在任意行按下 **Tab 鍵** 進行縮排並存檔。
3. **模擬繞過攻擊**：直接在本地資料夾將整個 **`.github` 資料夾刪除**。
4. 回到 Fork，將 `test.txt` 的修改與 `.github` 的刪除紀錄一併 **Stage** → **Commit** → **Push** 到 `test/init-ci`。
5. 回到 GitHub 網頁版重新整理該 PR 頁面（注意：GitHub 判定是否可合併只看**最新一筆 Commit** 的狀態），等待約 5 秒後驗證以下現象：
   * **防繞過成功**：即使 PR 分支內已完全刪除 `.github/`，系統依然強制執行了 `main` 的檢查腳本。
   * **精準抓錯與行號定位**：檢查狀態轉為紅叉 ❌，點擊該行右側的 **Details**（或切換至上方的 **Checks** 分頁），展開 `Check for Tab characters` 步驟，可看到精準的錯誤訊息：`❌ [格式錯誤] test.txt 第 X 行包含 Tab 字元（請強制使用 4 個空格縮排）`。
   * **歷史舊檔不誤報**：原本留在 `main` 裡包含 Tab 的 `legacy.txt` 完全沒有被報錯。
   * **合併按鈕徹底鎖死**：下方的 **Merge pull request** 按鈕變成反灰無法點擊，右上角 `Merge status` 同步顯示紅叉阻擋。

### 測試 2：還原 `.github` 並改用 4 空格縮排（解除鎖定）
1. 在本地端從 `main` 還原 `.github/` 資料夾（於 Fork 內建終端機執行或手動複製回來）：
   ```cmd
   git checkout origin/main -- .github
   ```
2. 打開 `test.txt`，將裡面的 Tab 刪除並改為 **4 個空格** 縮排，存檔後重新 **Stage** → **Commit** → **Push** 到 `test/init-ci`。
3. 回到 GitHub PR 頁面等待約 5 秒，`check-tabs` 轉為綠燈 ✅，下方的 **Merge pull request** 按鈕隨即恢復為綠色可點擊狀態，實驗完成。

---

## 五、 移植至正式專案快速檢查表

日後要將這套防線套用到其他正式的 Public 專案時，只需依序完成以下 4 步：

- [ ] **1. 放入檢查腳本**：在正式專案的 `main` 分支建立 `.github/workflows/check-format.yml`（確認包含 `pull_request_target` 與 `head.sha`）。
- [ ] **2. 放入權限鎖檔**：在正式專案的 `main` 分支建立 `.github/CODEOWNERS`（鎖定 `/.github/` 審核權）。
- [ ] **3. 跑一次註冊 PR**：發起一個小 PR 讓 `check-tabs` 執行一次，使 GitHub 後台註冊該檢查名稱。
- [ ] **4. 開啟 Ruleset**：於 `Settings -> Rules -> Rulesets` 建立 `Active` 規則，套用至 `Include default branch`，勾選 `Require a pull request before merging`（含 `Require review from Code Owners`）與 `Require status checks to pass`（加入 `check-tabs`）。