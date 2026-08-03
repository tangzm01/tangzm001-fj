# 客户全貌（包含详细参考SQL版）

# 一、功能概述

### 1.1 业务目标

为销售、财务、管理层提供**客户360度全景视图**，实现：

- 快速识别客户信用风险与健康状况
- 实时监控订单交付进度与物流状态
- 智能预警提醒，降低坏账风险
- 支持数据穿透，从汇总到明细逐层钻取

### 1.2 用户角色

| 角色  | 核心关注点 | 使用场景 |
| --- | --- | --- |
| 销售员 | 客户信用、订单交付、回款进度 | 日常客户跟进 |
| 销售经理 | 团队客户健康度、预警分布 | 周会/月度复盘 |
| 财务人员 | 应收账款、账龄分布、超期预警 | 对账、催收 |
| 管理层 | 客户价值排名、风险趋势 | 决策分析 |

### 1.3 功能模块

![](files/019facac-616f-7598-85c6-f0b605ec680d/image.png)![](files/019facac-616f-7598-85c6-f4f0cf7bde67/image.png)![](files/019facac-6170-706e-9191-fddc5185dbfa/image.png)

```
客户全貌
├── 客户列表首页
│   ├── 筛选栏（客户名称/等级/行业/销售员/风险状态）
│   ├── 汇总卡片（订单/出货/开票/收款/物流）
│   └── 客户列表表格
│
└── 客户详情页（系统级大弹窗）
    ├── Tab 1: 信用分析
    │   ├── 信用额度卡片（点击穿透）
    │   ├── 信用明细（在执行订单/应收账款/账期）
    │   ├── 账龄分布（柱状图可穿透）
    │   └── 信用变动记录
    │
    ├── Tab 2: 健康度
    │   ├── 综合评分环
    │   ├── 四维度评分（付款能力/合作价值/稳定性/满意度）
    │   ├── 风险等级指示器
    │   └── 健康模型维护（配置权重）
    │
    ├── Tab 3: 预警分析
    │   ├── 财务风险预警
    │   ├── 交付风险预警
    │   ├── 合同风险预警
    │   └── 服务风险预警
    │
    ├── Tab 4: 交付监控
    │   ├── 在执行订单甘特图
    │   └── 交付表现统计
    │
    └── Tab 5: 客户列表（子Tab）
        ├── 订单情况
        ├── 出货情况
        ├── 开票情况
        ├── 收款情况
        └── 物流跟踪
```

---

## 二、数据模型与统计SQL

### 2.1 用友U9核心表结构

| 表名  | 说明  | 关键字段 |
| --- | --- | --- |
| `Cust_Customer` | 客户档案 | CustCode, CustName, CustLevel, Industry, SalesMan |
| `SM_SO` | 销售订单主表 | DocNo, CustCode, DocDate, TotalMoney, Status, PromiseDate |
| `SM_SOLine` | 销售订单行表 | ID, SOID, ItemCode, Qty, Price, Amount |
| `SM_Ship` | 出库单主表 | DocNo, CustCode, ShipDate, TotalMoney |
| AR_ARBillHead | 应收账款表 | DocNo, CustCode, Amount, ARDate, DueDate, Status |
| AR_RecBillHead | 收款单主表 | DocNo, CustCode, ReceiptDate, Amount |
| AR_ARInvoiceLine | 发票主表 | DocNo, CustCode, InvoiceDate, TotalAmount |
| CBO_BankAccount | 客户银行账户 | CustCode, CreditLimit, CreditUsed |
| `CM_Contract` | 合同主表 | ContractNo, CustCode, StartDate, EndDate, Status |
| `CS_ServiceOrder` | 服务工单表 | DocNo, CustCode, Satisfaction, Status |

---

### 2.2 客户列表首页 SQL

#### 2.2.1 客户列表查询（带筛选条件）

```sql
-- ========================================
-- 客户列表查询（汇总统计）
-- 数据源：Cust_Customer + SM_SOMaster + AR_Receivable + BA_BankAccount
-- ========================================
SELECT 
    c.CustCode                    AS 客户编码,
    c.CustName                    AS 客户名称,
    c.CustLevel                   AS 客户等级,
    c.Industry                    AS 所属行业,
    u.UserName                    AS 销售员,
    
    -- 本年订单统计
    ISNULL(so.OrderCount, 0)      AS 本年订单数,
    ISNULL(so.OrderAmount, 0)     AS 订单金额,
    
    -- 信用额度
    ISNULL(ba.CreditLimit, 0)     AS 信用额度,
    ISNULL(ba.CreditUsed, 0)      AS 已占用额度,
    CASE 
        WHEN ba.CreditLimit > 0 
        THEN CAST(ba.CreditUsed * 100.0 / ba.CreditLimit AS DECIMAL(5,2))
        ELSE 0 
    END                           AS 占用率,
    
    -- 应收账款
    ISNULL(ar.ARAmount, 0)        AS 应收账款,
    ISNULL(ar.OverdueAmount, 0)   AS 超期金额,
    ISNULL(ar.OverdueDays, 0)     AS 超期天数,
    
    -- 风险等级计算
    CASE 
        WHEN ar.OverdueDays > 30 OR ar.OverdueAmount > 500000 THEN '重度'
        WHEN ar.OverdueDays > 15 OR ar.OverdueAmount > 200000 THEN '中度'
        WHEN ar.OverdueDays > 0 OR ar.OverdueAmount > 0 THEN '轻度'
        ELSE '无'
    END                           AS 风险等级,
    
    -- 预警标识
    CASE 
        WHEN ar.OverdueDays > 15 
             OR ba.CreditUsed * 100.0 / ba.CreditLimit > 80
             OR EXISTS (
                 SELECT 1 FROM CM_Contract ct 
                 WHERE ct.CustCode = c.CustCode 
                   AND ct.EndDate BETWEEN GETDATE() AND DATEADD(day, 30, GETDATE())
             )
        THEN 1 
        ELSE 0 
    END                           AS 是否预警

FROM Cust_Customer c

-- 关联销售员
LEFT JOIN Base_User u ON c.SalesMan = u.UserCode

-- 本年订单汇总
LEFT JOIN (
    SELECT 
        CustCode,
        COUNT(*) AS OrderCount,
        SUM(TotalMoney) AS OrderAmount
    FROM SM_SOMaster
    WHERE YEAR(DocDate) = YEAR(GETDATE())
      AND Status NOT IN ('关闭', '作废')
    GROUP BY CustCode
) so ON c.CustCode = so.CustCode

-- 应收账款汇总（含账龄）
LEFT JOIN (
    SELECT 
        CustCode,
        SUM(Amount) AS ARAmount,
        SUM(CASE WHEN DueDate < GETDATE() THEN Amount ELSE 0 END) AS OverdueAmount,
        MAX(CASE WHEN DueDate < GETDATE() 
                 THEN DATEDIFF(day, DueDate, GETDATE()) 
                 ELSE 0 END) AS OverdueDays
    FROM AR_Receivable
    WHERE Status NOT IN ('已核销', '作废')
    GROUP BY CustCode
) ar ON c.CustCode = ar.CustCode

-- 信用额度
LEFT JOIN BA_BankAccount ba ON c.CustCode = ba.CustCode

WHERE c.Status = '有效'

-- 筛选条件（动态拼接）
-- AND c.CustName LIKE '%{客户名称}%'
-- AND c.CustLevel = '{客户等级}'
-- AND c.Industry = '{行业}'
-- AND u.UserName = '{销售员}'
-- AND (风险等级条件)

ORDER BY 
    CASE WHEN ar.OverdueDays > 15 THEN 0 ELSE 1 END,  -- 预警客户优先
    so.OrderAmount DESC
```

#### 2.2.2 汇总卡片统计

```sql
-- ========================================
-- 首页汇总卡片数据
-- ========================================

-- 卡片1：本年订单总数
SELECT COUNT(*) AS 订单总数
FROM SM_SOMaster
WHERE YEAR(DocDate) = YEAR(GETDATE())
  AND Status NOT IN ('关闭', '作废');

-- 卡片2：本年出货金额
SELECT ISNULL(SUM(TotalMoney), 0) AS 出货金额
FROM SM_ShipMaster
WHERE YEAR(ShipDate) = YEAR(GETDATE())
  AND Status NOT IN ('作废');

-- 卡片3：本年开票金额
SELECT ISNULL(SUM(TotalAmount), 0) AS 开票金额
FROM AR_InvoiceMaster
WHERE YEAR(InvoiceDate) = YEAR(GETDATE())
  AND Status NOT IN ('作废');

-- 卡片4：本年收款金额
SELECT ISNULL(SUM(Amount), 0) AS 收款金额
FROM AR_ReceiptHead
WHERE YEAR(ReceiptDate) = YEAR(GETDATE())
  AND Status NOT IN ('作废');

-- 卡片5：在途物流单数
SELECT COUNT(DISTINCT sm.DocNo) AS 在途物流单
FROM SM_ShipMaster sm
LEFT JOIN SM_ShipLine sl ON sm.ID = sl.ShipID
WHERE sm.Status = '已出库'
  AND sl.LogisticsStatus NOT IN ('已签收', '已拒收')
  AND sm.ShipDate >= DATEADD(month, -1, GETDATE());
```

