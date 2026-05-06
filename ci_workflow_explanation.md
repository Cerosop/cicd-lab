# CI Workflow 說明

這份文件整理了這個專案的 GitHub Actions workflow 在做什麼、各個名詞的意思，以及我前面處理錯誤時的完整脈絡。

## 一、先說整體在做什麼

這個 workflow 的目的，是在每次 `push` 到 repository 時，自動幫你做三件事：

1. 檢查 TypeScript 程式碼型別是否正確
2. 檢查整份專案是否符合 Prettier 格式規範
3. 執行測試，並把測試結果顯示在 GitHub Actions 裡

如果其中任何一步回傳非 0 exit code，GitHub Actions 就會視為失敗。

---

## 二、名詞解釋

### 1. TypeScript typecheck

TypeScript typecheck 是指只做型別檢查，不真的輸出編譯後的 JavaScript。

在這個專案裡，對應的指令是：

```bash
npm run typecheck
```

它實際執行的是 `tsc --noEmit`，意思是：

- `tsc`：TypeScript compiler
- `--noEmit`：只檢查型別，不產生輸出檔

這一步的作用是確認程式碼有沒有型別錯誤，例如：

- 傳錯參數型別
- 少寫必要欄位
- 函式回傳型別不符

只要型別有問題，這一步就會失敗。

### 2. Prettier

Prettier 是程式碼格式化工具，用來統一排版風格。

在這個專案裡，對應的檢查指令是：

```bash
npm run format:check
```

它實際執行的是 `prettier --check .`，意思是檢查整個專案的檔案格式是否符合 Prettier 規則。

Prettier 會管這些事情：

- 縮排
- 引號風格
- 行尾格式
- 括號與空白排版

它不是在看你的程式邏輯對不對，而是在看格式整不整齊、風格有沒有統一。

如果檔案格式不符合規範，這一步就會失敗。

### 3. Test

Test 是測試程式是否符合預期。

這個專案裡的指令是：

```bash
npm run test
```

實際對應到 `vitest run`。

測試的用途是確認像這些功能有沒有正常：

- API 是否回傳正確結果
- 頁面或函式是否符合預期
- 修改程式後，有沒有把舊功能弄壞

### 4. Test Reporter

`dorny/test-reporter@v3` 是一個 GitHub Action，用來把測試結果顯示成 GitHub Actions 的報告。

它不是負責跑測試本身，而是負責：

- 讀取測試輸出檔
- 把結果轉成 GitHub 可以顯示的 check run
- 顯示通過、失敗、跳過的測試數量

這次我讓 Vitest 輸出 JUnit XML，再交給這個 action 解析。

---

## 三、YAML 的功能與內容對應

下面把 workflow 的每一段和它的功能對起來。

### 1. `on: push`

```yaml
on:
  push:
    branches:
      - '**'
```

這代表：只要有任何 branch 發生 push，就會觸發這個 workflow。

也就是說，每次你把程式碼推上去，GitHub Actions 就會自動跑 CI。

### 2. `permissions`

```yaml
permissions:
  contents: read
  actions: read
  checks: write
```

這是設定 GitHub Actions 的權限。

- `contents: read`：可以讀 repository 內容
- `actions: read`：可以讀 Actions 相關資訊
- `checks: write`：可以建立或更新 check run，也就是讓 test-reporter 顯示報告

沒有 `checks: write` 的話，test-reporter 可能無法正常建立報告。

### 3. `jobs.test`

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
```

這代表有一個叫 `test` 的 job，會跑在 GitHub 提供的 Ubuntu 虛擬機上。

### 4. Checkout code

```yaml
- name: Checkout code
  uses: actions/checkout@v5
```

這一步是把 repository 的程式碼抓到 runner 裡，因為 GitHub Actions 預設不會自動有你的原始碼。

### 5. Setup Node.js

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v5
  with:
    node-version: '22'
```

這一步是在 runner 上安裝 Node.js 22。

因為這個專案是 Node.js 專案，所以需要先把執行環境準備好，後面 `npm ci`、`npm run typecheck`、`npm run test` 才能執行。

### 6. Install dependencies

```yaml
- name: Install dependencies
  run: npm ci
```

這一步會根據 `package-lock.json` 安裝專案依賴。

`npm ci` 比 `npm install` 更適合 CI，因為它：

