# momo Databricks AI/BI Hands-On Workshop

> **目標對象**：產品經理、行銷人員、資料分析師
> **預計時間**：約 2 小時
> **難易度**：★★☆☆☆（初中級）

---

## 📋 大綱

| 步驟 | 主題 | 預計時間 |
|------|------|----------|
| Step 1 | 建立個人 Schema 並上傳資料 | 10 分鐘 |
| Step 2 | AI 自動生成資料表說明 | 5 分鐘 |
| Step 3 | 建立 Unity Catalog Metric View | 15 分鐘 |
| Step 4 | 建立 AI/BI Dashboard | 15 分鐘 |
| Step 5 | 認識 Genie Space | 5 分鐘 |
| Step 6 | 設定 Genie Space 最佳實踐 | 20 分鐘 |
| Step 7 | 再次問答驗證改善效果 | 5 分鐘 |
| Step 8 | 建立 Evaluation Dataset | 10 分鐘 |
| Step 9 | 查看 Genie Monitor | 5 分鐘 |
| Step 10 | 體驗 Databricks One | 5 分鐘 |
| Step 11 | 課後延伸 | — |

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

## Step 1｜建立個人 Schema 並上傳資料

**⏱ 預計時間：10 分鐘**

### 1-1 建立 Schema

1. 點選左側導覽列 **Catalog**（書本圖示）
2. 展開 **momo_workshop** catalog
3. 點選右上角 **＋ Create schema**
4. Schema 名稱填入：`你的英文名字`（例如：`eric`）
   > ⚠️ 請使用小寫英文，不要有空格或特殊符號
5. 點選 **Create** 確認建立

### 1-2 上傳 CSV 建立 Delta Table

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

✅ **完成確認**：在 Catalog 的你的 schema 下應看到 3 張 table

---

## Step 2｜透過 AI 自動生成 Table Description

**⏱ 預計時間：5 分鐘**

讓 AI 自動生成表格和欄位說明，幫助 Genie 更準確理解資料。

1. 點選左側導覽列 **Catalog**
2. 進入 `momo_workshop` → 你的 schema → 點選 `ft_ord_order_aggr`
3. 在 Overview 頁面點選右上角 **Generate AI comment** 按鈕
4. 等待 AI 生成說明後，點選 **Save** 儲存
5. 重複對 `dt_spc_goods` 和 `cus_customer` 執行相同操作

> 💡 **小提示**：AI 生成的說明可以手動編輯修改，幫助 Genie 對業務術語有更好的理解

✅ **完成確認**：三張表格都有 AI 生成的 Description

---

## Step 3｜在 SQL Editor 建立 UC Metric View

**⏱ 預計時間：10 分鐘**

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

> 💡 **Materialization 說明**：每 4 小時自動預先計算並快取指標結果，讓 Dashboard 和 Genie 查詢時大幅加速。`materialized_views` 定義要快取哪些維度組合 — 此處預先計算「月份 × 分類」和「月份 × 通路」兩種最常查詢的組合。`mode: relaxed` 表示快取更新期間仍可查詢（使用舊快取），不會中斷服務。

### 3-3 驗證 Metric View 是否正常

執行以下查詢確認 Metric View 可以正確回傳資料：

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

## Step 4｜建立 AI/BI Dashboard

**⏱ 預計時間：15 分鐘**

### 4-1 新增 Dashboard

1. 點選左側導覽列 **Dashboards**
2. 點選右上角 **Create dashboard**
3. 命名為：`momo電商分析儀表板_你的名字`

### 4-2 新增資料集（Datasets）

在 Dashboard 編輯頁面：

1. 點選上方 **Data** 分頁
2. 點選 **Add dataset**，選擇 **From table**，加入以下資料集：
   - `momo_workshop.<your_schema>.ecommerce_metrics`（Metric View）
   - `momo_workshop.<your_schema>.ft_ord_order_aggr`
   - `momo_workshop.<your_schema>.dt_spc_goods`
   - `momo_workshop.<your_schema>.cus_customer`

### 4-3 使用 Genie Code 生成 Widget

切換到 **Canvas** 分頁

開啟右上方**Genie Code**, 複製以下Prompt生成 4 個圖表(大約等待5分鐘)：

