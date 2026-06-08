---
name: sp-to-query-sql
description: "将 SQL Server 存储过程转换为纯只读接口查询 SQL。适用场景：医院上报系统中含过程化逻辑、DML 操作的存储过程转换为声明式 CTE + SELECT 查询。"
tags: [sql, database, etl, hospital, transformation]
allowed-tools: Read, Write, Bash, Task
output_format: markdown
metadata:
  author: '韩抒豪'
  version: '1.3.0'
  created: '2026-06-05'
  updated: '2026-06-08'
  changes: '新增规则3.0(UPDATE转JOIN风险)、规则4.3(DISTINCT使用规则)，修复LEFT JOIN多行匹配导致数据膨胀问题'
---

# 存储过程 → 接口查询SQL 转换技能

## 技能概述

将 SQL Server 存储过程（SP）转换为纯只读接口查询 SQL。核心原则：
- 剔除过程化、落地逻辑
- 后置清洗全部内联到 SELECT
- 物理临时表优先转 CTE（大数据量允许临时表物化）
- 参数改为占位符
- **直接输出结果集，不落地**

## 触发条件

**关键词**：转查询SQL、存储过程转查询、SP转SQL、接口ETL、医嘱转换、上报接口

**典型场景**：
- "把这个存储过程转成查询SQL"
- "将SP转换为纯SELECT语句"
- "医院上报接口需要转换存储过程"

## 功能特性

| 特性 | 说明 |
|------|------|
| 过程化代码删除 | 移除 CREATE PROC、BEGIN、END、GO、USE 等 |
| 临时表转CTE | `#临时表` → `cte_xxx`，大数据量允许临时表物化 |
| 后置UPDATE内联 | UPDATE 清洗逻辑转为 SELECT 字段表达式 |
| 参数占位符 | `@参数` → `${KSSJ}`、`${JSSJ}` |
| 空值处理 | `COALESCE(NULLIF(RTRIM(LTRIM(...)), ''), 默认值)` |
| 特殊值映射 | `CASE WHEN` 实现值转换 |
| 性能优化 | 物化中间结果、独立派生表、条件聚合 |

## 使用方法

### 参数说明

```
/sp-to-query-sql <存储过程文件路径> [目标SQL文件路径]
```

| 参数 | 必需 | 说明 |
|------|------|------|
| 存储过程文件路径 | ✓ | 原始 SP SQL 文件的完整路径 |
| 目标SQL文件路径 | ✗ | 输出文件路径，默认不落地直接返回 |

### 使用示例

**示例1：基础转换**
```
将 E:\work\sp\usp_yzmx.sql 转换为查询SQL
```

**示例2：指定输出路径**
```
把存储过程 E:\work\sp\原始.sql 转成查询SQL，保存到 E:\work\output\转换后.sql
```

## 工作流

```
┌─────────────────────────────────────────────────────────────┐
│  1. 接收输入 → 2. 分析结构 → 3. 评估规模                       │
├─────────────────────────────────────────────────────────────┤
│  >500行 或 UPDATE>10个 → 使用 task 子代理并行处理              │
│  基础表>10万行 → 建议使用临时表物化                             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  4. 逐规则转换 → 5. 输出检查 → 6. 返回结果                      │
└─────────────────────────────────────────────────────────────┘
```

## 转换规则（10条核心规则）

### 规则0：代码一致性原则（最重要）

**核心原则**：转换时必须**原样保持原始存储过程中的逻辑**，即使原始代码看起来像"bug"或非标准写法，**禁止"修正"原代码逻辑**。

#### 0.1 字段名错误必须保留

原SP中如果使用了错误的字段名（如变量名拼写错误），转换后必须**保持相同的错误**：

