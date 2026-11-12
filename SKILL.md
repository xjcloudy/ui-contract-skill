---
name: ui-contract
description: 使用 ui-contract 界面契约检查器的完整技能。当项目含 .ui-contract/ 或 tools/ui-contract/、用户要求跑 check/measure/pack/rules doctor、或想把 ui-contract 接入自己的项目、写自己的规则包时加载。涵盖接入流程、命令、measure 前置、结果解读、自定义规则包与常见坑。
---

# ui-contract 使用技能

ui-contract 是界面契约检查器：约束**布局、选型、用法**，不碰主题（色值字体字号数值归组件库）。规则不在代码里——引擎只实现 check，参数与决策写在规则包（pack）里，项目按需选用。

## 1. 接入自己的项目（一次性流程）

```sh
# 1) 引擎进项目（vendored 或 npm -D）
#    vendored：从 ui-contract 仓库复制 src/ bin/ 到 tools/ui-contract/，
#    scripts/sync-vendor.sh 可从引擎仓库同步并写 VENDORED.md 注明 commit
# 2) 建配置
mkdir -p .ui-contract
# 3) 写最小 config.yaml（见下）
# 4) 装规则包
node tools/ui-contract/bin/ui-contract.mjs pack add github.com/xjcloudy/ui-contract#admin-heavy@1.4.0
# 5) 存量项目先记基线
node tools/ui-contract/bin/ui-contract.mjs check --update-baseline
```

最小 `config.yaml`：

```yaml
schema: 1
extends: []            # 可继承 preset（机制保留中，一般留空）
packs:
  - admin-heavy@^1.4.0 # 装了哪些包，版本范围
overrides: {}          # 项目对 pack 参数的覆盖（见第 5 节）

paths:
  include:             # ⚠️ 扫描范围，新目录必须加进来，否则静默漏检
    - web/src/views/**
    - web/src/components/**
  exclude:
    - '**/*.spec.ts'

output: { format: pretty, fail_on: error }

measure:               # 不接 measure 可整段省略
  url: http://127.0.0.1:9180
  breakpoints: [1440, 768, 375]
  cookies_file: .ui-contract/cookies.txt
  pages:               # 每页可选 click: [文案] 走交互、faults 注入故障
    - { name: home, path: / }
    - { name: create-modal, path: /, click: [新增] }
    - name: list-loading
      path: /
      faults: [{ match: '/api/list', action: delay, delay_ms: 2500 }]
```

装完包会生成 `contract.lock`（来源/commit/摘要/安装时间）。本地改过安装目录会 `pack verify` 报 modified——要改就 fork 改名，不要手改。

## 2. 日常校验（每次界面改动）

```sh
# 完成门槛：全绿才算完
node tools/ui-contract/bin/ui-contract.mjs check       # 静态
node tools/ui-contract/bin/ui-contract.mjs measure     # 渲染后几何
node tools/ui-contract/bin/ui-contract.mjs pack verify
# 外加项目自己的 typecheck / build
```

### measure 前置条件（最容易卡住的一步）

measure 起 headless Chrome（CDP 端口 9300-9499）访问**运行中的应用**：

1. 服务必须在跑（`measure.url`）。
2. 要登录的应用先 `curl -c jar.txt` 登录；cookies 必须转成 **`name=value` 格式**（measure 不认 Netscape 格式）：
   `awk '$6=="会话cookie名"{print $6"="$7}' jar.txt > .ui-contract/cookies.txt`
3. 服务重启后 cookie 失效，重新登录。
4. 端口占用：`lsof -ti tcp:9180 | xargs -r kill`。
5. 自写截图脚本结尾要 `process.exit(0)`，否则 CDP 连接挂着不退。

## 3. 结果解读与处置

- 输出是 `found → expected` 偏差清单。处置顺序：① 改代码对齐 → ② 确认规则本身错了，找用户/团队裁决后改 pack → ③ 存量且本迭代不收敛，`check --update-baseline` 入基线并在提交信息说明。
- **warning 也要修**：基线只豁免存量，新增 warning 滞留会变成存量。
- **禁止**把新增违规写进基线；**禁止** `severity: off` 消音。
- 基线是棘轮：修掉的问题从基线消失并提示收缩。

## 4. 用自己的规则（写自己的 pack）

pack 是纯 YAML 声明（`packs/<name>/pack.yaml`），**不允许包含可执行代码**。装到 `.ui-contract/packs/<name>/`。结构六段，按需选写：

