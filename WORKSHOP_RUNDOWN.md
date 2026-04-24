# momo Databricks AI/BI Hands-On Workshop

---

## 📋 大綱

### 🔧 上半場（約 90 分鐘）｜目標對象：資料倉儲人員

| 步驟 | 主題 |
|------|------|
| Step 01 | 確認/建立個人 Schema 並上傳資料 
| Step 02 | AI 自動生成資料表說明 
| Step 03 | 建立 Unity Catalog Metric View 
| Step 04 | 建立 Genie Space
| Step 05 | 建立 Evaluation Dataset
| Step 06 | 執行 Evaluation
| Step 07 | 建立 AI/BI Dashboard
| Step 08 | 設定權限開放給下半場參加者

### 👥 下半場（約 60 分鐘）｜目標對象：資料倉儲人員 ＋ 商品經理／行銷人員／資料分析師

| 步驟 | 主題 | 
|------|------|
| Step 09 | Workshop資料內容說明
| Step 10 | Databricks One 導覽
| Step 11 | Genie 問答操作 
| Step 12 | 資料倉儲人員查看 Monitor 反饋並優化 Genie 
| Step 13 | 再次問答驗證改善效果 
| Step 14 | 用 Genie Code 自助建立報表 
| Step 15 | Workshop 心得分享 

---

## 📦 資料說明

本次 workshop 使用三份模擬電商資料：

| 資料表 | 說明 | 資料量 |
|--------|------|--------|
| `ft_ord_order_aggr` | 訂單事實表（每筆訂單紀錄） | ~623 萬筆 |
| `dt_spc_goods` | 商品維度表（商品分類、品牌、供應商） | ~13 萬筆 |
| `cus_customer` | 客戶維度表（性別、生日） | ~48 萬筆 |

**資料關聯**：
```
ft_ord_order_aggr
  ├── goods_code → dt_spc_goods.goods_code  （訂單 → 商品）
  └── cust_no   → cus_customer.cust_no      （訂單 → 客戶）
```

---

# 🔧 上半場｜資料倉儲人員實作

---

## Step 01｜確認/建立個人 Schema 並準備資料

### 1-1 確認／建立 Schema

1. 點選左側導覽列 **Catalog**
2. 展開 **momo_workshop** catalog
3. 確認是否已有自己名稱的 schema
4. 如果無，點選右上角 **＋ Create schema**
5. Schema 名稱填入：`你的英文名字`（例如：`eric`）
   > ⚠️ 請使用小寫英文，不要有空格或特殊符號
6. 點選 **Create** 確認建立

---

### 1-2 準備資料（擇一）

#### 選項 A｜上傳 CSV 建立 Delta Table

對每一份 CSV 重複以下步驟（共 3 次）：

1. 在你的 Schema 下點選 **Create table**
2. 選擇 **Upload files** 分頁
3. 將 CSV 檔案拖曳上傳（或點選選取檔案）：
   - `ft_ord_order_aggr.csv`
   - `dt_spc_goods.csv`
   - `cus_customer.csv`
4. 確認 **Catalog** 為 `momo_workshop`，**Schema** 為你的 schema
5. Table name 保留 CSV 檔名（不含 `.csv`）
6. 點選 **Create** 完成建立

---

#### 選項 B｜從 momo_workshop.default 複製資料表（SQL 指令）

1. 點選左側導覽列 **SQL Editor**
2. 確認右上角 Warehouse 已選取
3. **將 `<your_schema>` 取代為你的 schema 名稱**，複製並執行以下 SQL：

```sql
-- 複製訂單事實表
CREATE OR REPLACE TABLE momo_workshop.<your_schema>.ft_ord_order_aggr
AS SELECT * FROM momo_workshop.default.ft_ord_order_aggr;

-- 複製商品維度表
CREATE OR REPLACE TABLE momo_workshop.<your_schema>.dt_spc_goods
AS SELECT * FROM momo_workshop.default.dt_spc_goods;

-- 複製客戶維度表
CREATE OR REPLACE TABLE momo_workshop.<your_schema>.cus_customer
AS SELECT * FROM momo_workshop.default.cus_customer;
```

> 💡 **提示**：此方式適合網路環境較慢或不方便上傳大型 CSV 的情況，直接複製線上資料表速度更快。

---

✅ **完成確認**：在 Catalog 的你的 schema 下應看到 3 張 table

---

## Step 02｜透過 AI 自動生成 Table Description

讓 AI 自動生成表格和欄位說明，幫助 Genie 更準確理解資料。

1. 點選左側導覽列 **Catalog**
2. 進入 `momo_workshop` → 你的 schema → 點選 `ft_ord_order_aggr`
3. 在 Overview 會看到AI Suggested Description視窗 ，點選**Accept** 按鈕
4. 下面表格右上角可以點選**AI generate** 按鈕生成欄位描述，點選**Save all** 按鈕
5. 重複對 `dt_spc_goods` 和 `cus_customer` 執行相同操作