```sql
-- ❌ 错误做法：自动"修正"原代码的bug
-- 原SP使用 LEN(a.zyzd_zy)，转换后"修正"为 LEN(a.mzzd_zy)
CASE
    WHEN RIGHT(a.mzzd_zy, 1) = '.'
    THEN LEFT(a.mzzd_zy, LEN(a.mzzd_zy) - 1)  -- 错误！修正了原bug
    ELSE a.mzzd_zy
END

-- ✓ 正确做法：原样保留原代码的逻辑（即使原代码有bug）
-- 原SP使用 LEN(a.zyzd_zy)，转换后也必须用 LEN(a.zyzd_zy)
CASE
    WHEN RIGHT(a.mzzd_zy, 1) = '.'
    THEN LEFT(a.mzzd_zy, LEN(a.zyzd_zy) - 1)  -- 正确！保持原代码逻辑
    ELSE a.mzzd_zy
END
```

#### 0.2 常见原代码"bug"类型

| 原代码bug类型 | 示例 | 转换处理 |
|--------------|------|----------|
| 字段名拼写错误 | `LEN(zyzd_zy)` 应为 `LEN(mzzd_zy)` | **保留错误字段名** |
| 变量名引用错误 | `@jsrq` 应为 `@jssj` | **保留错误变量名**（转为占位符后也保持） |
| 条件逻辑错误 | `WHEN a=1` 应为 `WHEN a='1'` | **保留原条件逻辑** |
| 函数参数错误 | `SUBSTRING(x,1,100)` 截取长度不对 | **保留原参数** |

#### 0.3 例外情况

以下情况可以修正（需明确说明）：
- **纯语法错误**：如缺少括号、引号不匹配等会导致SQL无法执行的错误
- **用户明确要求**：用户明确指出需要修复某个bug

#### 0.4 转换后验证

转换完成后，必须对比原SP中每个表达式与转换后的表达式：
```
原SP表达式 → 转换后表达式 → 是否一致？
LEN(a.zyzd_zy) → LEN(a.zyzd_zy) → ✓ 一致
```

---

### 规则1：删除过程化代码

**删除内容**：
- 数据库上下文：`USE [HOSPITAL_DW]`、`GO`
- 存储过程定义：`CREATE PROC` / `ALTER PROC`、参数列表、`AS`、`BEGIN`、`END`
- 配置语句：`SET ANSI_NULLS ON`、`SET QUOTED_IDENTIFIER ON`
- 变量声明与赋值：`DECLARE @kssj...`、`SET @kssj = ...`
- 条件判断与默认值逻辑：`IF(@ksrq=''...)` → 完全移除，由外部程序提供完整时间范围
- 事务控制：`BEGIN TRANSACTION`、`COMMIT`、`ROLLBACK`

**替换为**：
- 文件头部增加脚本说明注释（接口名称、原始存储过程名、参数说明、转换日期）
- 参数直接使用占位符：`${KSSJ}`、`${JSSJ}`（格式：`yyyy-mm-dd HH:MM:SS`）

### 规则2：临时表处理（允许CTE或临时表，优先CTE）

| 原模式 | 转换后（优先CTE） | 备选（大数据量性能优化） |
|--------|------------------|--------------------------|
| `SELECT ... INTO #tmp FROM ...` | `WITH cte_tmp AS (SELECT ... FROM ...)` | 保持 `SELECT ... INTO #tmp`，但**不能写入最终业务表**，查询结束后需显式 `DROP` |

**判断标准**：
- 基础表行数 < 10万 → 优先使用纯CTE
- 基础表行数 > 10万 或 后续关联复杂 → 使用临时表物化

**临时表物化规范**：
- 临时表仅用于当前查询会话，不持久化
- 查询结束前显式删除临时表
- 在临时表上创建主键或聚集索引（如 `PRIMARY KEY (xh)`）加速连接