---

### 2.3 信用分析模块 SQL

#### 2.3.1 信用额度占用明细

```sql
-- ========================================
-- 信用额度占用明细（点击"200万"展开）
-- 数据源：SM_SOMaster + AR_Receivable
-- ========================================
SELECT 
    so.DocNo                    AS 单据编号,
    '销售订单'                   AS 单据类型,
    so.TotalMoney               AS 占用金额,
    so.DocDate                  AS 占用时间,
    so.PromiseDate              AS 预计释放日期,
    CASE 
        WHEN so.Status = '已关闭' THEN '↓ -' + CAST(so.TotalMoney AS VARCHAR)
        ELSE '↑ +' + CAST(so.TotalMoney AS VARCHAR)
    END                         AS 变动,
    so.Status                   AS 状态,
    
    -- 计算剩余天数
    DATEDIFF(day, GETDATE(), so.PromiseDate) AS 剩余天数

FROM SM_SOMaster so
WHERE so.CustCode = @CustCode
  AND so.Status IN ('已审核', '执行中', '已关闭')
  AND so.DocDate >= DATEADD(year, -1, GETDATE())

UNION ALL

SELECT 
    ar.DocNo                    AS 单据编号,
    '应收账款'                   AS 单据类型,
    ar.Amount                   AS 占用金额,
    ar.ARDate                   AS 占用时间,
    ar.DueDate                  AS 预计释放日期,
    '↑ +' + CAST(ar.Amount AS VARCHAR) AS 变动,
    CASE 
        WHEN ar.DueDate < GETDATE() THEN '已超期'
        ELSE '未到期'
    END                         AS 状态,
    DATEDIFF(day, GETDATE(), ar.DueDate) AS 剩余天数

FROM AR_Receivable ar
WHERE ar.CustCode = @CustCode
  AND ar.Status NOT IN ('已核销', '作废')

ORDER BY 占用时间 DESC;
```

#### 2.3.2 账龄分布统计

```sql
-- ========================================
-- 账龄分布统计（按区间汇总）
-- 数据源：AR_Receivable
-- ========================================
SELECT 
    CustCode,
    
    -- 账龄区间
    SUM(CASE WHEN DATEDIFF(day, ARDate, GETDATE()) BETWEEN 0 AND 30 
        THEN Amount ELSE 0 END) AS [0-30天],
    
    SUM(CASE WHEN DATEDIFF(day, ARDate, GETDATE()) BETWEEN 31 AND 60 
        THEN Amount ELSE 0 END) AS [31-60天],
    
    SUM(CASE WHEN DATEDIFF(day, ARDate, GETDATE()) BETWEEN 61 AND 90 
        THEN Amount ELSE 0 END) AS [61-90天],
    
    SUM(CASE WHEN DATEDIFF(day, ARDate, GETDATE()) > 90 
        THEN Amount ELSE 0 END) AS [90天以上],
    
    SUM(Amount)                 AS 合计

FROM AR_Receivable
WHERE CustCode = @CustCode
  AND Status NOT IN ('已核销', '作废')
GROUP BY CustCode;
```

#### 2.3.3 账龄明细穿透（点击柱状图）

```sql
-- ========================================
-- 账龄明细穿透（如点击"31-60天"柱状图）
-- ========================================
SELECT 
    ar.DocNo            AS 单据编号,
    ar.ARDate           AS 应收日期,
    ar.DueDate           AS 到期日期,
    ar.Amount           AS 金额,
    DATEDIFF(day, ar.ARDate, GETDATE()) AS 账龄天数,
    so.DocNo            AS 关联订单,
    ar.Status           AS 状态,
    DATEDIFF(day, ar.DueDate, GETDATE()) AS 超期天数

FROM AR_Receivable ar
LEFT JOIN SM_SOMaster so ON ar.SOID = so.ID
WHERE ar.CustCode = @CustCode
  AND ar.Status NOT IN ('已核销', '作废')
  AND DATEDIFF(day, ar.ARDate, GETDATE()) BETWEEN 31 AND 60  -- 动态条件
ORDER BY ar.ARDate DESC;
```

#### 2.3.4 信用变动记录

```sql
-- ========================================
-- 信用额度变动历史
-- 数据源：BA_CreditLog（需扩展）
-- ========================================
SELECT 
    ChangeDate          AS 变动日期,
    ChangeType          AS 变动类型,  -- 额度上调/订单占用/回款释放
    ChangeAmount        AS 变动金额,
    AfterLimit          AS 变动后额度,
    Remark              AS 备注,
    CreateUser          AS 操作人
FROM BA_CreditLog
WHERE CustCode = @CustCode
ORDER BY ChangeDate DESC;
```

---

### 2.4 健康度模块 SQL

#### 2.4.1 健康度评分计算

```sql
-- ========================================
-- 客户健康度评分（四维度加权计算）
-- ========================================
WITH PaymentScore AS (
    -- 付款能力评分：超期0天=100，每超7天扣10分
    SELECT 
        CustCode,
        CASE 
            WHEN MAX(OverdueDays) = 0 THEN 100
            ELSE MAX(100 - FLOOR(MAX(OverdueDays) / 7) * 10, 0)
        END AS Score
    FROM (
        SELECT 
            CustCode,
            DATEDIFF(day, DueDate, GETDATE()) AS OverdueDays
        FROM AR_Receivable
        WHERE Status NOT IN ('已核销', '作废')
    ) t
    GROUP BY CustCode
),

ValueScore AS (
    -- 合作价值评分：年交易额排名Top10%=100，每降10%减10分
    SELECT 
        CustCode,
        CASE 
            WHEN Percentile <= 10 THEN 100
            WHEN Percentile <= 20 THEN 90
            WHEN Percentile <= 30 THEN 80
            WHEN Percentile <= 40 THEN 70
            WHEN Percentile <= 50 THEN 60
            WHEN Percentile <= 60 THEN 50
            WHEN Percentile <= 70 THEN 40
            WHEN Percentile <= 80 THEN 30
            WHEN Percentile <= 90 THEN 20
            ELSE 10
        END AS Score
    FROM (
        SELECT 
            CustCode,
            PERCENT_RANK() OVER (ORDER BY SUM(TotalMoney) DESC) * 100 AS Percentile
        FROM SM_SOMaster
        WHERE YEAR(DocDate) = YEAR(GETDATE())
        GROUP BY CustCode
    ) t
),

StabilityScore AS (
    -- 合作稳定性评分：合作年数 >= 4年=100，每少1年扣15分
    SELECT 
        c.CustCode,
        CASE 
            WHEN DATEDIFF(year, MIN(so.DocDate), GETDATE()) >= 4 THEN 100
            ELSE MAX(100 - (4 - DATEDIFF(year, MIN(so.DocDate), GETDATE())) * 15, 0)
        END AS Score
    FROM Cust_Customer c
    LEFT JOIN SM_SOMaster so ON c.CustCode = so.CustCode
    GROUP BY c.CustCode
),

SatisfactionScore AS (
    -- 服务满意度评分：满意度>=90=100，每降10分扣20分
    SELECT 
        CustCode,
        CASE 
            WHEN AVG(Satisfaction) >= 90 THEN 100
            WHEN AVG(Satisfaction) >= 80 THEN 80
            WHEN AVG(Satisfaction) >= 70 THEN 60
            WHEN AVG(Satisfaction) >= 60 THEN 40
            ELSE 20
        END AS Score
    FROM CS_ServiceOrder
    WHERE Status = '已完成'
    GROUP BY CustCode
)

-- 最终评分（权重可在前端配置）
SELECT 
    c.CustCode,
    c.CustName,
    
    ISNULL(ps.Score, 70)    AS 付款能力评分,
    ISNULL(vs.Score, 70)   AS 合作价值评分,
    ISNULL(ss.Score, 70)   AS 合作稳定性评分,
    ISNULL(sfs.Score, 70)  AS 服务满意度评分,
    
    -- 综合评分 = Σ(维度得分 × 权重)
    ISNULL(ps.Score, 70) * 0.30 +
    ISNULL(vs.Score, 70) * 0.25 +
    ISNULL(ss.Score, 70) * 0.25 +
    ISNULL(sfs.Score, 70) * 0.20 AS 综合评分,
    
    -- 等级判定
    CASE 
        WHEN ISNULL(ps.Score, 70) * 0.30 + ISNULL(vs.Score, 70) * 0.25 + 
             ISNULL(ss.Score, 70) * 0.25 + ISNULL(sfs.Score, 70) * 0.20 >= 85 THEN 'A'
        WHEN ISNULL(ps.Score, 70) * 0.30 + ISNULL(vs.Score, 70) * 0.25 + 
             ISNULL(ss.Score, 70) * 0.25 + ISNULL(sfs.Score, 70) * 0.20 >= 70 THEN 'B'
        WHEN ISNULL(ps.Score, 70) * 0.30 + ISNULL(vs.Score, 70) * 0.25 + 
             ISNULL(ss.Score, 70) * 0.25 + ISNULL(sfs.Score, 70) * 0.20 >= 55 THEN 'C'
        ELSE 'D'
    END                     AS 健康等级

FROM Cust_Customer c
LEFT JOIN PaymentScore ps ON c.CustCode = ps.CustCode
LEFT JOIN ValueScore vs ON c.CustCode = vs.CustCode
LEFT JOIN StabilityScore ss ON c.CustCode = ss.CustCode
LEFT JOIN SatisfactionScore sfs ON c.CustCode = sfs.CustCode
WHERE c.CustCode = @CustCode;
```