> 💡 **小提示**：AI 生成的說明可以手動編輯修改，幫助 Genie 對業務術語有更好的理解

✅ **完成確認**：三張表格都有 AI 生成的 Description

---

## Step 03｜在 SQL Editor 建立 UC Metric View

Metric View 是 Databricks Unity Catalog 的業務語意層，可以定義標準化指標讓 Dashboard 和 Genie 共用。

### 3-1 開啟 SQL Editor

1. 點選左側導覽列 **SQL Editor**
2. 確認右上角 Warehouse 已選取（若無，選擇任一可用的 SQL warehouse）

### 3-2 執行建立 Metric View 的 SQL

複製以下 SQL，**將 `<your_schema>` 取代為你的 schema 名稱**，貼上後執行：

```sql
CREATE OR REPLACE VIEW momo_workshop.<your_schema>.ecommerce_metrics
WITH METRICS
LANGUAGE YAML
AS $$
  version: 1.1
  comment: "momo電商核心業務指標視圖，整合訂單與商品資料"
  source: momo_workshop.<your_schema>.ft_ord_order_aggr
  joins:
    - name: goods
      source: momo_workshop.<your_schema>.dt_spc_goods
      on: source.goods_code = goods.goods_code
  dimensions:
    - name: 訂單月份
      expr: DATE_TRUNC('MONTH', order_time)
      comment: "訂單所在月份"
    - name: 訂單日期
      expr: DATE(order_time)
      comment: "訂單日期"
    - name: 通路代碼
      expr: media_gb
      comment: "媒體通路代碼"
    - name: 商品類型
      expr: party_type
      comment: "1P自營 / 3P第三方"
    - name: 一級分類
      expr: goods.level1_name
      comment: "商品第一層分類"
    - name: 二級分類
      expr: goods.level2_name
      comment: "商品第二層分類"
    - name: 三級分類
      expr: goods.level3_name
      comment: "商品第三層分類"
    - name: 品牌名稱
      expr: goods.brand_name
      comment: "商品品牌"
    - name: 供應商名稱
      expr: goods.entp_name
      comment: "供應商名稱"
  measures:
    - name: 總營收
      expr: SUM(order_amt)
      comment: "訂購金額加總（元）"
    - name: 訂單數
      expr: COUNT(DISTINCT order_no)
      comment: "不重複訂單數"
    - name: 客戶數
      expr: COUNT(DISTINCT cust_no)
      comment: "不重複客戶數"
    - name: 總銷售量
      expr: SUM(order_qty)
      comment: "訂購數量加總"
    - name: 平均客單價
      expr: ROUND(SUM(order_amt) / COUNT(DISTINCT order_no), 0)
      comment: "平均每筆訂單金額（元）"
  materialization:
    schedule: every 4 hours
    mode: relaxed
    materialized_views:
      - name: monthly_by_category
        type: aggregated
        dimensions:
          - 訂單月份
          - 一級分類
          - 商品類型
        measures:
          - 總營收
          - 訂單數
          - 客戶數
      - name: monthly_by_channel
        type: aggregated
        dimensions:
          - 訂單月份
          - 通路代碼
          - 商品類型
        measures:
          - 總營收
          - 訂單數
          - 平均客單價
$$
```

> 💡 **Materialization 說明**：每 4 小時自動預先計算並快取指標結果，讓 Dashboard 和 Genie 查詢時大幅加速。`mode: relaxed` 表示快取更新期間仍可查詢，不會中斷服務。

### 3-3 驗證 Metric View 是否正常

```sql
SELECT
  `訂單月份`,
  `商品類型`,
  MEASURE(`總營收`) AS total_revenue,
  MEASURE(`訂單數`) AS total_orders,
  MEASURE(`客戶數`) AS total_customers,
  MEASURE(`平均客單價`) AS avg_order_value
FROM momo_workshop.<your_schema>.ecommerce_metrics
GROUP BY ALL
ORDER BY `訂單月份`
LIMIT 20;
```

✅ **完成確認**：看到每月的業績數字，且資料按月份排列

---

## Step 04｜建立 Genie Space

> 📌 **重點說明**：這個步驟我們會遵照 Genie best practice 建立一個具備基本能力的 Genie Space，但刻意預留部分調整空間（如某些業務術語未完整說明），讓下半場的 Evaluation 和使用者反饋能展示真實的改善循環。

### 4-1 建立 Genie Space

1. 點選左側導覽列 **Genie spaces**
2. 點選右上角 **New**
3. 在 **Tables** 區域加入以下資料表：
   - `momo_workshop.<your_schema>.ft_ord_order_aggr`
   - `momo_workshop.<your_schema>.dt_spc_goods`
   - `momo_workshop.<your_schema>.cus_customer`
   - `momo_workshop.<your_schema>.ecommerce_metrics`（Metric View）