```sql
-- 方式1：纯CTE（小数据量）
WITH cte_VW_CQYZK AS (
    SELECT * FROM VW_CQYZK WITH(NOLOCK)
    WHERE lrrq BETWEEN ${KSSJ} AND ${JSSJ}
)
SELECT ... FROM cte_VW_CQYZK ...

-- 方式2：临时表（大数据量，需物化）
SELECT * INTO #tmp_VW_CQYZK FROM VW_CQYZK WITH(NOLOCK)
WHERE lrrq BETWEEN ${KSSJ} AND ${JSSJ};

CREATE INDEX idx_tmp_xh ON #tmp_VW_CQYZK(xh);
-- 最后 DROP TABLE #tmp_VW_CQYZK;
```

### 规则3：后置UPDATE数据清洗 → 全量内联至SELECT字段

**核心原则**：原SP中所有在 `INSERT` 之后的 `UPDATE` 语句，都必须在 `SELECT` 列中直接计算出最终值。

#### ⚠️ 3.0 UPDATE转JOIN/子查询的关键原则（最重要）

**问题背景**：原SP中的UPDATE语句从其他表获取字段值（如医保编码），转换时常见错误是直接使用 LEFT JOIN，但这可能导致：

| 转换方式 | 问题 | 正确方案 |
|----------|------|----------|
| `LEFT JOIN 表 ON 条件` | **多行匹配导致数据膨胀**，即使加 DISTINCT 也无法正确去重（字段值不同） | 使用子查询 `SELECT TOP 1` 或 `OUTER APPLY` |
| 直接"修正"原SP逻辑 | **违反规则0**，原SP的"错误"逻辑必须原样保留 | 分析原SP条件，严格按原逻辑转换 |

**示例对比**：

```sql
-- ❌ 错误：LEFT JOIN 导致数据膨胀
-- 原 UPDATE 条件：xmlb=0(idm<>0)时从YK_YPCDMLK获取，xmlb=1(idm=0)时从YY_SFXXMK获取
-- 转换时"修正"为：idm=0→YK_YPCDMLK，idm<>0→YY_SFXXMK（逻辑完全反了！）
SELECT DISTINCT ...
FROM base fx
LEFT JOIN YK_YPCDMLK ypcdm ON fx.idm = 0 AND fx.idm = ypcdm.idm  -- idm=0时匹配药品目录
LEFT JOIN YY_SFXXMK sfxx ON fx.idm <> 0 AND fx.ypdm = sfxx.id    -- idm<>0时匹配收费项目
-- 问题1：逻辑与原SP相反
-- 问题2：LEFT JOIN多行匹配会产生笛卡尔积
-- 问题3：DISTINCT无法去重（dydm值可能不同）

-- ✓ 正确：使用子查询确保一对一，并保持原SP逻辑
SELECT ...,
    COALESCE(
        NULLIF(
            CASE
                -- 原SP逻辑：xmlb=0(idm<>0)用YK_YPCDMLK，xmlb=1(idm=0)用YY_SFXXMK
                WHEN fx.idm <> 0 THEN ISNULL((
                    SELECT TOP 1 ypcdm.dydm
                    FROM YK_YPCDMLK ypcdm WITH(NOLOCK)
                    WHERE ypcdm.idm = fx.idm AND ISNULL(ypcdm.dydm, '') <> ''
                ), '')
                ELSE ISNULL((
                    SELECT TOP 1 sfxx.dydm
                    FROM YY_SFXXMK sfxx WITH(NOLOCK)
                    WHERE sfxx.id = fx.ypdm AND ISNULL(sfxx.dydm, '') <> ''
                ), '')
            END,
            ''
        ),
        'ZZZZZZZZZZZZZZZ'  -- 默认值
    ) AS mxxmbmyb
FROM base fx
-- 无LEFT JOIN，数据量与原SP一致
```

**判断标准**：
- 原UPDATE从单表获取单字段 → 使用子查询 `(SELECT TOP 1 ...)`
- 原UPDATE从单表获取多字段 → 使用 `OUTER APPLY (SELECT TOP 1 ...)`
- 原UPDATE涉及复杂聚合 → 使用独立CTE + LEFT JOIN

