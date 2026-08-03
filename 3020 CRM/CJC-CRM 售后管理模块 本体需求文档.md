# CJC-CRM 售后管理模块 本体需求文档

> 计划：698 中粮个性化开发计划 | 模块：CJC-CRM 客户管理 | 版本：V1.0 | 日期：2026-06-05
> 
> 编写人：王慧丹 | 状态：待确认

---

## 1\. 概述

### 1.1 文档目的

本文档定义 CJC-CRM 售后管理模块的**概念数据模型**（Entity Model），以 UML 类图描述为核心，阐明模块内所有实体及其属性、实体间关系、业务规则。

本文档是 FDS 文档的理论骨架：FDS 描述"做什么"，本体模型描述"有哪些东西及其相互关系"，两者共同构成售后模块完整的业务需求定义。

### 1.2 适用范围

- 产品：CJC-CRM 客户管理
- 模块：售后管理（预测客户 → 售后回访 → 项目进度 → 现场任务 → 整机调试 → 设备维修）
- 终端：电脑端为主，手机端支持部分场景（签字、拍照）
- 使用者：产品经理、开发工程师、测试工程师、DBA

### 1.3 与其他模块的关系

```
MDM（主数据）
    │  客户档案同步
    ▼
CRM售后管理（本体模型主体）
    │
    ├──► PDM（项目管理）：项目进度引用任务节点配置
    │
    └──► 线下（审批流）：客户成功触发线下审批
```

---

## 2\. 实体总览

### 2.1 实体清单

| 序号  | 实体中文名 | 实体英文名 | 类型  | 说明  |
| --- | --- | --- | --- | --- |
| 1   | 客户档案 | Customer | 主实体 | 来自CRM基础模块，本模块引用 |
| 2   | 预测客户 | PredictCustomer | 主实体 | 潜在客户生命周期管理 |
| 3   | 预测客户拜访记录 | PredictCustomerVisit | 子实体 | 附属于预测客户 |
| 4   | 客户售后回访 | CustomerReturnVisit | 主实体 | 现有客户回访 |
| 5   | 回访沟通记录 | CustomerVisitRecord | 子实体 | 附属于售后回访 |
| 6   | 项目进度节点配置 | ProjectProgressNode | 配置实体 | 供PDM引用 |
| 7   | 项目进度 | ProjectProgress | 主实体 | 外采/自制项目进度跟踪 |
| 8   | 外采进度明细 | ProjectProgressExternal | 子实体 | 附属于项目进度（外采） |
| 9   | 自制进度明细 | ProjectProgressInternal | 子实体 | 附属于项目进度（自制） |
| 10  | 现场任务节点配置 | FieldTaskNode | 配置实体 | 供现场服务引用 |
| 11  | 现场任务 | FieldService | 主实体 | 现场安装调试任务 |
| 12  | 现场设备清单 | FieldServiceDevice | 子实体 | 附属于现场任务 |
| 13  | 现场任务明细 | FieldServiceTask | 子实体 | 附属于现场任务 |
| 14  | 整机调试服务 | MachineDebugService | 主实体 | 整机设备调试服务 |
| 15  | 调试任务 | MachineDebugTask | 子实体 | 附属于整机调试 |
| 16  | 设备点检任务 | MachineDebugInspection | 子实体 | 附属于设备点检 |
| 18  | 设备维修服务 | EquipmentRepairService | 主实体 | 设备故障维修服务 |
| 19  | 维修记录 | EquipmentRepairRecord | 子实体 | 附属于设备维修 |
| 20  | 投诉记录 | EquipmentRepairComplaint | 子实体 | 附属于设备维修 |
| 21  | 建议记录 | EquipmentRepairSuggestion | 子实体 | 附属于设备维修 |

### 2.2 实体关系图（概念层）