4. 點擊 **Create**
5. 可於右側About分頁中，點選鉛筆符號重新命名：`momo電商問答_你的名字`
6. 將你的**Space ID**記錄下來，後面章節會使用到

### 4-2 填寫 Instructions

點選 **Configure**，在 **Instructions** 分頁貼上，填寫完點擊**Save**：

```
你是 momo 購物網路的資料分析助手，協助用戶分析電商訂單、商品和客戶資料。

【資料表說明】
- ft_ord_order_aggr：訂單事實表，每一列代表一筆訂單中的一個商品
  - order_time: 訂單成立時間
  - order_no: 訂單編號
  - cust_no: 客戶編號
  - goods_code: 商品編號
  - order_amt: 訂購金額（新台幣元）
  - order_qty: 訂購數量
  - party_type: 商品區分（'1P'=自營商品, '3P'=第三方賣家商品）

- dt_spc_goods：商品維度表，記錄商品的分類、品牌和供應商資訊
  - goods_code: 商品編號（用於與訂單表 JOIN）
  - brand_name: 品牌名稱
  - level1_name ~ level4_name: 商品分類（第一層最大，第四層最細）
  - entp_name: 供應商名稱

- cus_customer：客戶維度表
  - cust_no: 客戶編號（用於與訂單表 JOIN）
  - sex: 性別（'1'=男性, '2'=女性）
  - birthday: 生日（格式 YYYYMMDD，如 19901231）

【指標定義】
- 總營收 = SUM(order_amt)
- 訂單數 = COUNT(DISTINCT order_no)
- 客戶數 = COUNT(DISTINCT cust_no)
- 平均客單價 = SUM(order_amt) / COUNT(DISTINCT order_no)
- 「最近」預設指最近 3 個月，除非用戶明確指定日期

【重要規則】
- 分析商品資訊時，請務必 JOIN dt_spc_goods
- 分析客戶資訊時，請務必 JOIN cus_customer
- 日期篩選請使用 DATE_TRUNC 或 YEAR/MONTH 函數
```

> ⚠️ **注意**：`media_gb` 通路代碼的對應（'01'=電視購物, '07'=網路購物）這個步驟**刻意不填入** Instructions，留作後續 Evaluation 顯示改善空間用。

### 4-3 設定 Join Relationships

在 **Joins** 區域點選 **Add**，新增兩條關聯：

**關聯 1：訂單 → 商品**

| 欄位 | 設定值 |
|------|--------|
| Left table | `ft_ord_order_aggr` |
| Left column | `goods_code` |
| Relationship type | **Many to one** |
| Right table | `dt_spc_goods` |
| Right column | `goods_code` |

**關聯 2：訂單 → 客戶**

| 欄位 | 設定值 |
|------|--------|
| Left table | `ft_ord_order_aggr` |
| Left column | `cust_no` |
| Relationship type | **Many to one** |
| Right table | `cus_customer` |
| Right column | `cust_no` |

### 4-4 新增 Example SQL

在 **SQL Queries** 區域，點選 **Add → Example query**，新增以下範例：

**範例 1：每月總營收**
- 問題：`每月的總營收是多少？`
```sql
SELECT
  DATE_TRUNC('MONTH', order_time) AS order_month,
  SUM(order_amt) AS total_revenue,
  COUNT(DISTINCT order_no) AS total_orders,
  COUNT(DISTINCT cust_no) AS total_customers
FROM ft_ord_order_aggr
GROUP BY 1
ORDER BY 1
```

點選 **Save** 儲存設定。

**範例 2：前 N 大品牌營收（含參數）**
- 問題：`哪些品牌的銷售額最高？`
```sql
SELECT
  g.brand_name,
  SUM(o.order_amt) AS total_revenue,
  COUNT(DISTINCT o.order_no) AS total_orders
FROM ft_ord_order_aggr o
JOIN dt_spc_goods g ON o.goods_code = g.goods_code
WHERE g.brand_name IS NOT NULL
  AND YEAR(o.order_time) = :year
GROUP BY 1
ORDER BY 2 DESC
LIMIT :top_n
```

參數設定：

| Parameter name | Type | Default value |
|----------------|------|---------------|
| `year` | Integer | `2025` |
| `top_n` | Integer | `10` |

點選 **Save** 儲存設定。

**範例 3：1P vs 3P 分析**
- 問題：`1P 和 3P 的業績各是多少？`
```sql
SELECT
  CASE party_type
    WHEN '1P' THEN '自營商品 (1P)'
    WHEN '3P' THEN '第三方商品 (3P)'
    ELSE party_type
  END AS channel_type,
  SUM(order_amt) AS total_revenue,
  COUNT(DISTINCT order_no) AS total_orders,
  ROUND(SUM(order_amt) / COUNT(DISTINCT order_no), 0) AS avg_order_value
FROM ft_ord_order_aggr
GROUP BY channel_type
ORDER BY total_revenue DESC
```