---

### 2.5 预警分析模块 SQL

#### 2.5.1 综合预警查询

```sql
-- ========================================
-- 客户预警综合查询（四类预警）
-- ========================================

-- 财务风险预警
SELECT 
    '财务风险'               AS 预警类别,
    '应收账款超期'           AS 预警类型,
    ar.DocNo                AS 关联单据,
    ar.Amount               AS 涉及金额,
    DATEDIFF(day, ar.DueDate, GETDATE()) AS 超期天数,
    '应收账款'               AS 数据来源,
    '需尽快催收'             AS 建议措施
FROM AR_Receivable ar
WHERE ar.CustCode = @CustCode
  AND ar.Status NOT IN ('已核销', '作废')
  AND ar.DueDate < GETDATE()

UNION ALL

SELECT 
    '财务风险',
    '信用额度占用率过高',
    '',
    (SELECT CreditUsed FROM BA_BankAccount WHERE CustCode = @CustCode),
    0,
    '信用额度',
    '建议控制新订单'
WHERE EXISTS (
    SELECT 1 FROM BA_BankAccount 
    WHERE CustCode = @CustCode 
      AND CreditUsed * 100.0 / CreditLimit > 70
)

UNION ALL

-- 交付风险预警
SELECT 
    '交付风险'               AS 预警类别,
    '订单交期临近'           AS 预警类型,
    so.DocNo                AS 关联单据,
    so.TotalMoney           AS 涉及金额,
    DATEDIFF(day, GETDATE(), so.PromiseDate) AS 剩余天数,
    '销售订单'               AS 数据来源,
    '查看生产进度'           AS 建议措施
FROM SM_SOMaster so
WHERE so.CustCode = @CustCode
  AND so.Status IN ('已审核', '执行中')
  AND so.PromiseDate BETWEEN GETDATE() AND DATEADD(day, 15, GETDATE())

UNION ALL

-- 合同风险预警
SELECT 
    '合同风险'               AS 预警类别,
    '合同即将到期'           AS 预警类型,
    ct.ContractNo            AS 关联单据,
    ct.ContractAmount        AS 涉及金额,
    DATEDIFF(day, GETDATE(), ct.EndDate) AS 剩余天数,
    '合同管理'               AS 数据来源,
    '安排续约谈判'           AS 建议措施
FROM CM_Contract ct
WHERE ct.CustCode = @CustCode
  AND ct.Status = '执行中'
  AND ct.EndDate BETWEEN GETDATE() AND DATEADD(day, 30, GETDATE())

ORDER BY 
    CASE 预警类别 WHEN '财务风险' THEN 1 WHEN '交付风险' THEN 2 WHEN '合同风险' THEN 3 ELSE 4 END,
    超期天数 DESC;
```

---

### 2.6 交付监控模块 SQL

#### 2.6.1 在执行订单甘特数据

```sql
-- ========================================
-- 在执行订单交付进度（甘特图数据）
-- ========================================
SELECT 
    so.DocNo                AS 订单编号,
    so.DocDate              AS 下单日期,
    so.PromiseDate          AS 承诺交期,
    DATEDIFF(day, so.DocDate, so.PromiseDate) AS 总周期,
    
    -- 生产进度
    ISNULL(po.ProgressPercent, 0) AS 生产进度,
    
    -- 发货进度
    ISNULL(sm.ShipPercent, 0) AS 发货进度,
    
    -- 当前时间位置（百分比）
    DATEDIFF(day, so.DocDate, GETDATE()) * 100.0 / 
        NULLIF(DATEDIFF(day, so.DocDate, so.PromiseDate), 0) AS 当前位置,
    
    -- 风险等级
    CASE 
        WHEN DATEDIFF(day, GETDATE(), so.PromiseDate) <= 7 THEN 'danger'
        WHEN DATEDIFF(day, GETDATE(), so.PromiseDate) <= 15 THEN 'warning'
        ELSE 'normal'
    END                     AS 交付风险等级,
    
    so.TotalMoney           AS 订单金额

FROM SM_SOMaster so

-- 生产进度
LEFT JOIN (
    SELECT 
        so.ID,
        SUM(mo.CompletedQty * 100.0 / NULLIF(mo.PlanQty, 0)) / COUNT(*) AS ProgressPercent
    FROM SM_SOMaster so
    LEFT JOIN SM_SOLine sol ON so.ID = sol.SOID
    LEFT JOIN MO_MOHead mo ON sol.ID = mo.SOLineID
    WHERE so.Status IN ('已审核', '执行中')
    GROUP BY so.ID
) po ON so.ID = po.ID

-- 发货进度
LEFT JOIN (
    SELECT 
        so.ID,
        SUM(sl.ShipQty * 100.0 / NULLIF(sol.Qty, 0)) / COUNT(DISTINCT sol.ID) AS ShipPercent
    FROM SM_SOMaster so
    LEFT JOIN SM_SOLine sol ON so.ID = sol.SOID
    LEFT JOIN SM_ShipLine sl ON sol.ID = sl.SOLineID
    WHERE so.Status IN ('已审核', '执行中')
    GROUP BY so.ID
) sm ON so.ID = sm.ID

WHERE so.CustCode = @CustCode
  AND so.Status IN ('已审核', '执行中')
ORDER BY so.PromiseDate ASC;
```

---

### 2.7 客户列表详情Tab SQL

#### 2.7.1 订单情况查询

```sql
-- ========================================
-- 客户订单明细列表
-- ========================================
SELECT 
    so.DocNo                AS 订单编号,
    i.ItemName              AS 产品名称,
    sol.Qty                 AS 数量,
    sol.Amount              AS 订单金额,
    so.DocDate              AS 签订日期,
    so.PromiseDate          AS 交货日期,
    
    -- 执行进度
    ISNULL(mo.CompletedQty, 0) * 100.0 / NULLIF(sol.Qty, 0) AS 执行进度,
    
    so.Status               AS 状态,
    
    -- 关联出库
    (SELECT TOP 1 DocNo FROM SM_ShipMaster WHERE SOID = so.ID ORDER BY ShipDate DESC) AS 最近出库单

FROM SM_SOMaster so
LEFT JOIN SM_SOLine sol ON so.ID = sol.SOID
LEFT JOIN Base_Item i ON sol.ItemCode = i.ItemCode
LEFT JOIN (
    SELECT SOLineID, SUM(CompletedQty) AS CompletedQty
    FROM MO_MOHead
    GROUP BY SOLineID
) mo ON sol.ID = mo.SOLineID

WHERE so.CustCode = @CustCode
ORDER BY so.DocDate DESC;
```

#### 2.7.2 出货情况查询

```sql
-- ========================================
-- 客户出库明细列表
-- ========================================
SELECT 
    sm.DocNo                AS 出库单号,
    i.ItemName              AS 产品名称,
    sl.Qty                  AS 数量,
    sl.Amount               AS 出库金额,
    sm.ShipDate             AS 出库日期,
    so.DocNo                AS 关联订单,
    sm.Status               AS 状态,
    
    -- 物流信息
    sl.LogisticsCompany     AS 物流公司,
    sl.TrackingNo           AS 运单号

FROM SM_ShipMaster sm
LEFT JOIN SM_ShipLine sl ON sm.ID = sl.ShipID
LEFT JOIN Base_Item i ON sl.ItemCode = i.ItemCode
LEFT JOIN SM_SOLine sol ON sl.SOLineID = sol.ID
LEFT JOIN SM_SOMaster so ON sol.SOID = so.ID

WHERE sm.CustCode = @CustCode
ORDER BY sm.ShipDate DESC;
```

#### 2.7.3 开票情况查询