```
┌─────────────────┐
│   客户档案       │  ← MDM同步来源
│   (Customer)    │
└───────┬─────────┘
        │
        │ ① N:1
        ▼
┌─────────────────┐      1:N      ┌────────────────────────┐
│   预测客户       │──────────────►│  预测客户拜访记录      │
│(PredictCustomer)│              │(PredictCustomerVisit) │
└───────┬─────────┘              └────────────────────────┘
        │
        │ ① 转化
        ▼
┌─────────────────┐      1:N      ┌────────────────────────┐
│   客户售后回访   │──────────────►│  回访沟通记录          │
│(CustomerReturn) │              │(CustomerVisitRecord)  │
└───────┬─────────┘              └────────────────────────┘
        │
        │ ① 可转回
        ▼
┌─────────────────┐
│   （预测客户）   │  ← 闭环
└─────────────────┘

┌─────────────────────┐      引用      ┌────────────────────────┐
│  项目进度节点配置    │────────────────►│    项目进度             │
│(ProjectProgressNode)│   ② 1:1       │(ProjectProgress)      │
└─────────────────────┘   按分类引用    └───────┬────────────────┘
                                                │
                        ┌────────────────────────┼────────────────────────┐
                        │ 1:N                   │ 1:N                     │
                        ▼                        ▼                          │
        ┌───────────────────────┐    ┌───────────────────────┐           │
        │   外采进度明细         │    │    自制进度明细        │           │
        │(ProjectProgressExt.)  │    │(ProjectProgressInt.)  │           │
        └───────────────────────┘    └───────────────────────┘           │
                                                                         
┌─────────────────────┐      引用      ┌────────────────────────┐
│  现场任务节点配置    │────────────────►│    现场任务             │
│  (FieldTaskNode)    │   ② 1:1       │  (FieldService)       │
└─────────────────────┘   按分类引用    └───────┬────────────────┘
                                                │
                        ┌────────────────────────┼────────────────────────┐
                        │ 1:N                   │ 1:N                     │
                        ▼                        ▼                          │
        ┌───────────────────────┐    ┌───────────────────────┐           │
        │   现场设备清单         │    │   现场任务明细         │           │
        │(FieldServiceDevice)  │    │ (FieldServiceTask)   │           │
        └───────────────────────┘    └───────────────────────┘           │
                                                                         
┌─────────────────────┐                       ┌────────────────────────┐
│   整机调试服务       │────────────────────────►│    设备点检             │
│(MachineDebugService)│      1:N              │(MachineDebugInspect)  │
└───────┬─────────────┘                       └───────┬────────────────┘
        │                                              │
        │ 1:N                                          │ 1:N
        ▼                                              ▼
┌─────────────────────┐                     ┌────────────────────────┐
│   调试任务           │                     │    点检明细             │
│(MachineDebugTask)   │                     │(MachineDebugInspectDet)│
└─────────────────────┘                     └────────────────────────┘
                                                                         
┌─────────────────────┐      1:N      ┌────────────────────────┐
│   设备维修服务        │──────────────►│    维修记录             │
│(EquipmentRepairSvc) │              │(EquipmentRepairRecord)│
└───────┬─────────────┘              └────────────────────────┘
        │
        ├──────────────────────► 投诉记录（二选一）
        │   (EquipmentRepairComplaint)
        │
        └──────────────────────► 建议记录（二选一）
            (EquipmentRepairSuggestion)
```

---

## 3\. 实体详细定义

---

### 3.1 客户档案（Customer）

> **来源**：MDM主数据同步，为本模块提供客户基础数据。本模块不负责客户档案的创建和维护。

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 客户编码 | CustomerCode | string(50) | 是   | 主键  |
| 客户名称 | CustomerName | string(200) | 是   | —   |
| 客户地址 | Address | string(500) | 否   | —   |
| 联系人 | ContactPerson | string(100) | 否   | —   |
| 联系电话 | ContactPhone | string(50) | 否   | 脱敏存储 |
| 企业规模 | EnterpriseScale | string(100) | 否   | —   |
| 客户分类 | Category | string(50) | 否   | —   |
| 状态  | Status | string(20) | 是   | 枚举：正常/禁用 |

**关系**：

- 1:N → 预测客户（PredictCustomer）：一个客户档案可对应多个预测客户
- 1:N → 客户售后回访（CustomerReturnVisit）：一个客户档案可有多次回访记录
- 1:N → 整机调试服务（MachineDebugService）：一个客户档案可有多次调试服务
- 1:N → 设备维修服务（EquipmentRepairService）：一个客户档案可有多次维修服务

---

### 3.2 预测客户（PredictCustomer）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 客户编码 | CustomerCode | string(50) | 是   | 主键，格式：YC-YYYYMMDD-序号 |
| 客户名称 | CustomerName | string(200) | 是   | 必填  |
| 客户地址 | Address | string(500) | 否   | —   |
| 潜在客户日期 | ProspectDate | date | 否   | —   |
| 客户人员 | ContactPerson | string(100) | 否   | 联系人姓名 |
| 客户联系方式 | ContactPhone | string(50) | 否   | 脱敏存储（存全号，展示时脱敏） |
| 业务经理 | BusinessManager | string(100) | 否   | 默认当前登录用户对应人员 |
| 企业规模 | EnterpriseScale | string(100) | 否   | —   |
| 企业运营情况 | OperationStatus | string(500) | 否   | —   |
| 生产线开机率 | ProductionLineRate | string(50) | 否   | —   |
| 客户评级 | CustomerRating | string(20) | 否   | —   |
| 状态  | Status | string(20) | 是   | 枚举：开立/已生效/审批中/已成功/已失效 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |
| 修改人 | Modifier | string(100) | 否   | —   |
| 修改时间 | ModifyTime | datetime | 否   | —   |

**状态机**：

```
开立 ──[批量生效]──► 已生效 ──[客户成功]──► 审批中 ──[BIP通过]──► 已成功
                           │
                           └──[批量失效]──► 已失效 ──[重新开立]──► 开立
```

**关系**：

- 1:N → 预测客户拜访记录（PredictCustomerVisit）
- N:1 → 客户档案（Customer）：通过客户名称关联

**业务规则**：

