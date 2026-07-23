# My tokens 一期埋点方案

版本：v1.0  
日期：2026-07-15  
适用范围：My tokens 一期原型与首版客户端埋点落地

---

## 1. 一期核心目标

一期上线前，埋点和测试都只围绕以下 3 条核心目标展开：

1. 用户能把 AI 工具顺利下载安装并正常使用。
2. 用户充值或订阅后能及时生效，并马上继续使用。
3. 用户自己的 API Key 能找到入口、配得上、用得起来。

### 1.1 埋点目标

一期埋点只服务 4 个问题，避免埋点过散：

1. 用户是否跑通核心闭环：安装工具 → 选择模型 → 发起调用 → 扣费 / 拦截 → 购买 / 充值 → 继续使用。
2. 用户主要走哪条路径：购买平台能力，还是接入外部供应商。
3. 购买成功后是否带来后续调用，而不是只付款不使用。
4. 用户主要流失在哪个节点：安装、登录、配置、支付，还是调用失败。

### 1.2 埋点验收原则

如果埋点最终不能回答下面 3 个问题，就说明一期口径还不够：

1. 工具为什么没有成功安装、启动或继续使用。
2. 支付成功后为什么没有立即恢复可用。
3. API Key 为什么没有被顺利配置并真正用起来。

---

## 2. 埋点范围

结合当前原型与一期 PRD，首版建议覆盖以下页面与全局弹层：

| 页面 / 模块 | 页面 ID / 入口 | 说明 |
|---|---|---|
| 登录弹层 | ls | 游客转登录、登录方式切换、登录成功 |
| P01 Tokens 管理 | p01 | 首页总览、余额入口、消耗图表 |
| P02 AI 工具管理 | p02 | 工具下载、打开、模型配置 |
| P03 模型供应商 | p03 | 新增供应商、验证、模型拉取、全局路由 |
| P05 订阅 / 充值 | p05 | 订阅购买、按量充值、订单记录 tab |
| P07 我的成就 | p07 | 勋章浏览、复制文案、分享 |
| P08 个人中心 | p08 | 账号信息、API Key、通知设置、订单摘要 |
| 全局弹层 / 系统状态 | 多处 | 无可用额度引导、登录拦截、支付结果、调用结果、异常处理引导 |

说明：P06 订单中心当前原型中挂载在 P05 的 orders tab 内，埋点口径仍按独立业务模块设计。

---

## 3. 命名规则

### 3.1 事件命名

统一使用英文小写下划线，格式：

`业务模块_对象_动作`

示例：

- `auth_login_submit`
- `dashboard_balance_entry_click`
- `supplier_verify_success`
- `purchase_pay_success`
- `api_key_reset_confirm`

### 3.2 事件分类

建议统一打 `event_type` 方便后续筛选：

- `exposure`：曝光
- `click`：点击
- `submit`：提交
- `success`：成功
- `fail`：失败
- `state`：状态变化

---

## 4. 公共属性

以下字段建议作为核心事件公共参数，尽量统一：

| 字段名 | 类型 | 说明 |
|---|---|---|
| event_time | string | 事件发生时间 |
| user_id | string | 登录用户 ID，游客为空 |
| visitor_id | string | 设备或安装实例 ID |
| login_status | string | `guest` / `logged_in` |
| page_id | string | 如 `p01` / `p03` / `login_modal` |
| page_name | string | 页面中文名 |
| source_page_id | string | 上一个页面或来源页面 |
| source_module | string | 来源模块，如 `sidebar` / `topbar_balance` |
| client_version | string | 客户端版本号 |
| channel | string | 安装包渠道或投放渠道 |
| tool_name | string | 工具名称，如 Claude、Codex |
| model_id | string | 模型标识 |
| model_source | string | `platform` / `external_supplier` |
| supplier_id | string | 供应商 ID，内置平台可固定值 |
| billing_mode | string | `subscription` / `balance` / `external_supplier` |
| remain_subscription_count | number | 事件发生时剩余订阅次数 |
| remain_balance | number | 事件发生时可用余额 |
| pay_sku_id | string | 套餐或充值方案 ID |
| fail_reason | string | 失败原因码 |
| fail_message | string | 失败文案，便于回放 |
| exception_scene | string | 异常场景，如安装失败、支付未生效、API Key 配置失败 |
| resolve_method | string | 当前引导方式，如重试、重下、去充值、去配置、联系客服 |