#### 3.1 空值/空串 → 默认值

```sql
-- 字符串字段（需trim后判断空串）
COALESCE(NULLIF(RTRIM(LTRIM(字段)), ''), '默认值')

-- 日期字段
COALESCE(日期字段, '1900-01-01')
```

| 原SP后置UPDATE | 转换后SELECT内联表达式 |
|----------------|------------------------|
| `update set yzlx='99' where yzlx='' or yzlx is null` | `COALESCE(NULLIF(CASE ... END, ''), '99') AS yzlx` |
| `update set ypgg='-' where ypgg='' or ypgg is null` | `COALESCE(NULLIF(SUBSTRING(ypgg,1,64), ''), '-') AS ypgg` |

#### 3.2 特殊值映射

```sql
-- 原: UPDATE v13_tb SET yypcdm='ST' WHERE yypcdm='ONCE'
-- 转:
CASE WHEN pc.[name] = 'ONCE' THEN 'ST' ELSE pc.[name] END AS yypcdm
```

#### 3.3 复杂后置逻辑内联

转换为**独立的派生表（CTE）**，然后在最终查询中 `LEFT JOIN`：

```sql
-- 原SP UPDATE
UPDATE a SET a.hzzs = ISNULL(CONVERT(VARCHAR(500), SUBSTRING(B.wbnr,1,500)), '1')
FROM #ghzdk a JOIN OUTP_JZJLK A ON a.xh = A.ghxh
JOIN EMR_OUTP_BLWJJGNRK B ON ...

-- 转换后（CTE）
hzzs_cte AS (
    SELECT A.ghxh AS xh,
           COALESCE(CONVERT(VARCHAR(500), SUBSTRING(B.wbnr,1,500)), '1') AS hzzs
    FROM OUTP_JZJLK A
    JOIN EMR_OUTP_BLWJJGNRK B ON ...
    WHERE B.wjjgdm = 'DLDM.280'
)
-- 最终 SELECT 中：
COALESCE(h.hzzs, '1') AS hzzs
```

#### 3.4 诊断信息的多行转一行（条件聚合）

```sql
diag_cte AS (
    SELECT ghxh AS xh,
           MAX(CASE WHEN zdlx=0 AND zdlb=0 THEN ISNULL(SUBSTRING(zdmc,1,100),'-') END) AS mzzzyzd,
           MAX(CASE WHEN zdlx=0 AND zdlb=0 THEN ISNULL(zddm,'-') END) AS mjzzyzdmb,
           MAX(CASE WHEN zdlx=1 AND zdlb=0 THEN ISNULL(SUBSTRING(zdmc,1,500),'-') END) AS mjzqtzd
    FROM VW_SF_YS_MZBLZDK
    GROUP BY ghxh
)
```

### 规则4：关联、WHERE过滤、UNION ALL结构完整保留

#### 4.1 关联表与过滤条件
- `JOIN` / `LEFT JOIN`：表名、别名、关联条件 **1:1** 原样迁移
- 业务过滤条件（如 `yz.yzzt <> 3`）**必须保留**
- 冗余日期范围过滤建议移除（避免重复扫描）

#### 4.2 UNION ALL 结构
- 两个分支必须保持原顺序与完整性
- 每个分支内字段数量、类型、顺序必须一致
- **必须为每个分支添加 `DISTINCT`**

#### 4.3 ⚠️ DISTINCT 使用规则（新增）

| 场景 | 是否使用DISTINCT | 原因 |
|------|------------------|------|
| UNION ALL 多分支去重 | **必须使用** | 原 SP 依赖目标表唯一约束去重 |
| LEFT JOIN 多表关联 | **禁止使用** | 多行匹配会导致不同字段值，DISTINCT 无法正确去重 |
| 原SP无DISTINCT/去重机制 | **禁止使用** | 会改变原数据量，导致结果不一致 |
| 子查询替代LEFT JOIN后 | **不需要** | 子查询确保一对一，无需去重 |

