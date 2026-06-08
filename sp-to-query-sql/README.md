# SP to Query SQL 转换技能

将 SQL Server 存储过程自动转换为纯只读接口查询 SQL。

## 功能特性

| 特性 | 说明 | 状态 |
|------|------|------|
| 过程化代码删除 | 移除 CREATE PROC、BEGIN、END、GO、USE 等 | ✓ |
| 临时表转CTE | `#临时表` → `cte_xxx`，移除 INDEX 提示 | ✓ |
| 后置UPDATE内联 | UPDATE 清洗逻辑转为 SELECT 字段表达式 | ✓ |
| 参数占位符替换 | `@参数` → `${KSSJ}`、`${JSSJ}` | ✓ |
| 空值智能处理 | `COALESCE(NULLIF(..., ''), 默认值)` | ✓ |
| 特殊值映射 | `CASE WHEN` 实现值转换 | ✓ |
| 大SP并行处理 | >500行自动拆分为子代理处理 | ✓ |

## 环境要求

- 无特殊依赖，纯文本处理
- 适用于 SQL Server 存储过程转换

## 使用方式

### 1. 作为 WinCoder 技能使用

**触发话术示例**：

```
将 E:\work\sp\usp_yzmx.sql 转换为查询SQL

把这个存储过程转成纯SELECT语句

医院上报接口需要转换这个SP
```

### 2. 参数格式

```
/sp-to-query-sql <存储过程文件路径> [目标SQL文件路径]
```

| 参数 | 必需 | 说明 |
|------|------|------|
| 存储过程文件路径 | ✓ | 原始 SP SQL 文件的完整路径 |
| 目标SQL文件路径 | ✗ | 输出文件路径，默认直接返回结果 |

## 转换规则速查

### 临时表 → CTE 映射

```sql
#VW_XXX → cte_VW_XXX
```

### 空值处理模板

```sql
COALESCE(NULLIF(RTRIM(LTRIM(字段)), ''), '默认值')   -- 字符串
COALESCE(日期字段, '1900-01-01')                     -- 日期
```

### 特殊值映射

```sql
CASE WHEN pc.[name] = 'ONCE' THEN 'ST' ELSE pc.[name] END
```

## 输出示例

**成功转换输出**：
```
✅ 存储过程转换完成！

转换摘要：
- 原始行数: 1,800
- 转换后行数: 450
- 删除的临时表: #VW_CQYZK, #VW_LSYZK (已转为CTE)
- 内联的UPDATE: 15个
- 参数占位符: @kssj → ${KSSJ}, @jssj → ${JSSJ}
```

## 项目结构

```
sp-to-query-sql/
├── SKILL.md              # 技能定义文件
├── README.md             # 本文档
├── assets/               # 示例文件
│   ├── 输入示例_原始存储过程.txt
│   └── 输出示例_转换后SQL.txt
└── references/           # 参考文档
    └── 转换规则.md
```

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0.0 | 2026-06-05 | 初始版本 |

## 维护者

韩抒豪

## 相关资源

- [SKILL规范文档](../SKILL规范.md)
- [转换规则详解](references/转换规则.md)