说明：

1. 涉及支付结果、扣费结果、调用结果的事件，最终以服务端事件为准。
2. 同一个行为若有前端点击和后端成功，建议保留两条事件，不要互相替代。

### 4.1 `fail_reason` 枚举建议

说明：

1. `fail_reason` 记录直接失败原因，尽量用标准枚举值，不要直接写整段中文报错。
2. 若需要保留原始错误文案，可写入 `fail_message`。
3. 前后端统一优先级：先写标准码，再补原始信息。

| 枚举值 | 适用场景 | 说明 |
|---|---|---|
| network_error | 通用 | 网络异常、请求超时、连接失败 |
| server_error | 通用 | 服务端 5xx、内部异常 |
| client_error | 通用 | 客户端自身异常、参数拼装错误 |
| validation_failed | 通用 | 表单校验失败、必填项缺失 |
| permission_denied | 通用 | 无权限、权限不足 |
| login_required | 通用 | 需要登录但当前未登录 |
| risk_control_blocked | 登录 / 验证码 / 支付 | 风控拦截 |
| sms_rate_limited | 验证码 | 验证码发送频控 |
| sms_code_invalid | 登录 / 验证码 | 验证码错误或失效 |
| password_invalid | 登录 | 密码错误 |
| agreement_unchecked | 登录 | 未勾选协议 |
| download_failed | 下载 | 下载任务失败 |
| install_package_invalid | 安装 | 安装包损坏或不完整 |
| install_permission_denied | 安装 | 安装权限不足 |
| install_path_invalid | 安装 | 安装路径异常或不可写 |
| app_executable_missing | 启动 | 程序被误删、主程序不存在 |
| app_launch_failed | 启动 | 程序启动失败 |
| app_process_crash | 启动 / 使用 | 启动后崩溃 |
| model_slot_limit_exceeded | 工具配置 | 模型槽位超限 |
| model_unavailable | 工具配置 / 调用 | 模型不可用或已下线 |
| supplier_key_invalid | 供应商 / API Key | API Key 无效 |
| supplier_endpoint_invalid | 供应商 | 请求地址错误 |
| supplier_verification_failed | 供应商 | 供应商检测未通过 |
| model_sync_failed | 供应商 | 模型列表拉取失败 |
| payment_cancelled | 支付 | 用户主动取消支付 |
| payment_timeout | 支付 | 支付超时 |
| payment_channel_error | 支付 | 支付渠道异常 |
| order_create_failed | 支付 | 支付订单创建失败 |
| entitlement_not_effective | 支付后生效 | 支付成功但权益未到账 |
| balance_not_updated | 充值后生效 | 余额未更新 |
| subscription_not_updated | 订阅后生效 | 订阅状态未更新 |
| insufficient_subscription | 调用 | 订阅次数不足 |
| insufficient_balance | 调用 | 余额不足 |
| api_key_missing | API Key | 未填写 API Key |
| api_key_format_invalid | API Key | Key 格式错误 |
| api_key_verification_failed | API Key | Key 校验失败 |
| base_url_invalid | API Key / 供应商 | Base URL 或请求地址不合法 |
| invoice_info_invalid | 开票 | 开票信息格式错误 |
| invoice_not_available | 开票 | 当前订单不可开票 |
| phone_update_failed | 个人资料 | 手机号更新失败 |
| unknown_error | 通用 | 无法归类的兜底错误 |

### 4.2 `exception_scene` 枚举建议

说明：

1. `exception_scene` 记录用户当前遇到的是哪一类业务问题，用于异常弹窗、客服入口和恢复漏斗统一归因。
2. 同一个异常弹层优先只归一个主场景，避免一个事件写多个场景值。

