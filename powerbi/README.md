# WYMY - MES IQC Report — TMDL 語意模型

`WYMY - MES IQC Report.SemanticModel/` 是這份報表的語意模型定義，以 **TMDL** 純文字撰寫，
儲存模式為 **Direct Lake on OneLake**，範圍聚焦 **流程 04：RM Inspection（IQC）**。

## 檔案結構

```
WYMY - MES IQC Report.SemanticModel/
├── .platform                     # Fabric 項目描述（type: SemanticModel）
├── definition.pbism              # 語意模型專案設定
└── definition/
    ├── database.tmdl             # compatibilityLevel 1604（Direct Lake 最低要求）
    ├── model.tmdl                # 模型層設定 + ref table 清單
    ├── expressions.tmdl          # DatabaseQuery — OneLake 連線（← 要換 GUID）
    ├── relationships.tmdl        # 12 條關聯
    └── tables/
        ├── Date.tmdl                        dim_date
        ├── Material.tmdl                    dim_material
        ├── Vendor.tmdl                      dim_vendor
        ├── Batch.tmdl                       dim_batch
        ├── Plant.tmdl                       dim_plant
        ├── UD Code.tmdl                     dim_ud_code
        ├── Inspection Characteristic.tmdl   dim_inspection_characteristic
        ├── Defect Code.tmdl                 dim_defect_code
        ├── Inspection Lot.tmdl              fact_inspection_lot   ← 含全部 30 個 measure
        ├── Inspection Result.tmdl           fact_inspection_result
        └── Inspection Defect.tmdl           fact_inspection_defect
```

## 星狀結構

`Inspection Lot` 是模型的樞紐（header fact，一列一張檢驗批次）。維度往 `Inspection Lot` 過濾，
`Inspection Lot` 再往兩張明細 fact 過濾，全部單向多對一，沒有雙向關聯：

```
Date ─┐
Material ─┤
Vendor ─┤                          ┌─→ Inspection Result ──→ Inspection Characteristic
Batch ─┼──→  Inspection Lot  ──────┤
Plant ─┤                           └─→ Inspection Defect  ──→ Inspection Characteristic
UD Code ─┘                                                └─→ Defect Code
```

| 表 | 粒度 |
|---|---|
| `Inspection Lot` | 一列 = 一張 inspection lot（ERP QM lot） |
| `Inspection Result` | 一列 = lot × 檢驗特性（MIC）× 樣本 |
| `Inspection Defect` | 一列 = lot × 缺陷代碼 |

日期為 **role-playing**：`GRDateKey → Date[DateKey]` 是 active（收貨日），
`UDDateKey → Date[DateKey]` 設 `isActive: false`，由 `USERELATIONSHIP` 的 measure 啟用（判定日）。

## Measure 清單（都放在 `Inspection Lot`，共 30 個）

| displayFolder | Measure |
|---|---|
| 01 Volume | Inspection Lots / Lots Pending UD / Lots Passed / Lots Failed / Lots Decided / Inspected Qty / Accepted Qty / Rejected Qty / Scrapped Qty / Returned Qty |
| 02 Quality | UD Pass Rate % / Lot Reject Rate % / Qty Reject Rate % / DPPM / Defect Qty / Critical Defect Qty / Lots With Defects / First Pass Yield % / Characteristic Reject Rate % |
| 03 Cycle Time | UD SLA Days / Avg Inspection Lead Time (Days) / Avg UD Lead Time (Days) / Avg Q-Stock Aging (Days) / Lots Overdue UD / UD On-Time Rate % |
| 04 Time Intelligence | Inspection Lots by UD Date / UD Pass Rate % by UD Date / Inspection Lots YTD / UD Pass Rate % PM / UD Pass Rate % MoM |

`UD SLA Days` 是常數 measure（預設 3 天），`Lots Overdue UD` 與 `UD On-Time Rate %` 都引用它，
改 SLA 只要改這一個地方。

## 部署前必做的兩件事

### 1. 換掉 `expressions.tmdl` 裡的 GUID

```
AzureStorage.DataLake("https://onelake.dfs.fabric.microsoft.com/{workspaceId}/{lakehouseId}")
```

目前是佔位值：workspace `00000000-...-000000000000`、lakehouse `11111111-...-111111111111`。
CI/CD 請用字串取代（不要改成 M 參數 —— 改成參數後 Power BI Desktop / Web 的
**Edit tables** 與 refresh 會失效）。

### 2. 確認 Lakehouse schema 設定

每個 partition 都寫了 `schemaName: dbo`。**如果 Lakehouse 沒有啟用 schema，
要把所有 `schemaName:` 行刪掉**，否則模型載入會失敗：

```bash
grep -rl 'schemaName' "WYMY - MES IQC Report.SemanticModel/definition/tables" \
  | xargs sed -i '/^\t\t\tschemaName: dbo$/d'
```

## 接到現有的報表

這個 repo 只放語意模型。把 `WYMY - MES IQC Report.SemanticModel/` 放到既有 `.pbip` 旁邊，
再讓報表指向它 —— `WYMY - MES IQC Report.Report/definition.pbir`：

```json
{
  "version": "4.0",
  "datasetReference": {
    "byPath": { "path": "../WYMY - MES IQC Report.SemanticModel" }
  }
}
```

> Direct Lake 模型在 Power BI Desktop 是 **live edit（remote modeling）**：Desktop 必須連到
> Fabric 工作區裡的模型才能編輯。本機 PBIP 用來存放定義並走 Git 部署，改動要透過
> Fabric Git Integration 或 REST `updateDefinition` 上去。

## Direct Lake 的模型限制（本模型已據此設計）

- **不能有 DAX 計算資料表** → `Date` 必須是 Lakehouse 裡真實的 `dim_date` Delta 表，不能用 `CALENDAR()`。
- **不能有 DAX 計算資料行**（Direct Lake on OneLake 僅 user-context preview）→ 這些欄位請在
  Gold layer 算好：`InspectionLeadTimeDays`、`UDLeadTimeDays`、`QStockAgingDays`、
  以及 `IsPass` / `IsFail` / `IsPendingUD` / `IsUDCompleted` / `IsRejected` / `IsWithinSpec` 等旗標。
- **不能有模型分割區** → 分割請做在 Delta 表層級。
- `discourageImplicitMeasures: true`，所有數字都走顯式 measure；代理鍵一律 `isHidden: true`。

## 尚未包含（下一步可加）

- **RLS**：多廠區（US1 SLC / US2 Rojas）建議加 Plant 層級的 row-level security。
  Direct Lake 下強烈建議搭配 fixed identity 雲端連線。
- **流程 03（RM Inbound）**：Inbound Delivery / GR / 數量差異事實表，可串出「收貨 → 檢驗 → 放行」全鏈。
- **Gap 追蹤**：GAP-IQC-01～04 的人工替代方案追蹤主檔。
- **Field parameters / calculation groups**：Direct Lake 模型可以加，但會建立隱含計算資料表，需在 Desktop live edit 下操作。