點選 **Save** 儲存設定。

✅ **完成確認**：Genie Space 已建立，包含 Instructions、Relationships、3 個 Example SQL

---

## Step 05｜建立 Evaluation Dataset

Evaluation Dataset 讓你用標準問答組評估 Genie 的準確度。

### 5-1 新增 Evaluation Questions

在 Genie Space 頁面，點選 **Benchmark** 分頁 → **Add benchmark**

> ⚠️ **注意**：Expected SQL 中的 `<your_schema>` 請替換成你自己的 schema 名稱。

---

**評估問題 1**
- 問題：`2025 年全年的總營收是多少？`
- 預期 SQL：
```sql
SELECT SUM(order_amt) AS total_revenue
FROM momo_workshop.<your_schema>.ft_ord_order_aggr
WHERE YEAR(order_time) = 2025
```

---

**評估問題 2**
- 問題：`前 5 大商品一級分類的總營收排名？`
- 預期 SQL：
```sql
SELECT
  g.level1_name,
  SUM(o.order_amt) AS total_revenue
FROM momo_workshop.<your_schema>.ft_ord_order_aggr o
JOIN momo_workshop.<your_schema>.dt_spc_goods g ON o.goods_code = g.goods_code
WHERE g.level1_name IS NOT NULL
GROUP BY 1
ORDER BY 2 DESC
LIMIT 5
```

---

**評估問題 3**
- 問題：`電視購物和網路購物的業績比較`
- 預期 SQL：
```sql
SELECT
  CASE media_gb
    WHEN '01' THEN '電視購物'
    WHEN '07' THEN '網路購物'
    ELSE CAST(media_gb AS STRING)
  END AS channel_name,
  SUM(order_amt) AS total_revenue,
  COUNT(DISTINCT order_no) AS total_orders
FROM momo_workshop.<your_schema>.ft_ord_order_aggr
WHERE media_gb IN ('01', '07')
GROUP BY media_gb
ORDER BY total_revenue DESC
```

---

**評估問題 4**
- 問題：`1P 和 3P 各自的平均客單價是多少？`
- 預期 SQL：
```sql
SELECT
  party_type,
  ROUND(SUM(order_amt) / COUNT(DISTINCT order_no), 0) AS avg_order_value
FROM momo_workshop.<your_schema>.ft_ord_order_aggr
GROUP BY 1
ORDER BY 1
```

---

## Step 06｜執行 Evaluation（發現改善空間）

1. 在 Benchmark 分頁確認問題清單無誤
2. 點選 **Run all benchmarks**
3. 等待執行完成（約 1-2 分鐘）

### 6-1 查看評估結果

每道題執行完後會產生以下資訊：

- ✅ **綠底文字表示Good**：Genie 生成的 SQL 執行結果與 Expected SQL 一致
- ❌ **紅底文字表示Bad**：結果不符或 SQL 執行失敗

> 📌 **預期觀察**：至少兩題評估結果為 Bad，但原因不同：
> - **電視購物和網路購物的業績比較**：Genie 不知道 `media_gb` 代碼對應（'01'=電視購物, '07'=網路購物），可能用錯誤欄位推算通路，屬於**語意錯誤**。
> - **1P 和 3P 各自的平均客單價**：Genie 計算結果正確，但輸出欄位名稱和排序方向與 ground truth 不同，屬於**格式不穩定**而非真正錯誤。

### 6-2 到Genie Space對話框進行問答

在開始提問前，先了解 Genie 的三種問答模式：

| 模式 | 運作方式 | 適合情境 |
|------|----------|----------|
| **Chat — Do not inspect answer** | 直接產生 SQL 並執行，不做額外驗證 | 簡單彙總、排序、切片等 straightforward 數據問題 |
| **Chat — Inspect answer** | 先產一版 SQL，再自動跑數個小 SQL 驗證關鍵邏輯（filter 值、聚合邏輯等），必要時重新產一版改良 SQL，回傳更可信的結果 | 需要較高準確度的查詢 |
| **Agent** | 先擬定 research plan，跑多個 SQL 反覆迭代，最後產出帶敘述、圖表、引用步驟的完整報告 | 複雜的多面向分析問題 |

現在你可切換到對話框進行問答，如同評估結果，在特定問題表現上並不穩定，我們會在下半場進行優化

---

## Step 07｜建立 AI/BI Dashboard

Dashboard 的內容設計涵蓋三種角色的使用需求：
- **商品經理**：商品分類業績、1P vs 3P 佔比、品牌排名
- **行銷人員**：通路業績比較、月度趨勢
- **資料分析師**：綜合指標、訂單量、客單價趨勢