```sql
-- ========================================
-- 客户发票明细列表
-- ========================================
SELECT 
    im.DocNo                AS 发票号,
    im.InvoiceType          AS 发票类型,
    im.TotalAmount          AS 价税合计,
    im.InvoiceDate          AS 开票日期,
    so.DocNo                AS 关联订单,
    im.Status               AS 状态,
    
    -- 核销情况
    CASE WHEN EXISTS (
        SELECT 1 FROM AR_ReceiptDetail rd 
        WHERE rd.InvoiceID = im.ID
    ) THEN '已核销' ELSE '未核销' END AS 核销状态

FROM AR_InvoiceMaster im
LEFT JOIN SM_SOMaster so ON im.SOID = so.ID
WHERE im.CustCode = @CustCode
ORDER BY im.InvoiceDate DESC;
```

#### 2.7.4 收款情况查询

```sql
-- ========================================
-- 客户收款明细列表
-- ========================================
SELECT 
    rh.DocNo                AS 收款单号,
    rd.Amount               AS 收款金额,
    rh.ReceiptDate          AS 收款日期,
    rh.PaymentMethod        AS 收款方式,
    im.DocNo                AS 核销发票,
    rh.Status               AS 状态,
    
    -- 关联银行流水
    rh.BankAccount          AS 收款账户

FROM AR_ReceiptHead rh
LEFT JOIN AR_ReceiptDetail rd ON rh.ID = rd.ReceiptID
LEFT JOIN AR_InvoiceMaster im ON rd.InvoiceID = im.ID
WHERE rh.CustCode = @CustCode
ORDER BY rh.ReceiptDate DESC;
```

#### 2.7.5 物流跟踪查询

```sql
-- ========================================
-- 客户物流跟踪列表
-- ========================================
SELECT 
    sm.DocNo                AS 出库单号,
    i.ItemName              AS 产品,
    sl.LogisticsCompany     AS 物流公司,
    sl.TrackingNo           AS 运单号,
    sm.ShipDate             AS 发货时间,
    DATEADD(day, sl.EstimatedDays, sm.ShipDate) AS 预计到达,
    
    sl.LogisticsStatus      AS 状态,
    
    -- 最新轨迹
    (SELECT TOP 1 Location + ' ' + Remark 
     FROM SM_LogisticsTrace 
     WHERE TrackingNo = sl.TrackingNo 
     ORDER BY TraceTime DESC) AS 最新位置

FROM SM_ShipMaster sm
LEFT JOIN SM_ShipLine sl ON sm.ID = sl.ShipID
LEFT JOIN Base_Item i ON sl.ItemCode = i.ItemCode

WHERE sm.CustCode = @CustCode
  AND sl.LogisticsStatus NOT IN ('已签收', '已拒收')
ORDER BY sm.ShipDate DESC;
```

---

## 三、页面交互逻辑说明

### 3.1 客户列表首页交互

#### 3.1.1 筛选栏

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 客户名称 [输入框]  客户等级 [下拉]  行业 [下拉]  销售员 [下拉]  风险 [下拉]  [查询] [重置] │
└─────────────────────────────────────────────────────────────────────────────┘

交互规则：
1. 客户名称：支持模糊搜索（LIKE '%{输入}%'
2. 下拉框：选中后立即触发查询（或点击查询按钮）
3. 重置按钮：清空所有筛选条件，恢复默认状态
4. 查询按钮：触发AJAX请求，刷新表格数据
5. 请求参数：
   - customerName: string
   - customerGrade: A/B/C/全部
   - industry: string
   - salesman: string
   - riskStatus: 预警/超期/无风险/全部
```

#### 3.1.2 汇总卡片（纯展示）

```
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  📄      │ │  🚚      │ │  🧾      │ │  💰      │ │  📍      │
│   156    │ │  4,280万 │ │  3,860万  │ │  3,420万 │ │    12    │
│ 本年订单 │ │ 本年出货  │ │ 本年开票  │ │ 本年收款  │ │ 在途物流 │
│ ↑ +23%   │ │ ↑ +18%   │ │ ↑ +15%   │ │ ↑ +12%   │ │ ↓ -3     │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘

交互规则：
1. 纯展示，不可点击
2. 数据实时从后端获取（页面加载时请求一次）
3. 显示同比变化（与去年同期对比）
```

#### 3.1.3 客户列表表格

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│ 客户编码 │ 客户名称      │ 等级 │ 行业     │ 销售员 │ 订单数 │ 订单金额 │ 信用额度 │ ... │ 操作 │
├──────────┼───────────────┼──────┼─────────┼───────┼───────┼─────────┼─────────┼─────┼──────┤
│ KH-0088  │ 内蒙古XX稀土... │ A级  │ 稀土新材 │ 王经理 │  12   │ 420万   │ 200万   │ ... │  →   │
└──────────┴───────────────┴──────┴─────────┴───────┴───────┴─────────┴─────────┴─────┴──────┘

交互规则：
1. 点击客户名称 → 进入客户详情页（系统级大弹窗）
2. 悬浮行末尾"→"按钮 → 点击同样进入详情页
3. 预警客户行背景色：#FFF9F2（浅橙色）
4. 分页：支持切换每页条数（10/20/50/100）
5. 排序：支持按"订单金额""占用率""应收账款"降序排列
6. 刷新：筛选条件变化后，表格自动刷新
```

---

### 3.2 客户详情页交互

#### 3.2.1 页面切换

```
入口：客户列表 → 点击客户名称
出口：点击"返回"按钮 → 返回列表页

弹窗规则：
1. 详情页以"系统级大弹窗"形式覆盖在列表页上方
2. 弹窗距离屏幕边缘各8px，圆角16px
3. 点击遮罩层或返回按钮均可关闭
4. Tab状态保持：关闭后再打开，恢复默认Tab（信用分析）
```

#### 3.2.2 Tab切换

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ [💳 信用分析] [❤️ 健康度] [🚨 预警分析 (4)] [📦 交付监控] [📊 客户列表]       │
└─────────────────────────────────────────────────────────────────────────────┘

交互规则：
1. 点击Tab切换内容，不刷新页面
2. 带数字徽章的Tab显示预警数量
3. Tab切换时保持页面滚动位置
4. URL参数：?tab=credit|health|alert|delivery|list
```

---

### 3.3 信用分析交互详解

#### 3.3.1 信用额度卡片

```
┌─────────────────────────────────────┐
│ 💳 信用额度         [战略客户]       │
│                                     │
│     [200万]  ← 点击穿透             │
│  📍 点击查看占用明细 →              │
│                                     │
│ 总授信额度                          │
│ ████████████████░░░░ 72%           │
│                                     │
│  144万       56万        72%        │
│  已占用 📍   剩余额度    占用率 📍   │
└─────────────────────────────────────┘

交互规则：
1. 点击"200万"数字 → 展开信用占用明细表格
2. 点击"已占用"数值 → 穿透查看占用订单列表
3. 点击"占用率"百分比 → 穿透查看占用趋势图
4. 进度条颜色：
   - 绿色：占用率 < 60%
   - 橙色：占用率 60% ~ 80%
   - 红色：占用率 > 80%
```

#### 3.3.2 信用占用明细展开区

```
展开动画：slideDown（0.3s ease）

表格列：
| 单据编号 | 单据类型 | 占用金额 | 占用时间 | 预计释放 | 变动 | 状态 |

交互规则：
1. 点击单据编号 → 弹出单据详情（新弹窗）
2. "变动"列：
   - 绿色向下箭头 + "↓ -金额"：已释放
   - 红色向上箭头 + "↑ +金额"：新增占用
3. 状态列：
   - 蓝色标签"执行中"
   - 绿色标签"已完成/已释放"
   - 红色标签"已超期"
```

#### 3.3.3 账龄分布柱状图

```
┌──────────────────────────────────────────────────────────────────┐
│ 📈 账龄分布                📍 穿透查看全部明细 →                  │
│                                                                   │
│    █        █                                                      │
│    █        █        █                                             │
│    █        █        █        █                                    │
│  ──┴──    ──┴──    ──┴──    ──┴──                                │
│  0-30天   31-60天  61-90天  >90天                                  │
│   56万     20万      10万      0                                   │
│    📍       📍        📍                                           │
└──────────────────────────────────────────────────────────────────┘

交互规则：
1. 点击柱状图 → 穿透查看该账龄区间的应收明细
2. 点击"穿透查看全部明细" → 查看完整账龄明细表
3. 柱状图高度：按金额比例计算（最大值100%高度）
4. 柱状图颜色：
   - 绿色：0-30天（正常）
   - 橙色：31-60天（关注）
   - 红色：61-90天（预警）
   - 深红：>90天（严重）
```

---

### 3.4 健康度交互详解

#### 3.4.1 健康度评分环

```
        综合评分
       ┌─────────┐
      ╱           ╲
     │     72      │  ← 动态数字动画（1.5s）
     │     C级     │
      ╲           ╱
       └─────────┘

动画效果：
1. 页面加载时，评分环从0%动画填充到目标百分比
2. 数字从0跳动到目标分数（每20ms增加1）
3. 等级根据分数自动判定：
   - A级：≥85分
   - B级：70-84分
   - C级：55-69分
   - D级：<55分