```
Widget 1：月度總營收趨勢
使用 ecommerce_metrics 資料集，建立折線圖顯示每月總營收的趨勢變化。
X 軸為訂單月份，Y 軸為總營收（元），按時間由舊到新排列，線條顏色為藍色。

Widget 2：1P vs 3P 營收佔比
使用 ecommerce_metrics 資料集，建立圓餅圖比較商品類型（party_type）的總營收佔比。
1P 代表自營商品，3P 代表第三方商品，標籤顯示類型名稱及百分比。

Widget 3：前 10 大商品分類營收排名
使用 ecommerce_metrics 資料集，建立水平長條圖顯示一級分類的總營收排名。
只顯示營收最高的前 10 個分類，依總營收由高到低降序排列，Y 軸為分類名稱，X 軸為總營收。

Widget 4：月度訂單數趨勢
使用 ecommerce_metrics 資料集，建立折線圖顯示每月訂單數的趨勢。
X 軸為訂單月份，Y 軸為訂單數，按時間由舊到新排列。
```

---

### 4-4 完成 Dashboard

1. 調整各圖表位置和大小，排列整齊
2. 點選右上角 **Publish** 發布 Dashboard

✅ **完成確認**：Dashboard 顯示 4 張圖表，資料正常呈現

---

## Step 4-1｜設定排程與嵌入（功能導覽）

**⏱ 預計時間：5 分鐘**

### 排程自動更新

1. 在 Dashboard 旁點擊**View Published**按鈕
2. 右上角點選 **Schedule** 圖示
3. 設定更新頻率（例如：每天 08:00）
4. **Advanced settings**展開後可以更多資訊，例如收件人 Email等，點選 **Save**

### Embed iframe 嵌入

1. 點選右上角 **⋮ 更多選項** → **Embed**
2. 複製 iframe 代碼，可嵌入到公司內部網頁或 Portal

> 💡 此功能讓非 Databricks 用戶也能在公司入口網站看到即時更新的報表

---

## Step 5｜建立 Genie Space（基礎版，先看問題）

**⏱ 預計時間：10 分鐘**

### 5-1 建立 Genie Space

1. 點選左側導覽列 **Genie**
2. 點選 **New Genie space**
3. 在 **Tables** 區域加入以下資料表：
   - `momo_workshop.<your_schema>.ft_ord_order_aggr`
   - `momo_workshop.<your_schema>.dt_spc_goods`
   - `momo_workshop.<your_schema>.cus_customer`
   - `momo_workshop.<your_schema>.ecommerce_metrics`（Metric View）
4. **暫時不填** Instructions，直接點選 **Save**
5. 重新命名：`momo電商問答_你的名字`

### 5-2 嘗試問答（觀察不穩定的結果）

在對話框中，模式選擇**Chat - Don't inspect answers**，接著依序輸入以下問題，觀察 Genie 的回應：

**問題 1（觀察：Genie 不知道通路代碼的意義）**
```
電視購物和網路購物的業績比較
```

**問題 2（觀察：Genie 可能無法正確 JOIN 或不知道指標定義）**
```
最近最熱賣的商品是什麼？
```

**問題 3（觀察：Genie 可能給出模糊或錯誤的分析）**
```
幫我分析客戶購買行為
```

> 📌 **觀察重點**：記下 Genie 回答的問題（SQL 錯誤、結果不合理、不知道業務術語等），接下來的 Step 6 會解決這些問題。

---

## Step 6｜遵照 Best Practice 優化 Genie Space

**⏱ 預計時間：10 分鐘**