### 7-1 新增 Dashboard

1. 點選左側導覽列 **Dashboards**
2. 點選右上角 **Create dashboard**
3. 命名為：`momo電商分析儀表板_你的名字`

### 7-2 新增資料集（Datasets）

在 Dashboard 編輯頁面：

1. 點選 **Data** 分頁
2. 點選 **Add dataset**，選擇 **From table**，加入以下資料集：
   - `momo_workshop.<your_schema>.ecommerce_metrics`（Metric View）
   - `momo_workshop.<your_schema>.ft_ord_order_aggr`
   - `momo_workshop.<your_schema>.dt_spc_goods`
   - `momo_workshop.<your_schema>.cus_customer`

### 7-3 使用 Genie Code 生成 Widget

切換到 **Canvas** 分頁，開啟右上方 **Genie Code**，複製以下 Prompt 生成圖表（**⏱ 預計時間：10-15 分鐘**）：

```
Widget 1：月度總營收趨勢（行銷人員 / 資料分析師）
使用 ecommerce_metrics 資料集，建立折線圖顯示每月總營收的趨勢變化。
X 軸為訂單月份，Y 軸為總營收（元），按時間由舊到新排列，線條顏色為藍色。

Widget 2：1P vs 3P 營收佔比（商品經理）
使用 ecommerce_metrics 資料集，建立圓餅圖比較商品類型（party_type）的總營收佔比。
1P 代表自營商品，3P 代表第三方商品，標籤顯示類型名稱及百分比。

Widget 3：前 10 大商品分類營收排名（商品經理）
使用 ecommerce_metrics 資料集，建立水平長條圖顯示一級分類的總營收排名。
只顯示營收最高的前 10 個分類，依總營收由高到低降序排列，Y 軸為分類名稱，X 軸為總營收。

Widget 4：月度訂單數趨勢（資料分析師）
使用 ecommerce_metrics 資料集，建立折線圖顯示每月訂單數的趨勢。
X 軸為訂單月份，Y 軸為訂單數，按時間由舊到新排列。

Widget 5：電視購物 vs 網路購物月度業績（行銷人員）
使用 ecommerce_metrics 資料集，建立折線圖比較通路代碼 '01'（電視購物）和 '07'（網路購物）的每月總營收。
X 軸為訂單月份，Y 軸為總營收，兩條線以不同顏色區分，圖例標示通路名稱。

Widget 6：前 10 大品牌營收排名（商品經理）
使用 ecommerce_metrics 資料集，建立水平長條圖顯示品牌名稱的總營收排名。
只顯示前 10 大品牌，依總營收降序排列。
```

### 7-4 連結已建立的 Genie Space

將 Dashboard 的 Ask Genie 功能連結到 Step 04 建立的 Genie Space，讓使用者提問時能使用我們設定好的 Instructions 和 Example SQL。

1. 在 Dashboard 編輯頁面，點選右上角 **⋮** → **Settings and themes**
2. 找到 **General**分頁，確認**Enale Genie** 設定項已開啟
3. 選擇 **Link existing Genie space** 並填入**你的Genie ID**


> 💡 **說明**：之後publish dashboard 下方會出現 **Ask Genie** 按鈕，會改用我們的 Genie Space 回答問題，擁有完整的 Instructions 背景知識和 Example SQL，而非只查詢 Dashboard 已載入的資料集。

### 7-5 完成 Dashboard

1. 調整各圖表位置和大小，排列整齊
2. 點選右上角 **Publish** 並選擇**Share data permission(default)**發布 Dashboard
3. 彈跳出視窗可以加入其人users/groups/service principles共享或者複製embed code以iframe內嵌到其他應用程式
4. 點擊儀表板名稱右邊的**View published**按鈕
5. 右上角點選 **Schedule** 圖示
6. 設定更新頻率（例如：每天 08:00）
7. **Advanced settings**展開後可以查看更多資訊，例如收件人 Email等，點選 **Save**

### 7-6 用 Ask Genie 測試問答

> 📌 **Dashboard 的 Ask Genie 採用 Agent 模式**，會進行深度分析並產出結構化報告，而非只回傳單一查詢結果。

Dashboard 右下角點選 **Ask Genie**，試著輸入以下問題：

```
哪個商品分類的總營收最高？
```

✅ **完成確認**：Dashboard 顯示 6 張圖表，Ask Genie 已連結 Genie Space 並能正確回答問題

---

## Step 08｜設定權限開放給下半場參加者

### 8-1 開放 Dashboard 存取

1. 在 Dashboard 頁面點選右上角 **⋮** → **Share**
2. 搜尋並加入**workshop_users**
3. 設定權限為 **Can view**
4. 點選 **Save**