| 枚举值 | 适用场景 | 说明 |
|---|---|---|
| install_failed | 安装 | 下载后安装失败 |
| install_interrupted | 安装 | 安装过程中被中断 |
| app_deleted | 启动 | 下载后被误删、启动文件缺失 |
| app_cannot_open | 启动 | 安装后无法启动 |
| app_crashed | 启动 / 使用 | 启动后崩溃或运行中崩溃 |
| tool_config_failed | 工具配置 | 工具内模型或供应商配置失败 |
| supplier_verify_failed | 供应商 | 供应商接入验证失败 |
| supplier_model_sync_failed | 供应商 | 供应商模型同步失败 |
| payment_failed | 支付 | 支付流程失败或中断 |
| subscription_not_effective | 支付后生效 | 订阅成功但未生效 |
| topup_not_effective | 支付后生效 | 充值成功但余额未到账 |
| post_payment_unusable | 支付后使用 | 支付成功后仍无法继续使用 |
| insufficient_subscription | 调用 | 订阅次数耗尽 |
| insufficient_balance | 调用 | 余额耗尽 |
| api_key_config_failed | API Key | API Key 配置或校验失败 |
| api_key_unusable | API Key | API Key 已配置但无法调用 |
| network_abnormal | 通用 | 网络异常导致功能不可用 |
| order_invoice_failed | 开票 | 开票申请失败 |
| phone_update_failed | 个人资料 | 手机号修改失败 |
| login_failed | 登录 | 登录流程失败 |
| unknown_exception | 通用 | 无法归类的异常兜底场景 |

---

## 5. 页面埋点表

## 5.1 登录弹层

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| auth_login_modal_view | 打开登录弹层 | trigger_source | 来源可区分下载、购买、查看订单、登录拦截 |
| auth_login_method_switch | 切换登录方式 | login_method | `wechat` / `phone` |
| auth_phone_login_mode_switch | 手机登录切换模式 | phone_login_mode | `code` / `password` |
| auth_send_code_click | 点击获取验证码 | phone_masked | 前端点击 |
| auth_send_code_success | 验证码发送成功 | phone_masked | 后端返回成功 |
| auth_send_code_fail | 验证码发送失败 | fail_reason | 建议记录风控、频控、号码异常 |
| auth_agreement_toggle | 勾选协议状态变化 | agree_checked | 用于定位登录前阻塞 |
| auth_login_submit | 点击登录 / 注册 | login_method, phone_login_mode | 前端提交 |
| auth_login_success | 登录成功 | login_method, is_register | 新老用户要区分 |
| auth_login_fail | 登录失败 | fail_reason | 如验证码错误、密码错误、未勾选协议 |

## 5.2 P01 Tokens 管理

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| dashboard_view | 进入 P01 | login_status | 首页 PV |
| dashboard_balance_exposure | 顶部余额区展示 | remain_subscription_count, remain_balance | 已登录态记录真实值 |
| dashboard_balance_entry_click | 点击今日剩余次数或可使用余额 | entry_type | `subscription_balance` / `cash_balance` |
| dashboard_guest_cta_click | 游客点击登录 / 注册或引导按钮 | cta_type | 游客态转化入口 |
| dashboard_tool_card_exposure | 工具卡片区曝光 | tool_name | 工具卡可按可视曝光或首次渲染 |
| dashboard_tool_entry_click | 在总览页点击工具入口 | tool_name, action_type | `download` / `open` / `config` |
| dashboard_usage_tab_click | 切换总览 / 供应商 / 消耗明细 | tab_name | 对应图表使用偏好 |
| dashboard_usage_range_select | 切换时间范围 | range_type | `today` / `7d` / `30d` / `custom` |
| dashboard_usage_custom_apply | 应用自定义时间 | start_date, end_date | 自定义筛选行为 |
| dashboard_usage_filter_click | 点击图表下方模型或供应商卡片筛选 | filter_type, filter_value | 观察用户是否做深度分析 |
| dashboard_tutorial_click | 点击 AI 工具配置教程 | source_module | 判断教程需求强度 |