```yaml
schema: 1
name: my-pack
version: 0.1.0          # 语义化版本；改规则必须升版本
title / intent          # 一句话说这个包管什么场景

layout:                 # 数值刻度（项目认可的分布，不是抄来的）
  containers: { reading: 640, form: 720, page: 1440 }
  fields: { xs: 120, sm: 160, md: 240, lg: 560 }
  overlays: { modal: { sm: 400, md: 560, lg: 720, xl: 960 } }
  spacing: [0, 4, 8, 12, 16, 24, 32, 48, 64]
  type: { sizes: [12, 14, 16, 18, 20, 24, 30], max_heading_levels: 3 }
  table: { head_h: 44, row_py: 12, cell_px: 16 }

roles:                  # 语义 → 实现（换组件库只改这层）
  color: { primary-action: primary, muted-text: muted-foreground }
  type: { page-title: text-xl, body: text-sm }

decisions:              # 选型决策：什么时候用什么、忌什么
  - id: create-edit-surface
    when: 创建或编辑实体
    use: Modal 400/560/720/960
    avoid: Drawer
    reason: 抽屉只用于只读详情

policy:                 # 机器执行的硬约束
  import:               # 页面直引组件白名单，其余走封装
    allow: [input, button, select, textarea, label, table, tabs]
    forbidden: [dialog, sheet, card]
  arbitrary: { allow: ['minmax('] }   # 放行的任意值写法
  patterns:             # 复用形态的声明（见下）
    - id: my-form-pattern
      min_breakpoint: 768
      skeleton: [title, subtitle, "row × N", action]
      invariants: { label: {value: 160, tol: 10}, controlH: {value: 36, tol: 2}, rowGap: {value: 20, tol: 10} }
      slots: [hint, group]            # 允许自由发挥的槽位
      forbid: [stacked-label, multiple-primary-actions]
      exemplar: web/src/components/xxx/FormModal.vue
      why: 从哪张认可的截图/哪个实现提炼
  project:              # 项目级两阶段检查参数
    field_widths: { min_fields: 3, min_distinct: 2 }
    table_states: { require_empty_state: true }

checks:                 # severity 决定 fail_on 门槛（默认 error）
  spacing: { severity: error }
  import: { severity: error }

rules:                  # 自然语言规则，带来源与样例
  - id: spacing-scale
    check: spacing
    source: B           # A 外部标准 / B 成熟设计系统 / C 本项目事实 / D 模型自拟(只能 warn)
    evidence: Tailwind spacing scale
    fixtures: { bad: fixtures/spacing-bad.vue, good: fixtures/spacing-good.vue }
    status: active
```

写规则铁律：

- **每条规则必须可证伪**：fixtures 反例必报、正例必沉默，`rules doctor` 验证，不过就进不了包。
- 没有来源（evidence）的规则不要写；D 类（模型自拟）默认只能 warn。
- 数值不变量从**已批准实现的分布**推导，不是从单张截图。
- pattern 双轨核对：`pattern-shape`（check 静态读源码，扫 `**/*Modal*.vue`、`**/*Form*.vue`，查骨架/forbid/action 数）+ `pattern-geometry`（measure 读真实布局，invariants 逐条判定；`min_breakpoint` 以下跳过；行距只量相邻兄弟行）。两层读同一份 invariants。
- 改 pack 流程：改 pack.yaml → 引擎仓库提交（若 pack 随引擎仓库走）→ `sync-vendor.sh` 同步副本 → `pack add` 重装更新 lock → 跑全部门槛。

## 5. 项目级覆盖（不动 pack 的前提下微调）

`config.yaml` 的 `overrides` 按路径深合并进 pack 参数：

```yaml
overrides:
  layout:
    containers: { detail: 1120 }   # 只改这一个值，其余继承
  checks:
    arbitrary: { severity: warn }  # 放松某条检查
```

两个包写同一 token 不同值会报 `conflict` 并列出双方来源，用 overrides 解除。

## 6. 常见坑

- **`paths.include` 漏目录 = 静默漏检**。新组件目录、负向对照不生效时，先查这个。
- `check` 的文件范围 ≠ `measure.pages` 的页面范围，两处都要加新页面。
- cookies 格式、端口清理见第 2 节；故障注入 faults 的 match 只拦 `*/api/*` 类请求，别用 `*`（会拦 document/css 导致白屏）。
- 引擎版本与 vendored 副本要同步演进，`VENDORED.md` 记 commit。

## 7. 排障顺序

1. `pack verify` 确认包没被手改。
2. `rules doctor` 确认规则自身健康（正反样例、来源、停用项）。
3. `check --print-config` 看合并后的最终参数（含 overrides 是否生效）。
4. `measure` 单页单断点跑，缩小问题面。