參考 [Genie Best Practices 文件](https://docs.databricks.com/aws/en/genie/best-practices)，進行以下設定。

### 6-1 填寫 Instructions（業務背景說明）

在 Genie Space 點選 **Edit**，在 **Instructions** 欄位貼上：

```
你是 momo 購物網路的資料分析助手，協助用戶分析電商訂單、商品和客戶資料。

【資料表說明】
- ft_ord_order_aggr：訂單事實表，每一列代表一筆訂單中的一個商品
  - order_time: 訂單成立時間
  - order_no: 訂單編號
  - media_gb: 通路代碼（'01'=電視購物, '07'=網路購物）
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

### 6-2 設定 Join Relationships

在 **Relationships** 區域點選 **Add relationship**，依序新增兩條關聯：

**關聯 1：訂單 → 商品**

| 欄位 | 設定值 |
|------|--------|
| Left table | `ft_ord_order_aggr` |
| Left column | `goods_code` |
| Relationship type | **Many to one** （多筆訂單對應一個商品） |
| Right table | `dt_spc_goods` |
| Right column | `goods_code` |

**關聯 2：訂單 → 客戶**

| 欄位 | 設定值 |
|------|--------|
| Left table | `ft_ord_order_aggr` |
| Left column | `cust_no` |
| Relationship type | **Many to one** （多筆訂單對應一位客戶） |
| Right table | `cus_customer` |
| Right column | `cust_no` |

> 💡 **Relationship type 說明**：選 **Many to one** 代表左側（訂單）的多筆資料可以對應右側（商品/客戶）的同一筆，這是事實表對維度表的標準關係，讓 Genie 在 JOIN 時不會產生重複計算。

### 6-3 新增 Example SQL（提供標準查詢範本）

在 **SQL Queries** 區域，點選 **Add -> Example query**，依序新增以下範例：

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

新增此 Example 時，點選 **Parameter** 確認兩個參數類型：

| Parameter name | Type | Default value |
|----------------|------|---------------|
| `year` | Integer | `2025` |
| `top_n` | Integer | `10` |

> 💡 **展示技巧**：儲存後在對話中輸入「**2024 年前 5 大品牌銷售額**」，Genie 會自動將 `:year` 填入 `2024`、`:top_n` 填入 `5`，讓業務用戶不需要改 SQL 就能自由調整條件。

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

**範例 4：性別消費分析**
- 問題：`男女客戶的消費差異？`
```sql
SELECT
  CASE c.sex
    WHEN '1' THEN '男性'
    WHEN '2' THEN '女性'
    ELSE '未知'
  END AS gender,
  SUM(o.order_amt) AS total_revenue,
  COUNT(DISTINCT o.order_no) AS total_orders,
  COUNT(DISTINCT o.cust_no) AS total_customers,
  ROUND(SUM(o.order_amt) / COUNT(DISTINCT o.order_no), 0) AS avg_order_value
FROM ft_ord_order_aggr o
JOIN cus_customer c ON o.cust_no = c.cust_no
GROUP BY 1
ORDER BY 2 DESC
```

**範例 5：月度指標摘要（引用 Metric View）**
- 問題：`每月的總營收、訂單數和平均客單價？`
```sql
SELECT
  `訂單月份`,
  MEASURE(`總營收`) AS total_revenue,
  MEASURE(`訂單數`) AS total_orders,
  MEASURE(`平均客單價`) AS avg_order_value
FROM momo_workshop.<your_schema>.ecommerce_metrics
GROUP BY ALL
ORDER BY `訂單月份`
```

**範例 6：分類 × 通路交叉分析（引用 Metric View）**
- 問題：`各商品分類在 1P 和 3P 的業績各是多少？`
```sql
SELECT
  `一級分類`,
  `商品類型`,
  MEASURE(`總營收`) AS total_revenue,
  MEASURE(`訂單數`) AS total_orders,
  MEASURE(`客戶數`) AS total_customers
FROM momo_workshop.<your_schema>.ecommerce_metrics
GROUP BY ALL
ORDER BY total_revenue DESC
```

> 💡 **Metric View 引用說明**：Genie 可以直接查詢 Metric View，使用 `MEASURE()` 函數取得預定義的指標，不需要手動寫聚合 SQL。Genie 碰到需要跨分類比較的複合問題時，會優先使用 Metric View 作為資料來源。

點選 **Save** 儲存所有設定。

---

## Step 7｜再次問答驗證改善效果

**⏱ 預計時間：5 分鐘**

重新回到對話視窗，使用以下具體問題測試：

```
2025 年每月的總營收是多少？
```

```
前 5 大商品一級分類的訂單數排名？
```

```
1P 和 3P 的平均客單價各是多少？
```

```
電視購物和網路購物的業績比較
```

```
2024 年前 5 大品牌銷售額
```

> 💡 最後一題會觸發範例 2 的參數查詢。

> ✅ **預期結果**：Genie 能夠生成正確的 SQL、資料有意義、JOIN 正確、對話框中參數欄位正常帶入

### 7-1 使用 Agent Model 進行深度分析

Genie 支援切換為 **Agent** 模式，讓 AI 自動規劃並串接多步驟查詢，適合需要跨維度比較、趨勢解讀或原因推論的複雜分析問題。

**操作方式：**

在對話輸入框左側找到模型切換按鈕，將模式從 **Chat** 切換為 **Agent**。

**輸入以下問題：**

```
請分析 2024 年與 2025 年的營收趨勢變化。哪些商品一級分類成長最快？哪些下滑最多？電視購物和網路購物各自的表現如何？請給出你的觀察與可能原因。
```

**Agent 模式行為說明：**

| 步驟 | Agent 自動執行的動作 |
|------|----------------------|
| 1 | 查詢 2024、2025 各月份總營收（比較基準） |
| 2 | 依一級分類拆解兩年度營收，計算成長率 |
| 3 | 依通路（media_gb）分別統計兩年業績 |
| 4 | 彙整數據並生成文字分析與結論 |

> 💡 **Agent vs Chat 模式差異**
> - **Chat 模式**：一問一查，每個問題產生一段 SQL 並回傳結果表格
> - **Agent 模式**：自動拆解複雜問題為多步驟查詢，並整合結果生成自然語言分析報告，適合探索性分析與業務洞察

> ✅ **預期結果**：Genie Agent 自動執行多次查詢後，輸出包含數字佐證的趨勢分析文字，無需使用者手動拆題

---

## Step 8｜建立 Evaluation Dataset

**⏱ 預計時間：10 分鐘**

Evaluation Dataset 讓你用標準問答組來評估 Genie 的準確度。

### 8-1 新增 Evaluation Questions

在 Genie Space 頁面，點選 **Benchmark** 分頁 → **Add benchmark**

依序新增以下問題和對應的標準 SQL。

> ⚠️ **注意**：Expected SQL 中的 `<your_schema>` 請替換成你自己的 schema 名稱，否則執行時會找不到資料表。

---

**評估問題 1**
- 問題（Question）：`2025 年全年的總營收是多少？`
- 預期 SQL（Expected SQL）：
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

**評估問題 4**
- 問題：`2025 年哪個月的訂單數最高？`
- 預期 SQL：
```sql
SELECT
  DATE_TRUNC('MONTH', order_time) AS order_month,
  COUNT(DISTINCT order_no) AS total_orders
FROM momo_workshop.<your_schema>.ft_ord_order_aggr
WHERE YEAR(order_time) = 2025
GROUP BY 1
ORDER BY 2 DESC
LIMIT 1
```

---

## Step 9｜執行 Evaluation

**⏱ 預計時間：5 分鐘**

1. 在 Benchmark 分頁確認問題清單無誤
2. 點選 **Run all benchmarks**
3. 等待執行完成（約 1-2 分鐘）

### 9-1 查看評估結果

每道題執行完後會產生以下資訊：

**Assessment（整體評分）**
- ✅ **Good**：Genie 生成的 SQL 執行結果與 Expected SQL 一致
- ❌ **Bad**：結果不符或 SQL 執行失敗

**Score Reason（評分說明）**
點選任一題可展開詳細說明，包含：
- Genie 實際生成的 SQL
- Expected SQL 的執行結果對照
- AI 判斷差異的理由（例如：Missing columns、Empty Result）

**Update Ground Truth（更新標準答案）**
若 Genie 生成的 SQL 比原本的 Expected SQL 更好、或你想修正標準答案：
1. 點選該題的 **Edit** 按鈕
2. 將 Expected SQL 更新為更正確的版本
3. 點選 **Save** 後可重新執行該題驗證

> 💡 **討論點**：Fail 的題目是很好的教學素材 — 帶大家看 Genie 生成的 SQL 哪裡出錯，再回到 Step 6 的 Instructions 或 Example SQL 補充對應的描述，形成「問題 → 改善 → 驗證」的循環。

---

## Step 10｜使用對話反饋並查看 Monitor

**⏱ 預計時間：5 分鐘**

### 10-1 留下對話反饋

1. 回到 Genie 對話視窗
2. 對任一回答點選 **👍** 或 **👎**
3. 點選 **Request review**，填寫comment並提交

### 10-2 查看 Monitor 統計

1. 在 Genie Space 頁面點選 **Monitor** 分頁
2. 可以看到：
   - 歷史問答記錄
   - 用戶反饋統計（好評/差評比例）
   - 哪些問題最常被問
   - 哪些問題 Genie 無法回答（需要改善的地方）

> 💡 **實際應用**：Monitor 是持續改善 Genie 準確度的重要工具，建議每週定期檢視

---

## Step 11｜Databricks One 導覽

**⏱ 預計時間：10 分鐘**

Databricks One 是整合 Dashboard 和 Genie 的入口，適合提供給非技術的業務同仁使用。

### 11-1 切換到 Databricks One

1. 點選左上角 Databricks Logo 旁的 **App Switcher**（方格圖示）
2. 選擇 **Databricks One**（或由講師提供連結）

### 11-2 Dashboard 導覽

1. 找到剛才建立的 `momo電商分析儀表板`
2. 確認圖表正常顯示

### 11-3 Genie 問答導覽

1. 在 Databricks One 找到你的 `momo電商問答` Genie Space
2. 體驗在簡潔介面中直接用自然語言問資料問題
3. 嘗試問：
   ```
   今年到目前為止，哪個商品分類最暢銷？
   ```


---

## ❓ 常見問題

**Q：Metric View 查詢很慢？**
A：正常，第一次查詢需要全表掃描。後續可設定 Materialization 或 Z-order 加速。

**Q：Genie 一直說看不懂我的問題？**
A：回到 Instructions，補充更多業務術語的定義；或在 Example SQL 新增相關範例。

**Q：Dashboard 圖表沒有資料？**
A：確認 Dataset 連結正確，以及 Metric View 可以正常查詢（先在 SQL Editor 測試）。

---