## 5.3 P02 AI 工具管理

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| tool_manage_view | 进入 P02 | login_status | 页面 PV |
| tool_tab_switch | 切换工具卡或工具标签 | tool_name | 多工具偏好 |
| tool_download_click | 点击下载工具 | tool_name | 前端点击 |
| tool_download_success | 工具下载完成 | tool_name, package_version | 建议安装器或下载服务回传 |
| tool_install_success | 工具安装成功 | tool_name, package_version | 与下载成功区分，作为可启动前置条件 |
| tool_install_fail | 安装失败 | tool_name, fail_reason, exception_scene | 安装器或客户端回传 |
| tool_reinstall_guide_view | 展示重新下载 / 重新安装引导 | tool_name, exception_scene | 覆盖安装失败、误删后恢复 |
| tool_reinstall_click | 点击重新下载 / 重新安装 | tool_name, resolve_method | 观察异常后恢复率 |
| tool_open_click | 点击打开工具 | tool_name | 核心使用起点 |
| tool_open_success | 工具成功启动 | tool_name | 区分点击打开与真正拉起成功 |
| tool_open_fail | 点击打开后启动失败 | tool_name, fail_reason, exception_scene | 如文件缺失、路径失效、权限异常 |
| tool_reopen_guide_view | 展示打不开引导弹层 | tool_name, exception_scene | 引导用户重下或重装 |
| tool_setup_guide_open | 打开工具配置教程 | tool_name | 与下载 / 打开联动分析 |
| tool_model_config_open | 打开更换模型或精细配置 | tool_name | 进入配置漏斗 |
| tool_model_change_submit | 提交工具级模型调整 | tool_name, model_id, supplier_id | 前端提交 |
| tool_model_change_success | 工具级模型调整成功 | tool_name, model_id, supplier_id | 后端配置写入成功 |
| tool_model_change_fail | 工具级模型调整失败 | tool_name, fail_reason | 如槽位超限、模型不可用 |
| tool_supplier_override_open | 打开工具维度供应商覆盖配置 | tool_name | 判断高级配置需求 |

## 5.4 P03 模型供应商

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| supplier_page_view | 进入 P03 | login_status | 页面 PV |
| supplier_builtin_card_exposure | 内置供应商卡片曝光 | supplier_id | 观察平台供应商关注度 |
| supplier_add_click | 点击添加供应商 | source_module | 核心配置入口 |
| supplier_preset_select | 选择预设供应商类型 | supplier_template | 观察常用供应商分布 |
| supplier_form_submit | 提交供应商表单验证 | supplier_name, has_official_url, has_request_url | 前端提交 |
| supplier_verify_success | 供应商验证成功 | supplier_id, supplier_name | 新增漏斗核心成功点 |
| supplier_verify_fail | 供应商验证失败 | supplier_name, fail_reason | 如地址错误、Key 无效 |
| supplier_model_sync_success | 拉取模型成功 | supplier_id, model_count | 验证后自动拉模 |
| supplier_model_sync_fail | 拉取模型失败 | supplier_id, fail_reason | 与验证成功区分开 |
| supplier_global_route_auto_fill | 系统自动填充 4 个全局路由槽位 | supplier_id, filled_slot_count | 用于判断默认策略有效性 |
| supplier_global_route_save | 保存全局路由 | supplier_id, route_model_ids | 记录 4 个槽位结果 |
| supplier_recheck_click | 点击重新检测 | supplier_id | 连接稳定性问题定位 |
| supplier_edit_submit | 编辑供应商后提交 | supplier_id | 配置维护频次 |
| supplier_delete_click | 点击删除供应商 | supplier_id | 删除意图 |
| supplier_delete_confirm | 二次确认删除 | supplier_id | 真正删除动作 |
| supplier_delete_success | 删除供应商成功 | supplier_id | 与工具配置影响联动分析 |

## 5.5 P05 订阅 / 充值

### 5.5.1 页面级事件

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| purchase_page_view | 进入 P05 | source_module | 来源要区分侧边栏、余额卡、拦截弹层 |
| purchase_tab_click | 切换 tab | tab_name | `subscription` / `topup` / `orders` |