```

#### 3.4.2 维度评分卡片

```
┌─────────────────────────────────────┐
│ 💰 付款能力                    68分 │
│ ████████████████████░░░░░░░░       │
│ 应收账款超期12天 · 警告            │
│ 📍 查看维度详情 →                   │
└─────────────────────────────────────┘

交互规则：
1. 点击卡片 → 穿透查看该维度的详细计算过程
2. 进度条颜色：
   - 绿色：≥80分
   - 橙色：60-79分
   - 红色：<60分
3. 描述文字根据分数动态生成
```

#### 3.4.3 健康模型维护

```
入口：点击"⚙️ 维护健康模型"按钮
弹窗内容：
1. 当前评分预览
2. 维度权重配置（滑块 + 数字输入）
3. 各维度评分规则说明

保存规则：
1. 权重合计必须等于100%
2. 权重配置保存在用户偏好表（Base_UserPreference）
3. 保存后立即重新计算健康度评分
```

---

### 3.5 预警分析交互详解

#### 3.5.1 预警卡片

```
┌─────────────────────────────────────────────────────────────────┐
│ 💸 财务风险                                     [🔴 2]        │
├─────────────────────────────────────────────────────────────────┤
│ 🚨 应收账款超期                                                 │
│    86万元已超期12天，需尽快催收                                  │
│    订单：SO-2026-0210  │  超期：12天          [立即处理]        │
│                                                                 │
│ ⚠️ 信用额度占用率过高                                           │
│    当前占用率72%，剩余额度56万                                   │
│    已占用：144万  │  建议：控制新订单                            │
└─────────────────────────────────────────────────────────────────┘

交互规则：
1. 有预警时卡片顶部红色边框 + 数字徽章显示数量
2. 点击"立即处理"按钮 → 跳转到对应的业务处理页面
   - 应收超期 → 应收账款催收页面
   - 合同到期 → 合同续约申请页面
   - 交期临近 → 订单交付进度页面
3. 预警数据每5分钟自动刷新一次
```

---

### 3.6 交付监控交互详解

#### 3.6.1 甘特图交付进度

```
┌─────────────────────────────────────────────────────────────────────────┐
│ SO-2026-0318 · 氧化镨钕 20吨                            [🔄 生产中 60%] │
│                                                                         │
│ ████████████████████████████░░░░░░░░░░░░  │               │            │
│ ↑下单 03-18                       ↑今天      ↑交期 05-20             │
│                                                                         │
│ 下单：03-18        预计按时交付              交期：05-20              │
└─────────────────────────────────────────────────────────────────────────┘

交互规则：
1. 蓝色进度条：正常执行中
2. 橙色进度条：交期临近（≤15天）
3. 红色进度条：交期风险（≤7天或已延期）
4. 点击订单号 → 穿透到订单详情页
5. "今天"标记线实时更新位置
```

---

### 3.7 穿透弹窗交互规范

#### 3.7.1 通用穿透弹窗

```
┌───────────────────────────────────────────────────────────────┐
│ 📋 单据详情                                            [×]    │
│ 内蒙古包头市XX稀土有限公司 · 应收账款明细                      │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 信息卡片区（4列）                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 明细表格区                                            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ 时间轴区（可选）                                      │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
└───────────────────────────────────────────────────────────────┘

交互规则：
1. 弹窗宽度：720px（固定），高度：自适应（最大80vh）
2. 点击遮罩层或右上角"×"关闭
3. 支持ESC键关闭
4. 弹窗内表格支持滚动
5. 点击单据编号可继续穿透（嵌套弹窗）
```

---

## 四、判定标准汇总

### 4.1 客户等级判定标准

#### 4.1.1 等级定义

| 等级  | 名称  | 定义  | 授信政策 | 服务优先级 |
| --- | --- | --- | --- | --- |
| A级  | 战略客户 | 年交易额Top 10%或战略意义客户 | 信用额度上浮20% | 最高优先级 |
| B级  | 重点客户 | 年交易额Top 30%或稳定合作客户 | 标准信用额度 | 高优先级 |
| C级  | 普通客户 | 年交易额Top 60%或一般合作客户 | 标准信用额度×80% | 标准优先级 |
| D级  | 观察客户 | 新客户或需观察客户 | 信用额度×50%或现款交易 | 低优先级 |

#### 4.1.2 等级判定规则

```yaml
判定维度:
  - 年度交易额（权重40%）：
      A级: ≥500万
      B级: 200-500万
      C级: 50-200万
      D级: <50万
  
  - 合作年限（权重25%）：
      A级: ≥5年
      B级: 3-5年
      C级: 1-3年
      D级: <1年
  
  - 回款及时率（权重20%）：
      A级: ≥98%
      B级: 95-98%
      C级: 90-95%
      D级: <90%
  
  - 战略价值（权重15%，人工评定）：
      A级: 行业标杆/战略意义
      B级: 区域重点
      C级: 一般客户
      D级: 潜力客户

计算公式:
  等级分数 = Σ(维度得分 × 权重)
  
  等级判定:
    A级: ≥90分
    B级: 75-89分
    C级: 60-74分
    D级: <60分
```

#### 4.1.3 等级SQL判定

```sql
-- ========================================
-- 客户等级自动判定（年度评估）
-- ========================================
WITH CustomerMetrics AS (
    SELECT 
        c.CustCode,
        c.CustName,
        
        -- 年度交易额
        ISNULL(so.AnnualAmount, 0) AS AnnualAmount,
        
        -- 合作年限
        DATEDIFF(year, ISNULL(so.FirstOrderDate, c.CreateDate), GETDATE()) AS CooperationYears,
        
        -- 回款及时率
        CASE 
            WHEN ISNULL(ar.TotalDue, 0) = 0 THEN 100
            ELSE CAST(ISNULL(ar.OnTimePayment, 0) * 100.0 / ar.TotalDue AS DECIMAL(5,2))
        END AS PaymentTimeliness,
        
        -- 战略价值（从客户档案获取）
        ISNULL(c.StrategicValue, 60) AS StrategicValue
        
    FROM Cust_Customer c
    LEFT JOIN (
        SELECT 
            CustCode,
            SUM(TotalMoney) AS AnnualAmount,
            MIN(DocDate) AS FirstOrderDate
        FROM SM_SOMaster
        WHERE YEAR(DocDate) = YEAR(GETDATE())
        GROUP BY CustCode
    ) so ON c.CustCode = so.CustCode
    LEFT JOIN (
        SELECT 
            CustCode,
            COUNT(*) AS TotalDue,
            SUM(CASE WHEN PaymentDate <= DueDate OR PaymentDate IS NULL THEN 1 ELSE 0 END) AS OnTimePayment
        FROM AR_Receivable
        WHERE YEAR(ARDate) = YEAR(GETDATE())
        GROUP BY CustCode
    ) ar ON c.CustCode = ar.CustCode
),

ScoreCalculation AS (
    SELECT 
        CustCode,
        CustName,
        
        -- 年度交易额得分（40分）
        CASE 
            WHEN AnnualAmount >= 5000000 THEN 40
            WHEN AnnualAmount >= 2000000 THEN 32
            WHEN AnnualAmount >= 500000 THEN 24
            ELSE 16
        END AS AmountScore,
        
        -- 合作年限得分（25分）
        CASE 
            WHEN CooperationYears >= 5 THEN 25
            WHEN CooperationYears >= 3 THEN 20
            WHEN CooperationYears >= 1 THEN 15
            ELSE 10
        END AS YearsScore,
        
        -- 回款及时率得分（20分）
        CASE 
            WHEN PaymentTimeliness >= 98 THEN 20
            WHEN PaymentTimeliness >= 95 THEN 16
            WHEN PaymentTimeliness >= 90 THEN 12
            ELSE 8
        END AS PaymentScore,
        
        -- 战略价值得分（15分）
        CASE 
            WHEN StrategicValue = 90 THEN 15  -- 行业标杆
            WHEN StrategicValue = 75 THEN 12  -- 区域重点
            WHEN StrategicValue = 60 THEN 9   -- 一般客户
            ELSE 6                            -- 潜力客户
        END AS StrategicScore
        
    FROM CustomerMetrics
)

SELECT 
    CustCode,
    CustName,
    AmountScore + YearsScore + PaymentScore + StrategicScore AS TotalScore,
    
    -- 等级判定
    CASE 
        WHEN AmountScore + YearsScore + PaymentScore + StrategicScore >= 90 THEN 'A级'
        WHEN AmountScore + YearsScore + PaymentScore + StrategicScore >= 75 THEN 'B级'
        WHEN AmountScore + YearsScore + PaymentScore + StrategicScore >= 60 THEN 'C级'
        ELSE 'D级'
    END AS CustomerLevel,
    
    -- 等级有效期
    DATEADD(year, 1, GETDATE()) AS ValidUntil
    