### 8-2 開放 Genie Space 存取

1. 在 Genie Space 頁面點選右上角 **⋮** → **Share**
2. 同樣加入**workshop_users**，設定 **Can view**
3. 點選 **Save**

✅ **完成確認**：下半場非技術人員可以在 Databricks One 看到你的 Dashboard 和 Genie Space

---

# 👥 下半場｜跨角色互動體驗

> **參加者**：上半場資料倉儲人員 ＋ 商品經理、行銷人員、資料分析師

---

## Step 09｜Databricks One 導覽（分角色場景）

Databricks One 是整合 Dashboard 和 Genie 的入口，適合提供給非技術業務同仁使用。

### 9-1 切換到 Databricks One

1. 點選左上角 Databricks Logo 旁的 **App Switcher**
2. 選擇 **Databricks One**

進入後瀏覽以下主要功能區塊：

**🔍 Search（搜尋）**
- 頁面頂部的搜尋列，可以直接輸入關鍵字搜尋 Dashboard、Genie Space、資料表等資源
- 輸入 `momo` 即可快速找到本次 workshop 建立的儀表板和問答空間
- 支援模糊搜尋，適合快速定位資產

**💬 Ask（提問）**
- 搜尋列右側或首頁的 **Ask** 入口，是 Databricks One 的自然語言問答功能
- 可以直接用中文提問，系統會自動路由到合適的 Genie Space 回答
- 例如輸入「電視購物業績如何？」會自動找到相關的 Genie Space 並查詢


---

### 9-2 角色場景導覽

各角色請找到上半場倉儲人員建立的 **momo電商分析儀表板**，依照以下場景查看對應圖表或者點擊**Ask Genie**協助深度探索內容：

---

#### 🛍️ 商品經理場景

**任務背景**：你是 momo 的商品經理，主管要求你在下週例會提出 Q2 商品策略調整建議。

**要回答的問題**：
1. 目前業績最好的前 3 個商品一級分類是哪些？我們應該加碼投資哪些分類？
2. 自營商品（1P）和第三方商品（3P）的業績佔比如何？策略上應如何調整？
3. 品牌排名前 3 是哪些？這些品牌的客戶是否有特殊偏好？

**看哪些 Widget**：
- Widget 3：前 10 大商品分類營收排名 → 找出重點投資分類
- Widget 2：1P vs 3P 營收佔比 → 評估自營 vs 第三方策略
- Widget 6：前 10 大品牌排名 → 品牌合作優先序

---

#### 📣 行銷人員場景

**任務背景**：你是 momo 的行銷企劃，正在規劃下一季的促銷活動節奏，需要了解哪個通路、哪個時間點最適合加大投入。

**要回答的問題**：
1. 電視購物和網路購物的業績比較，哪個通路成長更快？
2. 過去一年哪幾個月份的業績最高？可以推測什麼節慶效應？
3. 訂單數趨勢和營收趨勢是否一致？有沒有客單價偏高但訂單數低的時期？

**看哪些 Widget**：
- Widget 5：電視購物 vs 網路購物月度業績 → 通路策略依據
- Widget 1：月度總營收趨勢 → 找出業績高峰月份
- Widget 4：月度訂單數趨勢 → 交叉比對量與金額的差異

---

#### 📊 資料分析師場景

**任務背景**：你是 momo 的資料分析師，需要準備一份整體業績健康度報告，同時找出值得深挖的異常點。

**要回答的問題**：
1. 整體業績走勢是否健康？有無明顯的下滑或異常月份？
2. 哪些商品分類的成長速度超過平均？

**看哪些 Widget**：
- 全部 6 個 Widget → 建立全局觀
- 重點關注 Widget 1、4 的趨勢是否同步（量增但金額沒漲 = 客單價下降）

---

## Step 10｜Genie 問答操作（含特殊問題與反饋）

這次選擇特定Genie Space進入，體驗切換不同問答模式。

在開始提問前，先了解 Genie 的三種問答模式：

| 模式 | 運作方式 | 適合情境 |
|------|----------|----------|
| **Chat — Do not inspect answer** | 直接產生 SQL 並執行，不做額外驗證 | 簡單彙總、排序、切片等 straightforward 數據問題 |
| **Chat — Inspect answer** | 先產一版 SQL，再自動跑數個小 SQL 驗證關鍵邏輯（filter 值、聚合邏輯等），必要時重新產一版改良 SQL，回傳更可信的結果 | 需要較高準確度的查詢 |
| **Agent** | 先擬定 research plan，跑多個 SQL 反覆迭代，最後產出帶敘述、圖表、引用步驟的完整報告 | 複雜的多面向分析問題 |

### 10-1 一般問答（各角色依序嘗試）

各角色根據自己的場景嘗試 2-3 個一般問題，例如：