**关键原则**：
- `DISTINCT` 仅用于 **UNION ALL 分支去重**
- 若 LEFT JOIN 产生多行匹配，**应改用子查询/OUTER APPLY**，而非依赖 DISTINCT
- DISTINCT 去重基于所有字段值，若关联表字段不同，无法达到预期去重效果

#### 4.4 字段映射对照（以医嘱为例）

| 分支 | yzid前缀 | yzlb规则 |
|------|----------|----------|
| 长期医嘱（CQ） | `'cq' + CONVERT(VARCHAR, yz.xh)` | `CASE WHEN yz.yzlb=12 THEN 3 ELSE 1 END` |
| 临时医嘱（LS） | `'ls' + CONVERT(VARCHAR, yz.xh)` | `CASE WHEN yz.yzlb=12 THEN 3 ELSE 2 END` |

### 规则5：常量、硬编码字段原样保留

- 固定字符串：`'JKGKHSKP'`、`'-'`（**注意是单横线**）、`'0'`、`'1'` 等直接使用
- 拼接常量：`'cq'`、`'ls'`、`SUBSTRING('42503274500', 10, 2)` 完全保留
- 数值常量：`0`、`1`、`3` 等不做修改
- 业务固定逻辑：皮试判别 `'0'`、修改标志 `1` 等保持原值

### 规则6：字段列表与注释处理

#### 6.1 字段顺序严格对齐
原SP中 `INSERT INTO 目标表 (字段列表) SELECT ...`，转换后**必须保持 `SELECT` 后的字段顺序与原目标表字段顺序完全一致**

#### 6.2 注释清理
- 删除原SP中的行内注释（`--` 说明性注释），保留重要业务注释
- 在SQL文件头部增加统一注释，包含：查询名称、原始存储过程名称、参数说明、转换日期、依赖的表/视图说明

### 规则7：移除索引提示与锁优化

- 删除所有 `INDEX([...])` 提示
- **保留 `WITH(NOLOCK)`** 以避免锁争用
- 若原SP使用 `WITH(READUNCOMMITTED)`，同样保留

### 规则8：性能优化专项指南

#### 8.1 避免重复扫描基础表

**策略A（推荐）**：将基础数据物化为临时表，并创建主键索引
**策略B（纯CTE）**：每个补充数据源写成独立的CTE，直接从原始源表获取数据

#### 8.2 派生表独立性

```sql
-- 错误：依赖其他CTE会导致重复扫描
WITH base AS (...),
     hzzs AS (SELECT ... FROM base ...)  -- ❌

-- 正确：仅使用base的键值或直接从源表
WITH base AS (...),
     hzzs AS (SELECT ... FROM OUTP_JZJLK ... WHERE ghxh IN (SELECT xh FROM base))  -- ✓
```

#### 8.3 费用明细聚合注意点

`GROUP BY` 必须只包含主键字段（如 `xh`），**不能包含分类字段**，否则会造成结果集膨胀

#### 8.4 避免标量子查询，改用LEFT JOIN或OUTER APPLY

```sql
-- 低效
SELECT (SELECT TOP 1 zdmc FROM VW_SF_YS_MZBLZDK WHERE ghxh = a.xh AND zdlx=0) AS mzzzyzd

-- 高效
OUTER APPLY (SELECT TOP 1 zdmc FROM VW_SF_YS_MZBLZDK WHERE ghxh = a.xh AND zdlx=0 ORDER BY ...) d
```

#### 8.5 尾差处理内联

```sql
CASE WHEN (zfje_calc > mjzzf_calc AND zfje_calc - mjzzf_calc < 0.1) THEN mjzzf_calc ELSE zfje_calc END AS zfje
```

### 规则9：字段映射验证