- 数据权限：业务经理仅可见/可操作自己创建的记录
- 业务经理新增时，`业务经理`字段默认当前登录人
- `客户联系方式`存入全号，展示时调用脱敏接口（手机号隐藏中间4位）
- 「客户成功」触发BIP审批流，审批通过后状态变更为「已成功」，并记录转化时间
- 仅「开立」状态允许删除

---

### 3.3 预测客户拜访记录（PredictCustomerVisit）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 拜访编码 | VisitCode | string(50) | 是   | 主键，自动生成 |
| 客户编码 | CustomerCode | string(50) | 是   | 外键 → PredictCustomer |
| 拜访目的 | VisitPurpose | string(500) | 是   | —   |
| 拜访日期 | VisitDate | date | 是   | —   |
| 沟通记录 | CommunicationRecord | text | 否   | 沟通摘要 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |

**关系**：

- N:1 → 预测客户（PredictCustomer）

**业务规则**：

- `客户编码`不可修改
- 拜访日期默认当天
- 新增记录自动关联当前预测客户编码

---

### 3.4 客户售后回访（CustomerReturnVisit）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 回访单号 | ReturnVisitNo | string(50) | 是   | 主键，格式：HF-YYYYMMDD-序号 |
| 客户编码 | CustomerCode | string(50) | 是   | 外键 → Customer（来自CRM客户档案） |
| 客户名称 | CustomerName | string(200) | 是   | —   |
| 客户分类 | CustomerCategory | string(50) | 否   | —   |
| 拜访目的 | VisitPurpose | string(500) | 是   | —   |
| 拜访日期 | VisitDate | date | 是   | —   |
| 客户地址 | Address | string(500) | 否   | —   |
| 客户人员 | ContactPerson | string(100) | 否   | 联系人 |
| 客户评级 | CustomerRating | string(20) | 否   | 一星～五星 |
| 业务经理 | BusinessManager | string(100) | 否   | 默认当前登录用户 |
| 企业运营情况 | OperationStatus | string(500) | 否   | —   |
| 生产线开机率 | ProductionLineRate | string(50) | 否   | —   |
| 客户方联系方式 | ContactPhone | string(50) | 否   | 脱敏存储 |
| 状态  | Status | string(20) | 是   | 枚举：开立/已提交 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |

**状态机**：

```
开立 ──[提交]──► 已提交 ──[收回]──► 开立
```

**关系**：

- 1:N → 回访沟通记录（CustomerVisitRecord）
- N:1 → 客户档案（Customer）

**业务规则**：

- 客户字段从CRM客户档案中选择（支持模糊搜索）
- 业务经理仅可操作自己创建的回访记录
- 「提交」后主表锁定，「收回」后可重新编辑
- 「转潜在客户」时将字段携带至预测客户新增页

---

### 3.5 回访沟通记录（CustomerVisitRecord）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 记录编码 | RecordCode | string(50) | 是   | 主键，自动生成 |
| 回访单号 | ReturnVisitNo | string(50) | 是   | 外键 → CustomerReturnVisit |
| 沟通主要内容 | CommunicationContent | text | 是   | —   |
| 问题反馈及处理 | Feedback | text | 否   | 客户反馈的问题及处理情况 |
| 创建时间 | CreateTime | datetime | 是   | —   |

**关系**：

- N:1 → 客户售后回访（CustomerReturnVisit）

---

### 3.6 项目进度节点配置（ProjectProgressNode）

> **类型**：配置实体。供 CJC-PDM 项目管理模块在创建项目进度记录时引用。

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 节点编码 | NodeCode | string(50) | 是   | 主键，唯一 |
| 节点名称 | NodeName | string(100) | 是   | —   |
| 项目分类 | ProjectCategory | string(20) | 是   | 枚举：外采/自制 |
| 节点序号 | NodeOrder | int | 是   | 同一分类内唯一，决定排序 |
| 状态  | Status | string(20) | 是   | 枚举：生效/失效 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |
| 修改人 | Modifier | string(100) | 否   | —   |
| 修改时间 | ModifyTime | datetime | 否   | —   |

**业务规则**：

- 节点编码不可修改
- 同一项目分类内，`节点序号`不可重复
- 「失效」状态时不可被新建项目进度引用，已有引用不受影响
- 仅管理员可编辑，业务经理仅查看

---

### 3.7 项目进度（ProjectProgress）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 项目进度编码 | ProgressCode | string(50) | 是   | 主键，格式：PJ-YYYYMMDD-序号 |
| 项目编码 | ProjectCode | string(50) | 是   | —   |
| 项目名称 | ProjectName | string(200) | 是   | —   |
| 项目分类 | ProjectCategory | string(20) | 是   | 枚举：外采/自制 |
| 项目经理 | ProjectManager | string(100) | 否   | —   |
| 项目进度描述 | ProgressDescription | text | 否   | 当前进度摘要 |
| 合同总金额 | ContractAmount | decimal(18,2) | 否   | —   |
| 来款金额 | ReceivedAmount | decimal(18,2) | 否   | —   |
| 自动生成比例 | CalculatedRatio | string(20) | 否   | 自动计算：来款金额/合同总金额 |
| 状态  | Status | string(20) | 是   | 枚举：开立/已关闭 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |

**状态机**：

```
开立 ──[关闭]──► 已关闭（终态，不可逆）
```

**关系**：

- 1:N → 外采进度明细（ProjectProgressExternal）：当项目分类为「外采」时使用
- 1:N → 自制进度明细（ProjectProgressInternal）：当项目分类为「自制」时使用
- N:1 → 项目进度节点配置（ProjectProgressNode）：通过「进度节点」字段引用

**业务规则**：

- `自动生成比例` = 来款金额 / 合同总金额，自动计算，百分比展示
- 外采子表引用「项目进度节点配置」中外采类节点
- 自制子表引用「项目进度节点配置」中自制类节点
- 「关闭」为终态，不可逆，全员只读

---

### 3.8 外采进度明细（ProjectProgressExternal）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 记录编码 | RecordCode | string(50) | 是   | 主键，自动生成 |
| 项目进度编码 | ProgressCode | string(50) | 是   | 外键 → ProjectProgress |
| 开始时间 | StartTime | datetime | 否   | —   |
| 进度节点 | ProgressNode | string(100) | 是   | 外键 → ProjectProgressNode（外采类） |
| 结束时间 | EndTime | datetime | 否   | —   |
| 合同金额 | ContractAmount | decimal(18,2) | 否   | 该节点对应合同金额 |
| 付款金额 | PaymentAmount | decimal(18,2) | 否   | 已付款金额 |
| 未付款金额 | UnpaidAmount | decimal(18,2) | 否   | 自动计算：合同金额-付款金额 |
| 付款比例 | PaymentRatio | string(20) | 否   | 自动计算：付款金额/合同金额 |
| 来票金额 | InvoiceAmount | decimal(18,2) | 否   | 已收到发票金额 |
| 采购方式 | ProcurementMethod | string(20) | 否   | 枚举：直接采购/竞价/询比/谈判 |
| 进度描述 | ProgressDescription | text | 否   | —   |
| 异常原因 | ExceptionReason | text | 否   | —   |
| 异常处理 | ExceptionHandling | text | 否   | —   |

**关系**：

- N:1 → 项目进度（ProjectProgress）
- N:1 → 项目进度节点配置（ProjectProgressNode）

**业务规则**：

- `未付款金额` = 合同金额 - 付款金额，自动计算
- `付款比例` = 付款金额 / 合同金额，自动计算，百分比展示
- 仅当 ProjectProgress.ProjectCategory = "外采" 时维护此子表

---

### 3.9 自制进度明细（ProjectProgressInternal）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 记录编码 | RecordCode | string(50) | 是   | 主键，自动生成 |
| 项目进度编码 | ProgressCode | string(50) | 是   | 外键 → ProjectProgress |
| 开始时间 | StartTime | datetime | 否   | —   |
| 进度节点 | ProgressNode | string(100) | 是   | 外键 → ProjectProgressNode（自制类） |
| 进度描述 | ProgressDescription | text | 否   | —   |
| 异常原因 | ExceptionReason | text | 否   | —   |
| 异常处理 | ExceptionHandling | text | 否   | —   |

**关系**：

- N:1 → 项目进度（ProjectProgress）
- N:1 → 项目进度节点配置（ProjectProgressNode）

**业务规则**：

- 仅当 ProjectProgress.ProjectCategory = "自制" 时维护此子表

---

### 3.10 现场任务节点配置（FieldTaskNode）

> **类型**：配置实体。被现场任务管理、整机调试服务、设备维修服务模块引用。

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 节点编码 | NodeCode | string(50) | 是   | 主键，唯一 |
| 节点名称 | NodeName | string(100) | 是   | —   |
| 任务分类 | TaskCategory | string(50) | 是   | 枚举：机电交付项目现场/整机现场/设备维修/客户服务 |
| 节点序号 | NodeOrder | int | 是   | 同一分类内唯一，决定排序 |
| 状态  | Status | string(20) | 是   | 枚举：生效/失效 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |

**业务规则**：

- 同一 `任务分类` 内，`节点序号`不可重复
- 「失效」状态不可被新建现场任务引用，已有引用不受影响
- 4个分类分别对应：机电交付项目现场（14501）、整机现场（14504）、设备维修（14505）、客户服务

---

### 3.11 现场任务（FieldService）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 现场编码 | FieldCode | string(50) | 是   | 主键，格式：SC-YYYYMMDD-序号 |
| 项目  | Project | string(200) | 是   | —   |
| 安装地点 | InstallationAddress | string(500) | 否   | —   |
| 项目负责人 | ProjectLeader | string(100) | 否   | —   |
| 负责人电话 | LeaderPhone | string(50) | 否   | —   |
| 安装工期开始 | InstallationStartDate | date | 否   | —   |
| 安装工期结束 | InstallationEndDate | date | 否   | —   |
| 备注  | Remark | text | 否   | —   |
| 状态  | Status | string(20) | 是   | 枚举：草稿/已生效/已提交/已关闭 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |

**状态机**：

```
草稿 ──[生效]──► 已生效 ──[提交]──► 已提交 ──[关闭]──► 已关闭（终态）
       │
       └──[失效]──► 草稿
```

**关系**：

- 1:N → 现场设备清单（FieldServiceDevice）
- 1:N → 现场任务明细（FieldServiceTask）
- N:1 → 现场任务节点配置（FieldTaskNode）：通过任务明细.任务节点编码引用

**业务规则**：

- 支持「复制新增」：复制主表+设备子表+任务子表，生成新编码
- 「提交」前必须所有任务子表 `是否完成=是`
- 「关闭」为终态，不可逆

---

### 3.12 现场设备清单（FieldServiceDevice）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 设备编码 | DeviceCode | string(50) | 是   | 主键，自动生成 |
| 现场编码 | FieldCode | string(50) | 是   | 外键 → FieldService |
| 物料编码 | MaterialCode | string(50) | 是   | MDM物料档案编码 |
| 物料名称 | MaterialName | string(200) | 否   | —   |
| 规格型号 | Specification | string(200) | 否   | —   |
| 备注  | Remark | text | 否   | —   |

**关系**：

- N:1 → 现场任务（FieldService）

---

### 3.13 现场任务明细（FieldServiceTask）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 任务编码 | TaskCode | string(50) | 是   | 主键，自动生成 |
| 现场编码 | FieldCode | string(50) | 是   | 外键 → FieldService |
| 任务节点编码 | NodeCode | string(50) | 否   | 外键 → FieldTaskNode（机电交付项目现场类） |
| 任务名称 | TaskName | string(200) | 是   | —   |
| 开始时间 | StartTime | datetime | 否   | —   |
| 结束时间 | EndTime | datetime | 否   | —   |
| 是否完成 | IsCompleted | bool | 否   | —   |
| 现场安装描述 | InstallationDescription | text | 否   | —   |
| 异常处理记录 | ExceptionHandling | text | 否   | —   |

**关系**：

- N:1 → 现场任务（FieldService）
- N:1 → 现场任务节点配置（FieldTaskNode）

---

### 3.14 整机调试服务（MachineDebugService）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 服务单号 | ServiceNo | string(50) | 是   | 主键，格式：ZJTS-YYYYMMDD-序号 |
| 客户编码 | CustomerCode | string(50) | 是   | 外键 → Customer |
| 安装地点 | InstallationAddress | string(500) | 否   | —   |
| 产品  | Product | string(200) | 否   | —   |
| 型号  | Model | string(100) | 否   | —   |
| 出厂日期 | ManufacturingDate | date | 否   | —   |
| 售后服务人员 | ServiceStaff | string(500) | 否   | 多人，逗号分隔 |
| 电话  | Phone | string(50) | 否   | —   |
| 现场开始时间 | ServiceStartTime | datetime | 否   | —   |
| 现场结束时间 | ServiceEndTime | datetime | 否   | —   |
| 备注  | Remark | text | 否   | —   |
| 状态  | Status | string(20) | 是   | 枚举：草稿/已生效/已提交/已关闭 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |

**状态机**：

```
开立 ──[审核]──► 待处理 ──[手机端处理任务]──► 处理中/处理完毕 ──[关闭]──► 已关闭（终态）
```

**关系**：

- 1:N → 调试任务（MachineDebugTask）
- 1:N → 设备点检（MachineDebugInspection）
- N:1 → 客户档案（Customer）

**业务规则**：

- 售后服务人员支持多选（从人员档案中选择）
- 「提交」前必须：所有调试任务 `是否完成=是` 且所有点检子表 `客户确认` 均已签字
- 手机端支持客户手写签字确认
- 附件上传：满意度调查拍照，jpg/png，≤10MB/文件

---

### 3.15 调试任务（MachineDebugTask）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 任务编码 | TaskCode | string(50) | 是   | 主键，自动生成 |
| 服务单号 | ServiceNo | string(50) | 是   | 外键 → MachineDebugService |
| 任务节点编码 | NodeCode | string(50) | 否   | 外键 → FieldTaskNode（整机现场类） |
| 产品问题 | ProductIssue | text | 是   | 设备现场发现的问题描述 |
| 现场记录 | OnsiteRecord | text | 否   | 服务人员的处理记录 |
| 开始时间 | StartTime | datetime | 否   | —   |
| 结束时间 | EndTime | datetime | 否   | —   |
| 是否完成 | IsCompleted | bool | 否   | —   |
| 客户签字 | CustomerSignature | string(200) | 否   | 手写签名图片URL |
| 签字时间 | SignatureTime | datetime | 否   | 手机时间戳 |

**关系**：

- N:1 → 整机调试服务（MachineDebugService）
- N:1 → 现场任务节点配置（FieldTaskNode）

---