**商品經理**
```
2025 年前 5 大商品一級分類的訂單數排名？
```
```
2024 年前 5 大品牌銷售額
```

**行銷人員**
```
2025 年每月的總營收是多少？
```
```
1P 和 3P 的平均客單價各是多少？
```

**資料分析師**
```
男女客戶的消費差異？
```
```
最近 3 個月哪個商品分類的成長最快？
```

---

### 10-2 特殊問題（預期 Genie 無法有效回答）

請所有角色嘗試以下兩個特殊問題，觀察 Genie 的回應品質：

**特殊問題 1（通路術語不明確）**
```
電視購物和網路購物的業績比較
```
> 📌 **觀察**：Genie 不知道 `media_gb` 代碼對應關係，可能直接回傳代碼數字，無法顯示「電視購物」「網路購物」這樣友善的名稱，或甚至無法正確篩選。

**特殊問題 2（資料範圍不明確）**
```
各通路的取消訂單比例是多少？
```
> 📌 **觀察**：現有資料集（`ft_ord_order_aggr`）只包含成立訂單，不含取消或退貨記錄。Genie 不知道這個限制，可能嘗試用現有欄位湊出一個「看起來合理」的答案，或直接回傳錯誤結果——這是比邏輯錯誤更危險的幻覺型失敗。

---

### 10-3 留下對話反饋

針對特殊問題的不佳回答：

1. 點選該回答右側的 **👎**（差評）
2. 點選 **Request review**
3. 留下評論，說明問題：
   - 特殊問題 1 可填：`Genie 不知道電視購物對應 media_gb='01'，網路購物對應 '07'，請修正`
   - 特殊問題 2 可填：`此資料集沒有取消訂單記錄，Genie 不應該回答這道題，請修正`

✅ **完成確認**：至少提交 1 則差評反饋

---

## Step 11｜倉儲人員查看 Monitor 反饋並優化 Genie

> **操作角色**：上半場資料倉儲人員

### 11-1 查看 Monitor 統計

1. 切換回 Databricks 主介面（非 Databricks One）
2. 進入你的 Genie Space
3. 點選 **Monitor** 分頁
4. 查看：
   - 使用者反饋（差評列表）
   - 哪些問題最常被問
   - 哪些問題 Genie 無法回答

> 💡 **討論點：三種不同性質的失敗**
>
> Monitor 收到的反饋來自兩個來源（Step 6 Evaluation + Step 10 使用者差評），共呈現三種失敗類型：
>
> | 來源 | 問題 | 失敗原因 | 性質 | 修復方式 |
> |------|------|----------|------|----------|
> | Step 6 Evaluation | 電視購物和網路購物的業績比較 | Genie 不知道 `media_gb` 代碼對應，邏輯錯誤 | **術語不明確** | Instructions + Example SQL |
> | Step 6 Evaluation | 1P 和 3P 各自的平均客單價 | 計算正確但輸出格式不穩定，SQL-diff 誤判為 Bad | **格式不穩定**（非真正錯誤） | Example SQL 固定輸出格式 |
> | Step 10 使用者差評 | 各通路的取消訂單比例 | 資料集根本不含取消記錄，Genie 幻覺式推算 | **資料邊界不明確** | Instructions 定義資料範圍 |
>
> 這三種失敗對應三種不同的優化策略，說明 Monitor 不只是看「有幾個 Bad」，更重要的是**理解失敗的根本原因**，才能對症下藥。

### 11-2 根據反饋優化 Instructions

點選 **Edit**，補充兩項說明：

**① 通路代碼對應**（針對特殊問題 1）

在`- order_no: 訂單編號`下新增一列：

```
  - media_gb: 通路代碼（'01'=電視購物, '07'=網路購物，查詢通路業績時請使用這兩個值篩選）
```

**② 資料範圍限制**（針對特殊問題 2）

在 Instructions 末尾新增一段：

```
## 資料範圍限制
- 此 Genie Space 的資料僅包含**成立訂單**，不含取消或退貨記錄。
- 若使用者詢問退貨率、取消率、退款等相關問題，請明確告知此資料集不包含此類資訊，不要嘗試用現有欄位推算。
```

> 💡 **說明**：Instructions 的用途不只是補充術語，也能定義「Genie 不該回答什麼」。明確的資料範圍說明可以防止 Genie 在資料缺失時產生幻覺式答案，這對資料治理非常重要。

### 11-3 新增對應的 Example SQL

在 **SQL Queries** 點選 **Add → Example query**，新增：

**範例：電視購物 vs 網路購物業績比較**
- 問題：`電視購物和網路購物的業績比較`
```sql
SELECT
  CASE media_gb
    WHEN '01' THEN '電視購物'
    WHEN '07' THEN '網路購物'
    ELSE CAST(media_gb AS STRING)
  END AS channel_name,
  SUM(order_amt) AS total_revenue,
  COUNT(DISTINCT order_no) AS total_orders,
  ROUND(SUM(order_amt) / COUNT(DISTINCT order_no), 0) AS avg_order_value
FROM ft_ord_order_aggr
WHERE media_gb IN ('01', '07')
GROUP BY media_gb
ORDER BY total_revenue DESC
```