### 5.5.2 订阅制 Tab

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| subscription_plan_exposure | 套餐卡片曝光 | pay_sku_id, plan_name | 套餐浏览 |
| subscription_plan_select | 选择订阅套餐 | pay_sku_id, plan_name, price | 核心支付前动作 |
| subscription_rule_expand | 展开订阅规则说明 | section_name | 如退款规则、计次说明 |
| subscription_metering_expand | 展开详细计费档位与模型表 | section_name | 判断用户是否在意模型口径 |
| subscription_autorenew_toggle | 自动续费开关变化 | auto_renew_enabled | 支付策略意愿 |
| subscription_pay_click | 点击立即支付 | pay_sku_id, pay_method | 前端点击 |
| subscription_pay_success | 订阅支付成功 | pay_sku_id, pay_method, order_id | 以服务端支付成功为准 |
| subscription_pay_fail | 订阅支付失败 | pay_sku_id, pay_method, fail_reason | 支付漏斗关键流失点 |
| subscription_effect_fail | 支付成功后订阅未及时生效 | order_id, fail_reason, exception_scene | 支付成功但权益未到账 |

### 5.5.3 按量充值 Tab

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| topup_plan_exposure | 充值方案曝光 | pay_sku_id, amount, bonus_amount | 方案浏览 |
| topup_plan_select | 选择充值方案 | pay_sku_id, amount, bonus_amount | 核心支付前动作 |
| topup_price_table_expand | 展开全模型价格表 | section_name | 价格敏感度判断 |
| topup_auto_recharge_toggle | 自动充值开关变化 | auto_recharge_enabled, threshold_amount | 后续高级策略 |
| topup_pay_click | 点击立即充值 | pay_sku_id, pay_method | 前端点击 |
| topup_pay_success | 充值成功 | pay_sku_id, pay_method, order_id, credited_amount | 服务端为准 |
| topup_pay_fail | 充值失败 | pay_sku_id, pay_method, fail_reason | 支付漏斗关键流失点 |
| topup_effect_fail | 支付成功后余额未及时生效 | order_id, fail_reason, exception_scene | 支付到账异常重点排查 |

## 5.6 P06 订单中心（当前承载于 P05 orders tab）

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| order_center_view | 进入订单记录 tab | source_page_id | 订单中心 PV |
| order_type_filter_click | 切换订单类型 | order_type | `all` / `subscription` / `topup` |
| order_status_filter_change | 切换订单状态筛选 | order_status | `all` / `pending` / `success` / `refunded` |
| order_detail_click | 点击单条订单查看 / 操作 | order_id, order_type, order_status | 单条订单关注 |
| order_select | 勾选订单 | order_id, invoice_available | 为批量开票做准备 |
| order_select_all | 全选订单 | selected_count | 批量动作前置信号 |
| order_invoice_entry_click | 点击申请开票 | selected_count | 可区分单笔 / 批量 |
| order_invoice_submit | 提交开票申请 | selected_count, invoice_title_type | 前端提交 |
| order_invoice_success | 开票申请成功 | selected_count | 服务端回写 |
| order_invoice_fail | 开票申请失败 | fail_reason | 常见于邮箱、税号、状态不符 |

## 5.7 P07 我的成就

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| achievement_page_view | 进入 P07 | current_badge_id | 页面 PV |
| achievement_badge_click | 点击勋章卡片 | badge_id, badge_status | `unlocked` / `locked` |
| achievement_badge_detail_view | 打开勋章详情 | badge_id | 详情关注度 |
| achievement_copy_click | 点击复制成就文案 | badge_id | 判断分享前一步 |
| achievement_share_click | 点击分享当前成就 | badge_id, share_channel | 若无渠道可先记 `unknown` |
| achievement_avatar_badge_click | 点击侧边栏头像勋章挂件 | badge_id, source_module | P07 与 P08 的重要导流点 |