转换完成后，必须对比原始存储过程中 `UPDATE` 语句修改的字段名与CTE中输出的字段名是否一致。

**常见错误**：
- 原SP更新 `sfsy`，CTE中误写为 `sfys`
- 原SP更新 `hzzs`，CTE中误写为 `hzsz` 或 `hzzs_val`
- 常量默认值错误：原SP使用 `'-'`，CTE中误写为 `'--'`

**建议**：在转换过程中，建立一个字段对照表，逐一核对。

## 坑与对策

| 问题 | 对策 |
|------|------|
| **原代码bug被"修正"导致结果不一致** | **规则0：原样保留原代码逻辑，禁止自动修正** |
| **LEFT JOIN多行匹配导致数据膨胀** | **规则3.0：使用子查询 `SELECT TOP 1` 或 `OUTER APPLY` 确保一对一** |
| **DISTINCT无法正确去重关联表数据** | **规则4.3：仅用于UNION ALL去重，LEFT JOIN场景改用子查询** |
| 大SP单次处理太慢 | >500行用 task 拆为子代理并行处理 |
| UPDATE手写效率低 | 分类批量处理：单表赋值、跨表关联、复杂多步 |
| CTE链字段混乱 | 基础CTE只保留必要字段，用 LEFT JOIN 合并修改层 |
| 中文注释乱码 | 乱码注释直接删除 |
| INSERT字段错位 | SELECT字段顺序严格对应INSERT列清单 |
| 基础表重复扫描 | 使用临时表物化或独立派生CTE |
| 费用聚合多行 | GROUP BY 仅含主键字段，不含分类字段 |
| 标量子查询低效 | 改用 OUTER APPLY 或预聚合CTE |
| 字段名引用错误被修正 | 对比原SP表达式与转换后表达式，确保完全一致 |

## 转换质量检查清单

### 基础检查
- [ ] **每个表达式是否与原SP完全一致（含bug）？**
- [ ] **字段名引用错误是否原样保留？**
- [ ] **UPDATE转查询时，是否确保一对一匹配（子查询/OUTER APPLY而非LEFT JOIN）？**
- [ ] **DISTINCT仅用于UNION ALL去重，而非补救LEFT JOIN多行匹配？**
- [ ] 删除所有过程化语法？
- [ ] 删除所有 DML（DELETE、INSERT、SELECT INTO 业务表、DROP）？
- [ ] 所有 `#临时表` → CTE 或物化临时表？
- [ ] 移除 `INDEX([...])`，仅保留 `NOLOCK`？
- [ ] 参数改为 `${KSSJ}`、`${JSSJ}` 占位符？
- [ ] 后置 UPDATE 内联到 SELECT？
- [ ] 空值处理使用 `COALESCE(NULLIF(...))`？
- [ ] 特殊值映射用 `CASE WHEN`？
- [ ] 保留 JOIN、WHERE、UNION ALL 结构？
- [ ] UNION ALL 分支增加 `DISTINCT`？
- [ ] 字段顺序与原目标表一致？

### 性能检查
- [ ] **>500行用 task 子代理？**
- [ ] **UPDATE优先提取为CTE而非关联子查询？**
- [ ] **基础数据源扫描次数可控（≤2次）？**
- [ ] **费用明细聚合正确使用 GROUP BY xh？**
- [ ] **避免标量子查询，改用 LEFT JOIN 或 OUTER APPLY？**
- [ ] **大数据量场景考虑临时表物化？**

### 字段验证
- [ ] **所有原UPDATE中出现的字段名，在最终SELECT中均有正确映射且无拼写错误**
- [ ] **常量默认值正确（注意 `'-'` 不是 `'--'`）**

## 参考文件

- 完整转换规范：`references/转换规则.md`
- 输入示例：`assets/输入示例_原始存储过程.txt`
- 输出示例：`assets/输出示例_转换后SQL.txt`