- 安裝速度快
- 依賴版本固定
- 更適合自動化環境

### 7. Run TypeScript typecheck

```yaml
- name: Run TypeScript typecheck
  run: npm run typecheck
```

這一步是做型別檢查，確認 TypeScript 程式碼沒有型別錯誤。

對應到前面的名詞解釋，就是 `tsc --noEmit`。

### 8. Run Prettier check

```yaml
- name: Run Prettier check
  run: npm run format:check
```

這一步是檢查專案格式是否符合 Prettier 規範。

如果有某些檔案縮排、引號、格式不合規，就會在這裡失敗。

### 9. Run tests

```yaml
- name: Run tests
  id: tests
  run: |
    mkdir -p reports
    npm run test -- --reporter=default --reporter=junit --outputFile=reports/vitest-junit.xml
```

這一步會執行測試，並且把結果輸出到 `reports/vitest-junit.xml`。

拆開來看：

- `mkdir -p reports`：先建立 reports 資料夾
- `npm run test`：執行 Vitest 測試
- `--reporter=default`：保留一般測試輸出
- `--reporter=junit`：額外輸出 JUnit 格式
- `--outputFile=reports/vitest-junit.xml`：把 JUnit 報告寫到這個檔案

這樣做的原因是，`test-reporter` 需要一個測試結果檔案來解析。

### 10. Test Report

```yaml
- name: Test Report
  uses: dorny/test-reporter@v3
  if: ${{ !cancelled() && hashFiles('reports/vitest-junit.xml') != '' }}
  with:
    name: Vitest Results
    path: reports/vitest-junit.xml
    reporter: jest-junit
```

這一步是把測試結果顯示成 GitHub 的報告。

各個參數意思如下：

- `name: Vitest Results`：報告名稱
- `path: reports/vitest-junit.xml`：要讀取的測試結果檔
- `reporter: jest-junit`：檔案格式類型，告訴 action 這是 JUnit 類型的結果
- `if: ${{ !cancelled() && hashFiles('reports/vitest-junit.xml') != '' }}`：只有在 workflow 沒被取消，而且檔案真的存在時才執行

---

## 四、剛剛發生了什麼事

一開始我是在把你的 workflow 做成符合作業要求：

1. 觸發條件改成 push
2. 加入 typecheck
3. 加入 Prettier check
4. 加入 test
5. 再接上 test-reporter

後來你說 push 上去之後 Action 顯示錯誤，我先以為是 test-reporter 的問題，所以去查 GitHub Actions 的實際執行 log。

我後來發現：

- 真正失敗的是 `Run Prettier check`
- 不是測試，也不是 test-reporter
- log 顯示有幾個檔案格式不符合 Prettier 規範

被指出的檔案包含：

- `.github/workflows/ci_314553010.yaml`
- `docker-compose.yml`
- `snippets/01_hello.yaml`
- `snippets/02_run-test.yaml`

也就是說，CI 的失敗原因其實是「格式檢查不通過」，不是「功能測試沒過」。

---

## 五、怎麼解決的

我做了這些處理：

1. 用 Prettier 重新格式化那幾個檔案
2. 再本地確認：
   - `npm run format:check` 通過
   - `npm run typecheck` 通過
   - `npm run test` 通過
3. 同時把 workflow 的 action 版本升級到較新的版本，減少 Node 20 deprecation 警告

所以最後的結果是：

- 這份 workflow 的內容符合 CI 要求
- 格式檢查可以通過
- 型別檢查可以通過
- 測試可以通過

---

## 六、這份 YAML 最後的用途，用一句話說

這份 workflow 的用途就是：在每次 push 時，自動幫你檢查程式碼型別、格式和測試，並把測試結果顯示在 GitHub Actions 上，讓你快速知道這次改動有沒有把專案弄壞。

---

## 七、可以直接拿去交作業的簡短版本

這份 GitHub Actions workflow 是用來在 `push` 時自動執行 CI 檢查。它會先 checkout 程式碼、安裝 Node.js 22、執行 `npm ci` 安裝依賴，再依序做 TypeScript typecheck、Prettier 格式檢查與 Vitest 測試。測試結果會輸出成 JUnit XML，並交給 `dorny/test-reporter@v3` 顯示成 GitHub 的測試報告。這樣可以在每次推送後，自動確認程式碼有沒有型別錯誤、格式問題或測試失敗。