FROM ScoreCalculation
ORDER BY TotalScore DESC;
```

#### 4.1.4 等级调整规则

```
自动调整触发条件:
  1. 年度评估（每年12月）：根据当年数据重新评定
  2. 季度微调（每季度末）：连续2个季度健康度下降可降级
  3. 即时调整：出现重大风险事件可立即降级

重大风险事件定义:
  - 超期应收 > 90天且金额 > 50万
  - 年度交易额下降 > 50%
  - 法律诉讼记录
  - 经营异常（工商注销/吊销/停业）

调整流程:
  自动降级 → 销售经理确认 → 系统记录 → 通知相关人
```

---

### 4.2 风险等级判定标准

#### 4.2.1 风险等级定义

| 等级  | 名称  | 风险描述 | 应对措施 |
| --- | --- | --- | --- |
| 无风险 | ✓ 正常 | 无超期应收，信用占用率<60% | 正常业务 |
| 轻度  | ⚡ 轻度 | 有少量超期（≤15天）或信用占用60-80% | 加强关注 |
| 中度  | ⚠ 中度 | 超期16-30天或信用占用>80% | 限制授信 |
| 重度  | ⛔ 重度 | 超期>30天或存在重大风险事项 | 停止授信 |

#### 4.2.2 综合风险评分规则

```yaml
评分维度:
  
  1. 应收账款风险（权重40%）：
      - 超期天数评分：
          0天: 100分
          1-15天: 80分
          16-30天: 60分
          31-60天: 40分
          >60天: 20分
      
      - 超期金额评分：
          0元: 100分
          <10万: 80分
          10-50万: 60分
          50-100万: 40分
          >100万: 20分
  
  2. 信用额度风险（权重25%）：
      - 占用率评分：
          <50%: 100分
          50-60%: 85分
          60-70%: 70分
          70-80%: 55分
          80-90%: 40分
          >90%: 25分
  
  3. 合作稳定性风险（权重20%）：
      - 订单频率变化：
          正常: 100分
          减少30%: 70分
          减少50%: 50分
          停止下单: 20分
      
      - 投诉频率：
          无投诉: 100分
          1-2次/年: 80分
          3-5次/年: 60分
          >5次/年: 40分
  
  4. 外部风险（权重15%）：
      - 工商状态：
          正常: 100分
          异常名录: 60分
          停业/注销: 20分
      
      - 法律诉讼：
          无: 100分
          作为原告: 90分
          作为被告（≤3起）: 70分
          作为被告（>3起）: 40分

综合风险评分公式:
  风险分数 = Σ(维度得分 × 权重)
  
  风险等级判定:
    无风险: ≥85分
    轻度: 70-84分
    中度: 55-69分
    重度: <55分
```

#### 4.2.3 风险等级SQL判定

```sql
-- ========================================
-- 客户风险等级综合判定
-- ========================================
WITH RiskMetrics AS (
    SELECT 
        c.CustCode,
        c.CustName,
        
        -- 【维度1】应收账款风险（40%）
        -- 超期天数
        ISNULL(ar.MaxOverdueDays, 0) AS MaxOverdueDays,
        -- 超期金额
        ISNULL(ar.OverdueAmount, 0) AS OverdueAmount,
        
        -- 【维度2】信用额度风险（25%）
        ISNULL(ba.CreditUsed, 0) AS CreditUsed,
        ISNULL(ba.CreditLimit, 1) AS CreditLimit,
        CASE 
            WHEN ba.CreditLimit > 0 THEN ba.CreditUsed * 100.0 / ba.CreditLimit
            ELSE 100
        END AS CreditUsageRate,
        
        -- 【维度3】合作稳定性风险（20%）
        -- 订单频率变化（与去年同期对比）
        CASE 
            WHEN ISNULL(prev.OrderCount, 0) = 0 THEN 50  -- 无对比数据
            WHEN ISNULL(curr.OrderCount, 0) * 100.0 / prev.OrderCount >= 70 THEN 100
            WHEN ISNULL(curr.OrderCount, 0) * 100.0 / prev.OrderCount >= 50 THEN 70
            WHEN ISNULL(curr.OrderCount, 0) * 100.0 / prev.OrderCount > 0 THEN 50
            ELSE 20
        END AS OrderFrequencyScore,
        -- 投诉次数
        ISNULL(comp.ComplaintCount, 0) AS ComplaintCount,
        
        -- 【维度4】外部风险（15%）
        ISNULL(ext.BusinessStatus, 100) AS BusinessStatusScore,
        ISNULL(ext.LawsuitScore, 100) AS LawsuitScore
        
    FROM Cust_Customer c
    LEFT JOIN (
        SELECT 
            CustCode,
            MAX(DATEDIFF(day, DueDate, GETDATE())) AS MaxOverdueDays,
            SUM(CASE WHEN DueDate < GETDATE() THEN Amount ELSE 0 END) AS OverdueAmount
        FROM AR_Receivable
        WHERE Status NOT IN ('已核销', '作废')
        GROUP BY CustCode
    ) ar ON c.CustCode = ar.CustCode
    LEFT JOIN BA_BankAccount ba ON c.CustCode = ba.CustCode
    LEFT JOIN (
        -- 当前季度订单数
        SELECT CustCode, COUNT(*) AS OrderCount
        FROM SM_SOMaster
        WHERE DocDate >= DATEADD(quarter, -1, GETDATE())
        GROUP BY CustCode
    ) curr ON c.CustCode = curr.CustCode
    LEFT JOIN (
        -- 去年同期订单数
        SELECT CustCode, COUNT(*) AS OrderCount
        FROM SM_SOMaster
        WHERE DocDate >= DATEADD(year, -1, DATEADD(quarter, -1, GETDATE()))
          AND DocDate < DATEADD(year, -1, GETDATE())
        GROUP BY CustCode
    ) prev ON c.CustCode = prev.CustCode
    LEFT JOIN (
        -- 投诉次数（服务工单）
        SELECT CustCode, COUNT(*) AS ComplaintCount
        FROM CS_ServiceOrder
        WHERE ServiceType = '投诉'
          AND YEAR(DocDate) = YEAR(GETDATE())
        GROUP BY CustCode
    ) comp ON c.CustCode = comp.CustCode
    LEFT JOIN (
        -- 外部风险数据（需对接第三方API）
        SELECT 
            CustCode,
            CASE BusinessStatus 
                WHEN '正常' THEN 100
                WHEN '异常' THEN 60
                ELSE 20
            END AS BusinessStatus,
            CASE 
                WHEN LawsuitCount = 0 THEN 100
                WHEN LawsuitRole = '原告' THEN 90
                WHEN LawsuitCount <= 3 THEN 70
                ELSE 40
            END AS LawsuitScore
        FROM Ext_CompanyRisk
    ) ext ON c.CustCode = ext.CustCode
),

RiskScoreCalc AS (
    SELECT 
        CustCode,
        CustName,
        
        -- 【维度1】应收账款风险得分（40分）
        (
            CASE 
                WHEN MaxOverdueDays = 0 THEN 100
                WHEN MaxOverdueDays <= 15 THEN 80
                WHEN MaxOverdueDays <= 30 THEN 60
                WHEN MaxOverdueDays <= 60 THEN 40
                ELSE 20
            END * 0.5 +  -- 超期天数权重50%
            CASE 
                WHEN OverdueAmount = 0 THEN 100
                WHEN OverdueAmount < 100000 THEN 80
                WHEN OverdueAmount < 500000 THEN 60
                WHEN OverdueAmount < 1000000 THEN 40
                ELSE 20
            END * 0.5    -- 超期金额权重50%
        ) * 0.40 AS AR_RiskScore,
        
        -- 【维度2】信用额度风险得分（25分）
        CASE 
            WHEN CreditUsageRate < 50 THEN 100
            WHEN CreditUsageRate < 60 THEN 85
            WHEN CreditUsageRate < 70 THEN 70
            WHEN CreditUsageRate < 80 THEN 55
            WHEN CreditUsageRate < 90 THEN 40
            ELSE 25
        END * 0.25 AS Credit_RiskScore,
        
        -- 【维度3】合作稳定性风险得分（20分）
        (
            OrderFrequencyScore * 0.6 +  -- 订单频率权重60%
            CASE 
                WHEN ComplaintCount = 0 THEN 100
                WHEN ComplaintCount <= 2 THEN 80
                WHEN ComplaintCount <= 5 THEN 60
                ELSE 40
            END * 0.4    -- 投诉频率权重40%
        ) * 0.20 AS Stability_RiskScore,
        
        -- 【维度4】外部风险得分（15分）
        (
            BusinessStatusScore * 0.5 +
            LawsuitScore * 0.5
        ) * 0.15 AS External_RiskScore
        
    FROM RiskMetrics
)