### 3.16 设备点检（MachineDebugInspection）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 点检编码 | InspectionCode | string(50) | 是   | 主键，自动生成 |
| 服务单号 | ServiceNo | string(50) | 是   | 外键 → MachineDebugService |
| 点检情况 | InspectionStatus | text | 是   | 点检结果描述 |
| 客户签字 | CustomerSignature | string(200) | 否   | 手写签名图片URL |
| 签字时间 | SignatureTime | datetime | 否   | 手机时间戳 |

**关系**：

- 1:N → 点检明细（MachineDebugInspectionDetail）
- N:1 → 整机调试服务（MachineDebugService）

---

### 3.17 点检明细（MachineDebugInspectionDetail）

> **类型**：孙实体（挂载于设备点检子表下）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 明细编码 | DetailCode | string(50) | 是   | 主键，自动生成 |
| 点检编码 | InspectionCode | string(50) | 是   | 外键 → MachineDebugInspection |
| 序号  | Sequence | int | 是   | 自动序号 |
| 设备名称 | EquipmentName | string(100) | 是   | —   |
| 检查项目 | InspectionItem | string(200) | 是   | —   |
| 运行是否正常 | IsNormal | bool | 是   | —   |
| 备注  | Remark | text | 否   | —   |

**标准点检项（24项）**：

| 序号  | 设备名称 | 检查项目 |
| --- | --- | --- |
| 1   | 磨粉机 | 外形损坏检查 |
| 2   | 磨粉机 | 气压检查 |
| 3   | 磨粉机 | 电流检查 |
| 4   | 磨粉机 | 气动喂料检查 |
| 5   | 磨粉机 | 变频喂料检查 |
| 6   | 磨粉机 | 轴承温度检查 |
| 7   | 高方筛 | 外形损坏检查 |
| 8   | 高方筛 | 密封检查 |
| 9   | 高方筛 | 振动检查 |
| 10  | 高方筛 | 轴承温度检查 |
| 11  | 高方筛 | 润滑检查 |
| 12  | 混合机 | 外形损坏检查 |
| 13  | 混合机 | 气压检查 |
| 14  | 混合机 | 电流检查 |
| 15  | 混合机 | 变频喂料检查 |
| 16  | 混合机 | 轴承温度检查 |
| 17  | 振动仓底 | 外形损坏检查 |
| 18  | 振动仓底 | 振动检查 |
| 19  | 振动仓底 | 轴承温度检查 |
| 20  | 卸料器 | 外形损坏检查 |
| 21  | 卸料器 | 密封检查 |
| 22  | 绞龙  | 外形损坏检查 |
| 23  | 绞龙  | 轴承温度检查 |
| 24  | 绞龙  | 润滑检查 |

**关系**：

- N:1 → 设备点检（MachineDebugInspection）

---

### 3.18 设备维修服务（EquipmentRepairService）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 维修单号 | RepairNo | string(50) | 是   | 主键，格式：WX-YYYYMMDD-序号 |
| 客户编码 | CustomerCode | string(50) | 是   | 外键 → Customer |
| 问题投诉描述 | ComplaintDescription | text | 否   | —   |
| 客户地点 | CustomerAddress | string(500) | 否   | —   |
| 产品  | Product | string(200) | 否   | —   |
| 型号  | Model | string(100) | 否   | —   |
| 出厂日期 | ManufacturingDate | date | 否   | —   |
| 客户电话 | CustomerPhone | string(50) | 否   | —   |
| 售后服务人员 | ServiceStaff | string(500) | 否   | 多人，逗号分隔 |
| 售后人员电话 | ServiceStaffPhone | string(50) | 否   | —   |
| 现场日期 | OnsiteDate | date | 否   | —   |
| 备注  | Remark | text | 否   | —   |
| 状态  | Status | string(20) | 是   | 枚举：草稿/已生效/已提交/已确认完成/已关闭 |
| 创建人 | Creator | string(100) | 是   | —   |
| 创建时间 | CreateTime | datetime | 是   | —   |

**状态机**：

```
开立 ──[审核]──► 待处理 ──[手机端处理任务]──► 处理中/处理完毕 ──[关闭]──► 已关闭（终态）     

如果处理完毕发现有投诉，则变更为──待质量处理投诉──[质量处理投诉] ──待销售确认工单--[关闭]──► 已关闭（终态）

如果没有投诉，是客户建议，则变更为待销售确认工单--[关闭]-- 已关闭（终态）
```

**关系**：

- 1:N → 维修记录（EquipmentRepairRecord）
- 0..1 → 投诉记录（EquipmentRepairComplaint）：二选一
- 0..1 → 建议记录（EquipmentRepairSuggestion）：二选一
- N:1 → 客户档案（Customer）

**业务规则**：

- 投诉子表与建议子表为二选一维护（不必同时填写）
- 「提交」前所有维修记录的 `问题是否解决=是`
- 投诉/建议子表完成后才允许「确认完成」
- 「关闭」为终态，不可逆

---

### 3.19 维修记录（EquipmentRepairRecord）