## 5.8 P08 个人中心

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| profile_view | 进入 P08 | default_section | 页面 PV |
| profile_section_switch | 切换个人中心分区 | section_name | `account` / `achievement` / `api` / `notice` |
| profile_renew_entry_click | 点击续费或购买入口 | source_module | 从个人信息区导流购买 |
| profile_order_entry_click | 点击查看订单 | source_module | 导流到订单中心 |
| profile_badge_click | 点击头像勋章 | badge_id | 与 P07 联动 |
| api_key_entry_exposure | API Key 入口曝光 | source_module | 判断入口是否足够容易被发现 |
| api_key_copy_click | 点击复制 API Key | key_type | 平台 Key 核心使用动作 |
| api_key_entry_click | 点击进入 API Key 管理 | source_module | 判断入口是否找得到 |
| api_key_config_submit | 提交 API Key 配置或验证 | supplier_id, tool_name | 覆盖平台 Key 和外部 Key 配置 |
| api_key_config_success | API Key 配置成功 | supplier_id, tool_name | 是否配得上 |
| api_key_config_fail | API Key 配置失败 | supplier_id, tool_name, fail_reason, exception_scene | 是否用得起来前的核心阻塞 |
| api_key_reset_click | 点击重置 API Key | key_type | 风险动作起点 |
| api_key_reset_confirm | 确认重置 API Key | key_type | 二次确认 |
| api_key_reset_success | 重置 API Key 成功 | key_type | 影响工具可用性 |
| api_key_guide_click | 点击前往设置或教程 | guide_type | 指导用户接入第三方工具 |
| notice_toggle_change | 通知开关变化 | notice_type, enabled | 如余额不足提醒、服务中断提醒 |
| profile_phone_update_submit | 提交手机号修改 | has_code | 若一期保留该动作建议埋 |
| profile_phone_update_success | 手机号修改成功 |  | 账号资料维护 |
| profile_phone_update_fail | 手机号修改失败 | fail_reason | 失败定位 |

## 5.9 全局弹层与系统状态

| 事件名 | 触发时机 | 关键属性 | 备注 |
|---|---|---|---|
| login_required_popup_view | 触发登录拦截弹层 | trigger_action | 哪个动作被登录拦住 |
| no_token_guide_view | 打开无可用额度引导弹层 | trigger_action | 额度不足后的关键承接 |
| no_token_guide_click | 点击引导弹层按钮 | action_type | `claim_gift` / `go_purchase` / `use_own_key` |
| payment_result_popup_view | 支付结果弹层展示 | pay_result, order_id | 支付反馈确认 |
| payment_result_cta_click | 支付结果弹层点击按钮 | action_type | 如继续使用、查看订单 |
| exception_popup_view | 展示异常问题引导弹层 | exception_scene, resolve_method | 一期异常统一收口 |
| exception_popup_cta_click | 点击异常引导弹层按钮 | exception_scene, action_type, resolve_method | 如重试、重下、去配置、去充值 |
| support_entry_exposure | 展示联系客服入口 | exception_scene | 当前可先预留埋点 |
| support_entry_click | 点击联系客服入口 | exception_scene, source_module | 后续客服链路的重要来源 |

---

## 6. 后端权威事件

以下事件建议以后端为主产出，避免前端口径不准：

| 事件名 | 触发时机 | 关键属性 | 说明 |
|---|---|---|---|
| usage_request_success | 一次模型请求成功完成 | tool_name, model_id, supplier_id, billing_mode, request_count, token_in, token_out, cash_cost | 核心消费事件 |
| usage_request_fail | 一次模型请求失败 | tool_name, model_id, supplier_id, fail_reason | 判断使用受阻原因 |
| usage_request_blocked_insufficient_subscription | 请求因订阅次数不足被拦截 | tool_name, model_id | 用于看订阅耗尽后回流购买 |
| usage_request_blocked_insufficient_balance | 请求因余额不足被拦截 | tool_name, model_id | 同上 |
| usage_request_fallback_to_balance | 订阅不足后自动切到余额扣费 | model_id, cash_cost | 验证扣费优先级是否命中 |
| billing_subscription_deduct | 成功扣减订阅次数 | model_id, deduct_multiplier, remain_subscription_count | 1x / 2x / 4x 档位 |
| billing_balance_deduct | 成功扣减余额 | model_id, cash_cost, remain_balance | 余额计费口径 |
| payment_order_create | 创建支付订单 | pay_sku_id, pay_method, order_type | 付款前关键节点 |
| payment_order_paid | 支付订单成功 | order_id, order_type, amount_paid | 成功订单口径 |
| payment_order_refund | 订单退款成功 | order_id, refund_amount | 售后口径 |
| entitlement_effective_success | 支付成功后权益生效 | order_id, order_type, effective_type | 区分订阅到账和余额到账 |
| entitlement_effective_fail | 支付成功后权益未生效 | order_id, order_type, fail_reason | 一期关键异常事件 |
| first_call_after_payment_success | 支付成功后首次调用成功 | order_id, tool_name, model_id, days_since_payment | 用于衡量充值后使用率 |
| first_call_after_payment_fail | 支付成功后首次调用失败 | order_id, tool_name, model_id, fail_reason | 区分用户未回流与回流后继续失败 |
| api_key_first_success_call | API Key 配置后首次成功调用 | supplier_id, tool_name, model_id | 判断“用得起来”是否成立 |

