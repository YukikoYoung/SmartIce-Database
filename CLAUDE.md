# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

野百灵餐饮集团多品牌连锁管理数据库系统 - PostgreSQL 15 数据库，包含47张表、17个视图，支持双成本率财务分析和BOM递归分解。

| 属性 | 值 |
|-----|---|
| 数据库 | PostgreSQL 15 (Homebrew) |
| 数据库名 | yebailing_db |
| Python库 | pandas, openpyxl, psycopg2 |
| 扩展依赖 | pgcrypto, uuid-ossp, pg_trgm, btree_gin |

## Common Commands

```bash
# 连接数据库
psql yebailing_db

# 执行SQL文件
psql yebailing_db -f db/schema/schema_core_mvp.sql

# 运行Python ETL脚本 (使用uv虚拟环境)
uv run python db/scripts/generate_cost_card_sql.py
uv run python db/scripts/import_platform_data.py

# 测试BOM分解函数
psql yebailing_db -c "SET app.encryption_key = 'ybl-restaurant-encryption-key-2025'; SELECT * FROM explode_bom((SELECT product_id FROM product WHERE product_name = '贵州非遗丝娃娃'), 1.0);"

# 验证表数量
psql yebailing_db -c "SELECT count(*) FROM pg_tables WHERE schemaname = 'public';"
```

## SQL Execution Order

严格按此顺序执行（存在依赖关系）：

1. `db/schema/schema_core_mvp.sql` - 核心表结构
2. `db/functions/functions_cost_encryption.sql` - 加密函数
3. `db/functions/functions_bom_explosion.sql` - BOM递归分解
4. `db/functions/triggers_automatic_calculation.sql` - 触发器
5. `db/functions/procedures_data_validation.sql` - 存储过程
6. `db/data/data_init_mvp.sql` - 初始化数据
7. `db/data/data_cost_card_import.sql` - 成本卡数据
8. `db/schema/schema_extension_order_system.sql` - 订单系统扩展
9. `db/views/views_financial_analysis.sql` - 财务视图
10. `db/views/views_operations_kpi.sql` - 运营KPI视图

## Architecture

```
db/
├── schema/          # 表结构DDL (先执行)
├── functions/       # 函数、触发器、存储过程 (依赖schema)
├── data/            # 数据导入SQL (依赖functions)
├── views/           # 视图 (最后执行，依赖所有表)
└── scripts/         # Python ETL脚本
```

**表结构分层**：
- 核心MVP (24张): organization_unit, brand, store, product, recipe, recipe_item等
- 订单扩展 (13张): sales_order, sales_order_item, group_buy_deal等
- 平台数据 (4张): platform_daily_operations等

## Core Business Rules

**双成本率计算（核心公式）**：
```sql
-- 永远用折后收入作为分母！
实际成本率 = 理论成本 / 菜品收入(折后) × 100%
标准成本率 = 理论成本 / 销售额(折前) × 100%
成本率差异 = 实际成本率 - 标准成本率  -- 促销侵蚀程度
```

**BOM分解**：11个半成品需递归分解到原材料（水晶滑肉混合物、香茅酱、干巴菌酱等）。使用 `explode_bom()` 函数。

**成本数据加密**：所有成本相关字段使用pgcrypto加密，操作前需设置：
```sql
SET app.encryption_key = 'ybl-restaurant-encryption-key-2025';
```

## Known Issues

1. **UNIQUE约束语法错误**: `schema_core_mvp.sql`中`COALESCE()`在UNIQUE约束中会失败，需手动移除
2. **订单系统表依赖顺序**: `sales_order`引用`group_buy_deal`，需按正确顺序创建
3. **explode_bom函数列名**: 如遇`unit_price_encrypted`不存在，改用`unit_price`

## Documentation Index

| 文档 | 内容 |
|-----|------|
| `docs/DESIGN.md` | 数据库架构设计、ER图 |
| `docs/BUSINESS_RULES.md` | 业务规则、计算公式详解 |
| `docs/DEPLOYMENT.md` | 完整部署步骤、已知问题解决方案 |
| `docs/计量标准.md` | 单位换算规则（g/ml为基准单位） |