| 属性  | 英文名 | 类型  | 必填  | 说明  |
| --- | --- | --- | --- | --- |
| 记录编码 | RecordCode | string(50) | 是   | 主键，自动生成 |
| 维修单号 | RepairNo | string(50) | 是   | 外键 → EquipmentRepairService |
| 任务  | Task | string(200) | 是   | 维修任务名称 |
| 开始时间 | StartTime | datetime | 否   | —   |
| 结束时间 | EndTime | datetime | 否   | —   |
| 产品问题描述 | ProductIssueDescription | text | 是   | 本次发现的产品问题 |
| 维修记录 | RepairRecord | text | 是   | 维修处理过程和方法 |
| 问题是否解决 | IsResolved | bool | 是   | 服务人员勾选 |
| 客户签字 | CustomerSignature | string(200) | 否   | 手写签名图片URL |
| 签字时间 | SignatureTime | datetime | 否   | 手机时间戳 |

**关系**：

- N:1 → 设备维修服务（EquipmentRepairService）

---

### 3.20 投诉记录（EquipmentRepairComplaint）

> **类型**：可选子实体（与建议记录二选一）

| 属性  | 英文名 | 类型  | 必填  | 说明  | 填写阶段 |
| --- | --- | --- | --- | --- | --- |
| 投诉编码 | ComplaintCode | string(50) | 是   | 主键，自动生成 | —   |
| 维修单号 | RepairNo | string(50) | 是   | 外键 → EquipmentRepairService | —   |
| 投诉问题描述及影响 | ComplaintDescription | text | 是   | —   | 服务人员 |
| 现场处理方式 | OnsiteHandling | text | 是   | —   | 服务人员 |
| 现场人员 | OnsiteStaff | string(100) | 否   | 系统自动 | —   |
| 质量人员 | QualityStaff | string(100) | 否   | —   | 质量部门 |
| 质量分析时间 | QualityAnalysisTime | datetime | 否   | —   | 质量部门 |
| 质量判定责任部门 | ResponsibleDept | string(100) | 是   | —   | 质量部门 |
| 责任人所采取措施 | MeasureTaken | text | 否   | —   | 责任部门 |
| 责任部长 | DeptManager | string(100) | 否   | —   | 责任部门 |
| 处理日期 | HandlingDate | date | 否   | —   | 责任部门 |
| 责任部门处理意见 | DeptOpinion | text | 否   | —   | 责任部门 |
| 责任人 | ResponsiblePerson | string(100) | 否   | —   | 责任部门 |
| 销售部门意见 | SalesOpinion | text | 否   | —   | 销售部门 |
| 销售部门日期 | SalesDate | date | 否   | —   | 销售部门 |

**关系**：

- N:1 → 设备维修服务（EquipmentRepairService）

**业务规则**：

- 服务人员填写前四列 → 质量部门填写分析+判定责任部门 → 责任部门填写处理措施+部长+日期 → 销售部门填写意见

---

### 3.21 建议记录（EquipmentRepairSuggestion）

> **类型**：可选子实体（与投诉记录二选一）

| 属性  | 英文名 | 类型  | 必填  | 说明  | 填写阶段 |
| --- | --- | --- | --- | --- | --- |
| 建议编码 | SuggestionCode | string(50) | 是   | 主键，自动生成 | —   |
| 维修单号 | RepairNo | string(50) | 是   | 外键 → EquipmentRepairService | —   |
| 客户建议描述 | SuggestionDescription | text | 是   | 建议内容 | 服务人员 |
| 质量人员 | QualityStaff | string(100) | 否   | —   | 质量部门 |
| 质量分析时间 | QualityAnalysisTime | datetime | 否   | —   | 质量部门 |
| 质量判定责任部门 | ResponsibleDept | string(100) | 是   | —   | 质量部门 |
| 责任部门处理方案 | HandlingPlan | text | 否   | —   | 责任部门 |
| 部长  | DeptManager | string(100) | 否   | —   | 责任部门 |
| 处理日期 | HandlingDate | date | 否   | —   | 责任部门 |
| 责任部门实施记录 | ImplementationRecord | text | 否   | —   | 责任部门 |
| 售后人员跟踪 | ServiceFollowUp | text | 否   | —   | 售后服务人员 |

**关系**：

- N:1 → 设备维修服务（EquipmentRepairService）

---

## 4\. 实体关系矩阵