點選 **Save** 儲存。

### 11-4 補充 1P/3P 客單價的 Example SQL

雖然 Genie 對「1P 和 3P 各自的平均客單價」的計算結果正確，但輸出欄位名稱與排序方向不穩定。補充 Example SQL 可確保 Genie 輸出格式一致。

在 **SQL Queries** 點選 **Add → Example query**，新增：

**範例：1P 和 3P 平均客單價**
- 問題：`1P 和 3P 各自的平均客單價是多少？`
```sql
SELECT
  party_type,
  ROUND(SUM(order_amt) / COUNT(DISTINCT order_no), 0) AS avg_order_value
FROM ft_ord_order_aggr
GROUP BY party_type
ORDER BY party_type
```

點選 **Save** 儲存。

> 💡 **說明**：這正是 Genie best practice 的核心流程 — 不是一次把所有問題都預測到，而是**透過真實使用者反饋驅動持續優化**。Monitor 是這個循環的起點。

✅ **完成確認**：Instructions 已更新通路代碼說明、補充資料範圍限制，並新增兩條 Example SQL（通路業績比較、1P/3P 客單價）

---

## Step 12｜再次問答驗證改善效果

> **操作角色**：商品經理、行銷人員、資料分析師

### 12-1 重新測試特殊問題 1

回到 Databricks One 的 Genie Space，再次輸入：

```
電視購物和網路購物的業績比較
```

> ✅ **預期結果**：這次 Genie 能正確識別通路代碼，回傳「電視購物」和「網路購物」的業績比較，並顯示友善的名稱。

### 12-2 重新測試特殊問題 2

再次輸入：

```
各通路的取消訂單比例是多少？
```

> ✅ **預期結果**：Genie 不再嘗試回答，而是明確說明此資料集不含取消訂單記錄，建議使用者透過其他方式取得退貨/取消資訊。

### 12-3 驗證改善前後的差異

| 問題 | 優化前（Step 10） | 優化後（Step 12） | 修復方式 |
|------|------------------|------------------|----------|
| 電視購物和網路購物的業績比較 | Genie 用 `web_part_code` 猜測通路，邏輯錯誤 | 正確使用 `media_gb IN ('01','07')` 篩選，顯示中文名稱 | 補 Instructions + Example SQL |
| 各通路的取消訂單比例 | Genie 嘗試用現有欄位推算，產生幻覺式答案 | 明確告知資料不含取消記錄，不亂猜 | 補 Instructions（資料範圍限制） |

---

## Step 13｜用 Genie Code 自助建立報表

> **操作角色**：商品經理、行銷人員、資料分析師

非技術人員也可以透過 Genie Code 自行建立報表，不需要寫 SQL。

### 13-1 建立新的 Dashboard

1. 點選左上角 Databricks Logo 旁的 **App Switcher** 點選 **Lakehouse**
2. 點選左側 **Dashboards**
2. 點選右上角 **Create Dashboard**，建立全新的空白 Dashboard
3. 切換到 **Canvas** 分頁
4. 點選右上角 **Genie Code**

### 13-2 各角色嘗試自建 Widget

各角色依照自己的需求，用自然語言描述想要的圖表。提示中請指定使用 `momo_workshop.default` 的資料表，讓 Genie Code 知道去哪裡取資料。

**商品經理**
```
使用 momo_workshop.default 的資料，建立一個長條圖，顯示各商品二級分類在 1P 和 3P 的業績佔比比較。
X 軸為二級分類名稱，Y 軸為總營收，以堆疊長條圖呈現，顏色區分 1P（藍色）和 3P（橘色）。
只顯示一級分類中業績前 3 名的分類底下的二級分類。
```

**行銷人員**
```
使用 momo_workshop.default 的資料，建立一個折線圖，同時顯示月度總營收（左 Y 軸）和月度訂單數（右 Y 軸）的趨勢。
X 軸為訂單月份，使用雙 Y 軸讓兩個指標都能清楚呈現，並標示每月的平均客單價數值。
```

**資料分析師**
```
使用 momo_workshop.default 的資料，建立一個散點圖，X 軸為月份，Y 軸為平均客單價，氣泡大小代表該月訂單數，
顏色區分電視購物（'01'）和網路購物（'07'）兩個通路。
```

### 13-3 發布更新後的 Dashboard

1. 確認新增的 Widget 位置和大小
2. 點選右上角 **Publish** 更新 Dashboard

✅ **完成確認**：非技術人員成功新增自己需要的報表 Widget