SELECT 
    CustCode,
    CustName,
    
    -- 各维度得分
    AR_RiskScore,
    Credit_RiskScore,
    Stability_RiskScore,
    External_RiskScore,
    
    -- 综合风险评分
    AR_RiskScore + Credit_RiskScore + Stability_RiskScore + External_RiskScore AS TotalRiskScore,
    
    -- 风险等级判定
    CASE 
        WHEN AR_RiskScore + Credit_RiskScore + Stability_RiskScore + External_RiskScore >= 85 THEN '无风险'
        WHEN AR_RiskScore + Credit_RiskScore + Stability_RiskScore + External_RiskScore >= 70 THEN '轻度'
        WHEN AR_RiskScore + Credit_RiskScore + Stability_RiskScore + External_RiskScore >= 55 THEN '中度'
        ELSE '重度'
    END AS RiskLevel,
    
    -- 主要风险原因（供前端展示）
    CASE 
        WHEN MaxOverdueDays > 30 THEN '应收账款严重超期'
        WHEN OverdueAmount > 500000 THEN '超期金额较大'
        WHEN CreditUsageRate > 80 THEN '信用额度占用过高'
        WHEN ComplaintCount > 3 THEN '投诉次数较多'
        WHEN BusinessStatusScore < 80 THEN '工商状态异常'
        ELSE '综合风险'
    END AS MainRiskReason
    
FROM RiskScoreCalc
ORDER BY TotalRiskScore ASC;  -- 风险分数低的排前面
```

#### 4.2.4 风险预警响应措施

```
┌─────────────────────────────────────────────────────────────┐
│ 风险等级  │ 响应措施                                          │
├─────────────────────────────────────────────────────────────┤
│ 无风险    │ ✓ 正常开展业务                                    │
│           │ ✓ 年度评估客户等级                                │
├─────────────────────────────────────────────────────────────┤
│ 轻度      │ ⚡ 加强关注，每周检查应收状态                      │
│           │ ⚡ 销售员主动沟通回款计划                          │
│           │ ⚡ 新订单需销售经理审批                            │
├─────────────────────────────────────────────────────────────┤
│ 中度      │ ⚠ 限制信用额度，超额需财务审批                     │
│           │ ⚠ 每周发送对账单，电话催收                        │
│           │ ⚠ 暂停非紧急订单的授信                            │
│           │ ⚠ 评估是否需要调整客户等级                         │
├─────────────────────────────────────────────────────────────┤
│ 重度      │ ⛔ 停止授信，要求现款交易                          │
│           │ ⛔ 启动法律催收程序                               │
│           │ ⛔ 冻结客户账户，停止新业务                        │
│           │ ⛔ 上报管理层，制定处置方案                        │
└─────────────────────────────────────────────────────────────┘
```

---

### 4.3 健康度等级判定标准

#### 4.3.1 健康度等级定义

| 等级  | 分数区间 | 状态描述 | 业务含义 |
| --- | --- | --- | --- |
| A级  | ≥85分 | 🟢 优质客户 | 合作稳定，回款及时，优先服务 |
| B级  | 70-84分 | 🔵 正常客户 | 合作正常，标准服务流程 |
| C级  | 55-69分 | 🟡 需关注客户 | 存在风险隐患，需加强跟进 |
| D级  | <55分 | 🔴 风险客户 | 风险较高，需限制授信 |

#### 4.3.2 健康度与客户等级的关系

```
健康度等级 ≠ 客户等级

区别说明:
┌───────────────────────────────────────────────────────────────────┐
│ 维度         │ 客户等级              │ 健康度等级               │
├───────────────────────────────────────────────────────────────────┤
│ 评定周期     │ 年度评定              │ 实时计算（建议每日）     │
│ 主要依据     │ 历史交易额+战略价值   │ 当前经营状况+付款能力    │
│ 调整频率     │ 相对稳定              │ 动态变化                 │
│ 业务用途     │ 授信政策+服务优先级   │ 风险预警+业务控制        │
└───────────────────────────────────────────────────────────────────┘

关联规则:
  - 客户等级 A级 + 健康度 ≥85 → 维持 A级，优先服务
  - 客户等级 A级 + 健康度 <70 → 触发降级评估流程
  - 客户等级 C级 + 健康度 ≥85 → 触发升级评估流程
  - 连续2个季度健康度 <55 → 自动降级
```

---

### 4.4 账龄区间判定标准

| 区间  | 天数范围 | 颜色标识 | 业务含义 | 处理建议 |
| --- | --- | --- | --- | --- |
| 正常  | 0-30天 | 🟢 绿色 | 账期内，正常状态 | 无需特殊处理 |
| 关注  | 31-60天 | 🟡 黄色 | 接近账期，需关注 | 提醒销售员跟进 |
| 预警  | 61-90天 | 🟠 橙色 | 已超期，需催收 | 发送催款函 |
| 严重  | \>90天 | 🔴 红色 | 长期超期，高风险 | 启动法律程序 |

---

### 4.5 信用占用率预警标准

| 占用率区间 | 预警级别 | 颜色标识 | 响应措施 |
| --- | --- | --- | --- |
| <50% | 无预警 | 🟢 绿色 | 正常业务 |
| 50-60% | 提示  | 🔵 蓝色 | 关注趋势 |
| 60-80% | 黄色预警 | 🟡 黄色 | 新订单需审批 |
| 80-100% | 红色预警 | 🔴 红色 | 限制新订单，催收 |
| \>100% | 紧急预警 | ⛔ 深红 | 停止授信，冻结账户 |

---

### 4.6 交付风险预警标准

| 剩余天数 | 预警级别 | 颜色标识 | 响应措施 |
| --- | --- | --- | --- |
| \>15天 | 无预警 | 🟢 绿色 | 正常跟踪 |
| 8-15天 | 关注  | 🔵 蓝色 | 检查生产进度 |
| 1-7天 | 预警  | 🟠 橙色 | 每日跟进，协调资源 |
| 0天  | 到期  | 🟡 黄色 | 确认交付时间 |
| <0天 | 延期  | 🔴 红色 | 协调客户，启动应急预案 |

---

### 4.7 合同到期预警标准

| 剩余天数 | 预警级别 | 颜色标识 | 响应措施 |
| --- | --- | --- | --- |
| \>30天 | 无预警 | 🟢 绿色 | 正常跟踪 |
| 15-30天 | 关注  | 🔵 蓝色 | 提醒销售员准备续约 |
| 1-14天 | 预警  | 🟠 橙色 | 启动续约流程 |
| 0天  | 到期  | 🟡 黄色 | 确认续约意向 |
| <0天 | 过期  | 🔴 红色 | 暂停相关业务，催办续约 |

---

## 五、功能说明

### 4.1 信用额度管理

#### 4.1.1 业务规则

```
1. 信用额度来源：BA_BankAccount.CreditLimit
2. 额度占用计算：
   - 销售订单审核时，订单金额计入占用
   - 出库单生成应收时，订单占用转为应收占用
   - 收款核销时，应收占用释放

3. 占用率预警阈值：
   - 黄色预警：60% ~ 80%
   - 红色预警：> 80%

4. 超额控制：
   - 可配置是否允许超额下单
   - 超额需审批流程
```

#### 4.1.2 账龄计算规则

```
账龄 = DATEDIFF(day, ARDate, GETDATE())

区间划分：
- 0-30天：正常账期
- 31-60天：关注账期
- 61-90天：预警账期
- >90天：严重超期

超期天数 = MAX(0, DATEDIFF(day, DueDate, GETDATE()))
```

---

### 4.2 健康度评分模型

#### 4.2.1 评分维度权重（可配置）

```
默认权重：
- 付款能力：30%
- 合作价值：25%
- 合作稳定性：25%
- 服务满意度：20%

配置保存：
- 存储表：Base_UserPreference
- Key：HealthModel_Weights
- Value：JSON格式 {"payment":30,"value":25,"stability":25,"satisfaction":20}
```

#### 4.2.2 各维度评分规则

```
【付款能力】（数据源：AR_Receivable）
├─ 满分条件：无超期应收
├─ 扣分规则：每超期7天扣10分
└─ 底分：0分

【合作价值】（数据源：SM_SOMaster）
├─ 满分条件：年交易额行业Top 10%
├─ 扣分规则：每下降10%排名扣10分
└─ 底分：10分

【合作稳定性】（数据源：SM_SOMaster）
├─ 满分条件：合作年限≥4年
├─ 扣分规则：每少1年扣15分
└─ 底分：25分

【服务满意度】（数据源：CS_ServiceOrder）
├─ 满分条件：满意度≥90分
├─ 扣分规则：每下降10分扣20分
└─ 底分：20分
```

#### 4.2.3 综合评分计算

```
综合评分 = Σ(维度得分 × 权重)

