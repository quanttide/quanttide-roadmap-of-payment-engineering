# 应用层文档更新计划（apps/qtcloud-pay/docs）

`apps/qtcloud-pay/docs/` 下三篇对外文档的更新计划 —— 对齐 [provider.md](provider.md) 里程碑 M1–M4（账本核心 v0.1.0）与 [tests.md](tests.md) 的 API 事实源，把文档从 v0.0.1 支付通道时代升级到 v0.1.0 账本核心。

- 负责人：shishiyuan
- 配套文档：[provider.md](provider.md)（服务端路线图）、[studio.md](studio.md)（操作台）、[tests.md](tests.md)（测试用例）、[cli.md](cli.md)（CLI）
- 事实源：端点与字段以集成测试（tests.md / `internal/itest`）为准，设计依据服务端设计文档（[src/provider/docs](../../../apps/qtcloud-pay/src/provider/docs/index.md)）

## 一、现状

| 文档 | 读者 | 现状（v0.0.1 通道） | 问题 |
|------|------|--------------------|------|
| `user-guide.md` | 管理人员/商户 | 支付接入指南：`POST /pay`、`GET /query/{id}`、`POST /refund`，金额浮点元 | 无账本核心内容；读者与 v0.1.0 实际使用方式不符 |
| `api-reference.md` | 联调开发 | 通道三端点参考 | 缺 v0.1.0 全部账本端点；金额单位需统一为整数分 |
| `dev-guide.md` | 开发者/运维 | 通道时代目录、配置（渠道环境变量）、测试策略 | 目录/配置/测试未含账本模块 |

## 二、更新目标

三篇文档随 M1–M4 交付逐里程碑更新，达到：

1. **用户指南**覆盖管理人员日常操作：开户、充值登记、发券、下单结算、每日对账（对齐 studio.md §2.1 操作入口速查与 §2.2–2.5 流程）
2. **API 参考**覆盖 v0.1.0 全部端点，金额统一整数分，与集成测试对齐（保留通道三端点，标注 M5 预留）
3. **开发指南**反映账本模块结构、双存储配置（SQLite/PostgreSQL）与集成测试策略

通道内容（`POST /pay` 等）不删除，v0.2.0（M5）前保留在文档中并标注「v0.0.1 通道能力，M5 接入后启用」。

## 三、分文档更新计划

### 3.1 api-reference.md（随里程碑增量更新）

新增端点，按 provider.md 里程碑分组：

| 里程碑 | 端点 | 说明 |
|--------|------|------|
| M1 | `POST /accounts` | 创建账户 |
| M1 | `POST /accounts/{id}/recharges` | 充值登记（幂等键：打款凭证号 `voucher_no`） |
| M1 | `GET /accounts/{id}` | 账户与余额 |
| M1 | `GET /accounts/{id}/transactions` | 交易流水（类型/金额/方向/来源） |
| M2 | `POST /accounts/{id}/coupons` | 发放优惠券（幂等键：发放批次号 `batch_no`） |
| M2 | `POST /accounts/{id}/vouchers` | 发放代金券（幂等键：发放批次号 `batch_no`） |
| M2 | `GET /accounts/{id}/coupons`、`GET /accounts/{id}/vouchers` | 券列表（状态：已发放/已使用/已过期） |
| M3 | `POST /orders` | 下单并结算（幂等键：订单号 `order_id`；返回抵扣明细） |
| M3 | `GET /orders/{id}` | 订单与结算明细 |
| M4 | `GET /accounts/{id}/statement` | 账单导出（对公核对用） |

同步更新：

- 金额单位：全篇统一整数分（现有通道示例 `99.99` 元改为分，或加单位说明）
- 错误码：补充 v0.1.0 错误场景（幂等冲突、过期券、余额不足 422、余额不足整体回滚，对齐 tests.md）
- 幂等键字段命名对齐服务端约定（conventions.md）：`voucher_no` / `batch_no` / `order_id`
- 文档顶部事实源声明保持（以集成测试为准）

### 3.2 user-guide.md（M1 后重写主体）

读者从「接入支付的商户」改为「管理人员」，主体改为账本核心使用流程（对齐 studio.md §二）：

| 章节 | 内容 | 依据 |
|------|------|------|
| 账户管理 | 创建账户、查余额、查流水 | studio.md §2.1 |
| 充值登记 | 对公打款 → 凭证号登记 → 核对余额增加；重复提交拦截 | studio.md §2.2 |
| 发券 | 优惠券/代金券发放、有效期与状态流转 | studio.md §2.3 |
| 订单结算 | 下单、抵扣顺序（优惠券 → 代金券 → 余额）、结算明细核对 | studio.md §2.4 |
| 每日对账 | 账单导出、余额校验、银行流水 CSV 比对 | studio.md §2.5 |
| 错误处理 | 异常现象与处置（重复登记/过期券/余额对不上） | studio.md §五 |

支付通道章节保留并标注「v0.2.0 启用」（M5 前作为参考）。

### 3.3 dev-guide.md（M1 交付时更新）

- 项目结构：新增账本模块目录（account/transaction/coupon/voucher/order/billing/reconciliation），对齐 provider/docs/index.md 目录结构
- 配置：新增 `DB_DRIVER` / `DATABASE_URL`（开发 SQLite / 生产 PostgreSQL，GORM 方言切换）；渠道环境变量保留
- 测试策略：新增 `internal/itest` 场景级集成测试（SQLite `:memory:` + 全模块组装，对齐 tests.md §测试环境）；原通道测试保持不变
- 添加新模块指引：更新「添加新支付提供商」为「新增账本模块」（含写 `docs/<module>.md` 的约定）

## 四、里程碑对应

| 里程碑 | 更新动作 | 验收 |
|--------|---------|------|
| M1 账户与账本 | api-reference 补账户/充值/流水；user-guide 重写主体；dev-guide 更新结构/配置/测试 | 三篇覆盖 M1 端点与流程，无通道时代残留的账本缺失 |
| M2 优惠券与代金券 | api-reference 补券端点；user-guide 补发券章节 | 端点、字段、状态枚举与 tests.md 一致 |
| M3 订单与计费规则 | api-reference 补订单端点；user-guide 补结算与抵扣顺序 | 结算明细结构与 provider.md §M3 一致 |
| M4 对账与可查 | api-reference 补 statement；user-guide 补每日对账 | 对账流程与 studio.md §2.5 一致 |
| M5 支付通道对接（v0.2.0） | 三篇启用通道章节：回调自动入账、`POST /pay` 等转正 | 排期后单独规划 |

## 五、维护约定

- **随里程碑更新，不攒到版本末**：每个里程碑验收前完成对应章节更新
- **事实源对齐**：API 参考以集成测试为事实源；联调契约以服务端设计文档为准
- **金额单位**：API 一律整数分；用户指南示例以元展示并注明换算（如 100.00 元 = 10000 分）
- **不删除通道内容**：v0.0.1 通道三端点保留并标注 M5 预留，避免 v0.2.0 重复编写