| 源实体 | 目标实体 | 关系类型 | 关系说明 |
| --- | --- | --- | --- |
| 客户档案 | 预测客户 | 1:N | 一个客户档案对应多个预测客户（培育阶段可并存） |
| 客户档案 | 客户售后回访 | 1:N | 一个客户档案可有多次回访 |
| 客户档案 | 整机调试服务 | 1:N | 一个客户档案可有多次整机调试 |
| 客户档案 | 设备维修服务 | 1:N | 一个客户档案可有多次维修记录 |
| 预测客户 | 预测客户拜访记录 | 1:N | 一个预测客户可有多次拜访 |
| 客户售后回访 | 回访沟通记录 | 1:N | 一次回访可有多条沟通记录 |
| 项目进度节点配置 | 项目进度 | 1:N（按分类） | 一个节点配置可被多个项目进度引用 |
| 项目进度 | 外采进度明细 | 1:N（条件） | 仅外采类项目进度维护 |
| 项目进度 | 自制进度明细 | 1:N（条件） | 仅自制类项目进度维护 |
| 现场任务节点配置 | 现场任务 | 1:N（按分类） | 一个节点配置可被多个现场任务引用 |
| 现场任务 | 现场设备清单 | 1:N | 一个现场可关联多台设备 |
| 现场任务 | 现场任务明细 | 1:N | 一个现场可有多个任务节点 |
| 整机调试服务 | 调试任务 | 1:N | 一次调试可有多个任务 |
| 整机调试服务 | 设备点检 | 1:N | 一次调试可有多个点检项 |
| 设备点检 | 点检明细 | 1:N | 每项点检含多个检查明细 |
| 设备维修服务 | 维修记录 | 1:N | 一次维修可有多次维修任务 |
| 设备维修服务 | 投诉记录 | 0..1:1 | 与建议记录二选一 |
| 设备维修服务 | 建议记录 | 0..1:1 | 与投诉记录二选一 |

---

## 5\. 跨模块关系

### 5.1 与 MDM（主数据管理）

| 方向  | 实体  | 说明  |
| --- | --- | --- |
| MDM → CRM | 客户档案（Customer） | 同步来源，客户唯一标识 |

### 5.2 线下（审批流）

| 触发点 | 实体  | 说明  |
| --- | --- | --- |
| 预测客户 → 客户成功 | PredictCustomer | 「客户成功」触发线下审批，审批通过后状态变更为「已成功」，同步完成客户转正 |

### 5.3 与 PDM（项目管理）

| 方向  | 实体  | 说明  |
| --- | --- | --- |
| CRM → PDM | ProjectProgressNode | PDM引用CRM维护的节点配置 |
| PDM → CRM | （暂无反向依赖） | —   |

---

## 6\. 通用业务规则

### 6.1 编码规则

| 实体  | 编码格式 | 示例  |
| --- | --- | --- |
| 预测客户 | YC-YYYYMMDD-序号 | YC-20260605-001 |
| 客户售后回访 | HF-YYYYMMDD-序号 | HF-20260605-001 |
| 项目进度 | PJ-YYYYMMDD-序号 | PJ-20260605-001 |
| 现场任务 | SC-YYYYMMDD-序号 | SC-20260605-001 |
| 整机调试服务 | ZJTS-YYYYMMDD-序号 | ZJTS-20260605-001 |
| 设备维修服务 | WX-YYYYMMDD-序号 | WX-20260605-001 |

### 6.2 数据权限规则

| 角色  | 数据可见范围 |
| --- | --- |
| 业务经理 | 仅自己创建的数据 |
| 售后服务人员 | 仅自己名下（服务人员字段）的数据 |
| 售后主管 | 本部门全部数据 |
| 销售主管 | 全部数据（只读） |
| 管理员 | 全部数据（可编辑） |

### 6.3 脱敏规则

- `联系方式`字段：存入全号，调用脱敏服务后展示（手机号隐藏中间4位，如 138\*\*\*\*5678）
- 签字图片：仅服务创建人和当前登录人可见

### 6.4 状态机汇总

| 实体  | 初始状态 | 终态  | 状态值 |
| --- | --- | --- | --- |
| 预测客户 | 开立  | 已成功/已失效 | 开立/已生效/审批中/已成功/已失效 |
| 客户售后回访 | 开立  | 已提交 | 开立/已提交 |
| 项目进度 | 开立  | 已关闭 | 开立/已关闭 |
| 现场任务 | 草稿  | 已关闭 | 草稿/已生效/已提交/已关闭 |
| 整机调试服务 | 草稿  | 已关闭 | 草稿/已生效/已提交/已关闭 |
| 设备维修服务 | 草稿  | 已关闭 | 草稿/已生效/已提交/已确认完成/已关闭 |

---

## 7\. 附录

### 7.1 术语表

| 术语  | 定义  |
| --- | --- |
| 预测客户 | 尚未转正的意向客户，处于培育阶段 |
| 客户成功 | 预测客户通过BIP审批后转为正式客户档案 |
| 外采  | 从外部供应商采购设备或服务的项目 |
| 自制  | 公司内部自行生产的设备或服务项目 |
| 现场任务 | 机电类设备的现场安装调试交付作业 |
| 整机调试 | 对已出货整机设备进行的现场调试服务 |
| 设备点检 | 对设备各部件按标准项目进行运行状态检查 |
| 投诉子表 | 客户正式投诉记录，含责任判定和处理措施 |
| 建议子表 | 客户改进建议记录，含质量分析和建议落实跟踪 |

### 7.2 变更记录

| 版本  | 日期  | 修改人 | 变更内容 |
| --- | --- | --- | --- |
| V1.0 | 2026-06-05 | 王慧丹 | 初稿创建 |