等级判定：
- A级：≥85分 → 优质客户，优先服务
- B级：70-84分 → 正常客户，标准服务
- C级：55-69分 → 需关注客户，加强跟进
- D级：<55分 → 风险客户，限制授信
```

---

### 4.3 预警规则配置

#### 4.3.1 财务风险预警

```
┌─────────────────────────────────────────────────────────────┐
│ 预警类型：应收账款超期                                       │
├─────────────────────────────────────────────────────────────┤
│ 触发条件：DueDate < GETDATE()                               │
│ 预警级别：                                                  │
│   - 轻度：超期1-15天                                        │
│   - 中度：超期16-30天                                       │
│   - 重度：超期>30天                                         │
│ 提醒频率：每日                                              │
│ 提醒对象：销售员 + 财务主管                                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 预警类型：信用额度占用过高                                   │
├─────────────────────────────────────────────────────────────┤
│ 触发条件：CreditUsed / CreditLimit > 阈值                   │
│ 预警阈值：                                                  │
│   - 黄色：60%                                               │
│   - 红色：80%                                               │
│ 提醒频率：实时                                              │
│ 提醒对象：销售员 + 信用主管                                 │
└─────────────────────────────────────────────────────────────┘
```

#### 4.3.2 交付风险预警

```
┌─────────────────────────────────────────────────────────────┐
│ 预警类型：订单交期临近                                       │
├─────────────────────────────────────────────────────────────┤
│ 触发条件：PromiseDate - GETDATE() ≤ 提前天数                │
│ 预警级别：                                                  │
│   - 关注：15天内                                            │
│   - 预警：7天内                                             │
│   - 紧急：已延期                                            │
│ 提醒频率：每日                                              │
│ 提醒对象：销售员 + 生产计划员                               │
└─────────────────────────────────────────────────────────────┘
```

#### 4.3.3 合同风险预警

```
┌─────────────────────────────────────────────────────────────┐
│ 预警类型：合同即将到期                                       │
├─────────────────────────────────────────────────────────────┤
│ 触发条件：EndDate - GETDATE() ≤ 提前天数                    │
│ 预警级别：                                                  │
│   - 关注：30天内                                            │
│   - 预警：15天内                                            │
│   - 紧急：已到期                                            │
│ 提醒频率：每周                                              │
│ 提醒对象：销售员 + 合同管理员                               │
└─────────────────────────────────────────────────────────────┘
```

---

### 4.4 数据刷新机制

#### 4.4.1 刷新策略

```
┌─────────────────────────────────────────────────────────────┐
│ 数据类型              │ 刷新频率    │ 刷新方式             │
├─────────────────────────────────────────────────────────────┤
│ 汇总卡片数据          │ 页面加载    │ 主动请求             │
│ 客户列表数据          │ 筛选触发    │ 主动请求             │
│ 预警数据              │ 5分钟       │ 定时轮询             │
│ 健康度评分            │ 30分钟      │ 定时轮询             │
│ 交付进度              │ 1分钟       │ WebSocket推送         │
│ 物流轨迹              │ 5分钟       │ 定时轮询             │
└─────────────────────────────────────────────────────────────┘
```

#### 4.4.2 缓存策略

```
┌─────────────────────────────────────────────────────────────┐
│ 数据类型              │ 缓存时间  │ 缓存位置             │
├─────────────────────────────────────────────────────────────┤
│ 客户基础信息          │ 30分钟    │ Redis                │
│ 信用额度              │ 5分钟     │ Redis                │
│ 健康度评分            │ 30分钟    │ Redis                │
│ 预警数据              │ 实时      │ 不缓存               │
└─────────────────────────────────────────────────────────────┘
```

---

## 五、接口定义

### 5.1 客户列表查询接口

```yaml
接口路径: /api/customer/list
请求方式: GET
请求参数:
  - customerName: string (可选) - 客户名称模糊搜索
  - customerGrade: string (可选) - A/B/C
  - industry: string (可选) - 行业编码
  - salesman: string (可选) - 销售员编码
  - riskStatus: string (可选) - warning/overdue/normal
  - pageNum: int - 页码，默认1
  - pageSize: int - 每页条数，默认10

响应示例:
{
  "code": 200,
  "data": {
    "total": 38,
    "list": [
      {
        "custCode": "KH-2024-0088",
        "custName": "内蒙古包头市XX稀土有限公司",
        "grade": "A级",
        "industry": "稀土新材料",
        "salesman": "王经理",
        "orderCount": 12,
        "orderAmount": 4200000,
        "creditLimit": 2000000,
        "creditUsed": 1440000,
        "creditUsage": 72,
        "arAmount": 860000,
        "arOverdueDays": 12,
        "riskLevel": "中度",
        "hasAlert": true
      }
    ]
  }
}
```

### 5.2 客户详情查询接口

```yaml
接口路径: /api/customer/detail/{custCode}
请求方式: GET
响应示例:
{
  "code": 200,
  "data": {
    "basicInfo": { ... },
    "creditInfo": {
      "limit": 2000000,
      "used": 1440000,
      "available": 560000,
      "usageRate": 72,
      "details": [ ... ]
    },
    "healthInfo": {
      "score": 72,
      "grade": "C",
      "dimensions": { ... }
    },
    "alerts": [ ... ],
    "deliveryInfo": { ... }
  }
}
```

### 5.3 健康度评分接口

```yaml
接口路径: /api/customer/health/score/{custCode}
请求方式: GET
请求参数:
  - weights: json (可选) - 自定义权重配置

响应示例:
{
  "code": 200,
  "data": {
    "totalScore": 72,
    "grade": "C",
    "dimensions": {
      "payment": { "score": 68, "weight": 30, "contribution": 20.4 },
      "value": { "score": 85, "weight": 25, "contribution": 21.25 },
      "stability": { "score": 80, "weight": 25, "contribution": 20 },
      "satisfaction": { "score": 65, "weight": 20, "contribution": 13 }
    }
  }
}
```

### 5.4 预警列表接口

```yaml
接口路径: /api/customer/alerts/{custCode}
请求方式: GET
请求参数:
  - alertType: string (可选) - finance/delivery/contract/service

响应示例:
{
  "code": 200,
  "data": [
    {
      "alertId": "ALT-2026-001",
      "alertType": "财务风险",
      "alertSubType": "应收账款超期",
      "severity": "danger",
      "refDocNo": "AR-2026-0318",
      "amount": 860000,
      "overdueDays": 12,
      "suggestion": "需尽快催收",
      "createTime": "2026-04-20"
    }
  ]
}
```

---

## 六、权限配置

### 6.1 功能权限

```
┌─────────────────────────────────────────────────────────────┐
│ 权限编码                    │ 权限名称        │ 角色分配     │
├─────────────────────────────────────────────────────────────┤
│ customer:list:view          │ 客户列表查看    │ 全员可见     │
│ customer:detail:view        │ 客户详情查看    │ 销售相关     │
│ customer:credit:drill       │ 信用明细穿透    │ 销售+财务    │
│ customer:health:config      │ 健康模型配置    │ 管理员       │
│ customer:export             │ 数据导出        │ 管理员       │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 数据权限

```
┌─────────────────────────────────────────────────────────────┐
│ 数据范围                    │ 可见客户范围    │ 角色         │
├─────────────────────────────────────────────────────────────┤
│ 全部客户                    │ 所有客户        │ 管理层       │
│ 本部门客户                  │ 所属部门客户    │ 部门经理     │
│ 本人客户                    │ 负责的客户      │ 销售员       │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、附录

### 7.1 用友U9表字段映射

```
Cust_Customer:
├─ CustCode → 客户编码
├─ CustName → 客户名称
├─ CustLevel → 客户等级（A/B/C/D）
├─ Industry → 所属行业
├─ SalesMan → 销售员编码
└─ Status → 状态

SM_SOMaster:
├─ DocNo → 单据编号
├─ CustCode → 客户编码
├─ DocDate → 单据日期
├─ TotalMoney → 总金额
├─ Status → 状态（已审核/执行中/已关闭）
└─ PromiseDate → 承诺交期

AR_Receivable:
├─ DocNo → 单据编号
├─ CustCode → 客户编码
├─ Amount → 金额
├─ ARDate → 应收日期
├─ DueDate → 到期日期
└─ Status → 状态
```

### 7.2 扩展需求说明

```
若用友U9标准表字段不足，建议扩展以下字段：

1. BA_BankAccount 扩展：
   - CreditLimit DECIMAL(18,2) — 信用额度
   - CreditUsed DECIMAL(18,2) — 已占用额度
   - CreditUpdateTime DATETIME — 额度更新时间

2. Cust_Customer 扩展：
   - HealthScore INT — 健康度评分
   - HealthGrade CHAR(1) — 健康等级
   - HealthUpdateTime DATETIME — 评分更新时间

3. 新增表 BA_CreditLog：
   - ID BIGINT PRIMARY KEY
   - CustCode VARCHAR(50)
   - ChangeType VARCHAR(20)
   - ChangeAmount DECIMAL(18,2)
   - AfterLimit DECIMAL(18,2)
   - Remark NVARCHAR(200)
   - CreateTime DATETIME
   - CreateUser VARCHAR(50)
```

---

**文档结束**