---

## 7. 一期核心漏斗

研发落地后，数据分析至少要能直接拉出以下漏斗：

### 7.1 登录漏斗

`auth_login_modal_view` → `auth_login_submit` → `auth_login_success`

### 7.2 外部供应商接入漏斗

`supplier_add_click` → `supplier_form_submit` → `supplier_verify_success` → `supplier_model_sync_success` → `supplier_global_route_save`

### 7.3 订阅购买漏斗

`purchase_page_view(tab=subscription)` → `subscription_plan_select` → `subscription_pay_click` → `subscription_pay_success` → `first_call_after_payment_success`

失败观察支线：

`subscription_pay_success` → `first_call_after_payment_fail`

### 7.4 充值购买漏斗

`purchase_page_view(tab=topup)` → `topup_plan_select` → `topup_pay_click` → `topup_pay_success` → `first_call_after_payment_success`

失败观察支线：

`topup_pay_success` → `first_call_after_payment_fail`

### 7.5 额度不足回流漏斗

`usage_request_blocked_insufficient_subscription` 或 `usage_request_blocked_insufficient_balance` → `purchase_page_view` → `subscription_pay_success` / `topup_pay_success` → `first_call_after_payment_success`

### 7.6 工具安装与恢复漏斗

`tool_download_click` → `tool_download_success` → `tool_install_success` → `tool_open_click` → `tool_open_success` → `usage_request_success`

异常恢复支线：

`tool_install_fail` 或 `tool_open_fail` → `exception_popup_view` → `tool_reinstall_click` → `tool_open_click`

### 7.7 API Key 可用性漏斗

`api_key_entry_exposure` → `api_key_entry_click` → `api_key_config_submit` → `api_key_config_success` → `api_key_first_success_call`

---

## 8. 一期最小可用埋点清单

如果研发资源有限，优先保证以下 20 个事件先上线：

1. `auth_login_modal_view`
2. `auth_login_submit`
3. `auth_login_success`
4. `dashboard_view`
5. `dashboard_balance_entry_click`
6. `tool_download_click`
7. `tool_open_click`
8. `tool_install_success`
9. `tool_open_success`
10. `tool_open_fail`
11. `api_key_entry_exposure`
12. `api_key_entry_click`
13. `api_key_config_success`
14. `api_key_config_fail`
15. `supplier_add_click`
16. `supplier_verify_success`
17. `subscription_pay_success`
18. `entitlement_effective_success`
19. `first_call_after_payment_success`
20. `first_call_after_payment_fail`

---

## 9. 落地注意事项

1. 支付成功、退款成功、调用成功、扣费成功，必须以后端事件为准，前端只记录点击与页面行为。
2. 同一事件的 `page_id`、`source_module`、`billing_mode` 枚举值要统一，不要各端各写一套。
3. 供应商验证成功和模型拉取成功必须拆成两个事件，否则无法判断失败卡点。
4. 订单中心的开票动作至少拆成：入口点击、提交申请、申请成功、申请失败。
5. API Key 复制是重要高价值事件，它代表用户准备在第三方工具真正使用平台能力。
6. 若客户端支持离线缓存和延迟上报，要加 `event_id` 做去重。
7. 若后续要做渠道投放归因，安装时要把渠道参数写入 `visitor_id` 关联链路。
8. 一期所有异常弹窗都要带 `exception_scene`，否则后面无法判断是安装问题、支付问题还是 API Key 问题。
9. 【联系客服】即使后续才上线，现在也建议先预留入口曝光和点击埋点，避免上线后还要改事件口径。

---

## 10. 建议的下一步

1. 产品确认事件范围与命名。
2. 前后端一起确认并固化字段枚举表、异常场景表和失败原因码表。
3. 研发按“前端行为事件 + 后端权威事件”双轨接入。
4. 数据侧按第 7 节直接建漏斗与转化报表。
