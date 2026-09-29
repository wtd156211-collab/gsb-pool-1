# 多语言资源包一致性（从 0 实现）

仓库里只有本说明、`samples/**` 与 `.gitignore`：库、命令行入口与离线页面都从零写。`samples/notes.md`
是现场记录，不参与判定。

## 一、范围

交付物（Python 3.13 标准库）：

```text
python -m langcheck --packs samples/packs --out var/out
```

默认值就是上例：读 `samples/packs/` 下的语言包与元数据，按第二、三节的口径找不一致，
在 `--out` 写三个文件：问题清单 `issues.tsv`、对照矩阵 `matrix.tsv`、自包含页面 `report.html`。

做：键集合与回退解析、占位符一致性、复数类别完整性、空值与显式删除、多余键、逐行确定的输出、
离线页面。不做：不推导 CLDR 复数规则（复数类别只做结构比对，不按语言算类别）；不检查主语言自身的
文案质量（拼写、风格、空值）；不做机器翻译与术语库；不做增量缓存、多版本对比与历史记录；页面不做
编辑与回写；不引入第三方依赖、不加构建步骤、不联网。

## 二、口径与公式

**语言包**：`samples/packs/<locale>.json`，UTF-8（无 BOM），顶层是对象。文件名去掉 `.json` 就是
语言标识，须匹配 `^[a-z]{2,3}(-[A-Za-z0-9]{2,8})*$`；以 `_` 开头的文件不是语言包。

**键**：`段.段…`，每段匹配 `[a-z][A-Za-z0-9]*`；层级只用点号表达，不允许用嵌套对象。

**值**：字符串、`null`（显式删除标记），或复数对象——类别名取 `zero|one|two|few|many|other`，
每个类别的值是非空字符串。

**占位符**：`{name}`，名字匹配 `[A-Za-z_][A-Za-z0-9_]*`；`{{`、`}}` 表示字面花括号，未配对的单个
花括号按普通字符处理。占位符一致性 = 名字的集合相等，与顺序、出现次数无关。

**元数据** `_meta.json`：`{"base": "en", "fallbacks": {"de-AT": ["de"]}}`，`base` 缺省 `en`。
主语言必须完整：不允许出现 `null` 与 `""`，出现即配置错误。

**规范键集合 K**：主语言里值不是 `null` 的键，按码位升序；它既是对照矩阵的行集合，也是问题判定的范围。

**回退链**：`chain(L) = [L] + fallbacks[L] + [base]`，去重保序；回退链里不许出现主语言，
也不许出现不存在的语言。

**解析**：沿 `chain(L)` 找第一个含该键的包——值为 `null` 记 `deleted`（阻断回退），值为 `""`
记 `empty`（同样阻断回退），否则取该值，并把提供者记作来源。主语言兜底，所以每个键都有结果。

**判定强度**：错误（`E_*`）——缺键、显式删除主语言仍需要的键、类型不一致、占位符不一致、
复数类别缺失；提醒（`W_*`）——多余键、空值、复数类别多余。

## 三、判定顺序与数据结构

对每个非主语言 L、每个 `k ∈ K` 按下序判定，命中一条就停（`deleted`、`empty`、缺键都不再做后续比对）：

1. `deleted` → `E_DELETED`；`empty` → `W_EMPTY_VALUE`；来源是主语言 → `E_MISSING_KEY`。
2. 与主语言的值类型不一致（字符串 ↔ 复数对象）→ `E_TYPE_MISMATCH`。
3. 两边都是字符串：占位符集合不等 → `E_PLACEHOLDER_MISMATCH`。
4. 两边都是复数对象：主语言有而本语言没有的类别逐个报 `E_PLURAL_MISSING`；本语言多出的类别逐个报
   `W_PLURAL_EXTRA`；两边都有的类别再按第 3 步比对一次（结果里带类别）。
5. 多余键只看 L 自身文件：键不在 K 中且值不是 `null` → `W_EXTRA_KEY`；值 `null` 表示有意清理，不报。

继承来的问题会在继承者身上重复出现；主语言自身不参与判定。内部只需要规范键表、
每个（语言，键）的解析结果（来源、状态、值）与问题表。

## 四、输入输出与文件格式

本节文件一律 UTF-8、LF、无 BOM，末行有换行。退出码：0 = 无错误（可有提醒），1 = 有错误，
2 = 用法或配置错误（非法 JSON、键名或值类型不合法、主语言 `null` 或空值、回退链非法）；
2 时不写产物并把原因写到 stderr。

`issues.tsv`：表头 `locale key code detail`（制表符分隔），每个问题一行，按
（语言，键，问题码，明细）码位升序。`detail` 的写法：

```text
E_MISSING_KEY / E_DELETED / W_EMPTY_VALUE   source=<命中的语言>
E_TYPE_MISMATCH                             expected=<string|plural> actual=<string|plural>
E_PLACEHOLDER_MISMATCH                      missing=[名字,…] extra=[名字,…]（空集合写 []）
E_PLURAL_MISSING / W_PLURAL_EXTRA           category=<类别>
W_EXTRA_KEY                                 -
复数类别里的占位符问题，在最前面加 category=<类别> 和一个空格。
```

`matrix.tsv`：表头 `locale key source state value`，每个语言 × 每个 `k ∈ K` 一行，按（语言，键）
码位升序。`state` 取 `self`（自身提供）、`inherited`（继承回退语言）、`untranslated`（回退到主语言）、
`empty`、`deleted`；`source` 是命中的语言；`value` 是 JSON 编码，等价于
`json.dumps(v, ensure_ascii=False, sort_keys=True)`：字符串带引号、复数对象按键名排序、删除写
`null`、非 ASCII 原样保留，分隔符用标准库默认值。

`report.html`：单文件、无外部资源、不 fetch，双击（`file://`）即可打开；数据内嵌在页面里，
覆盖全部语言、矩阵与问题；用原生 ES module 渲染，至少能按语言筛选问题；配色与布局自定。

## 五、性能与验收口径

预算：语言 ≤ 50 个，单个包 ≤ 20 000 键，总键数 ≤ 200 000；端到端 ≤ 5 秒，峰值内存 ≤ 256 MiB。

验收：

1. 只用标准库；`python -m langcheck --packs samples/packs --out var/out` 在仓库根直接跑通。
2. `var/out/issues.tsv` 与 `var/out/matrix.tsv` 要与 `samples/expected/` 里的同名文件逐字节相同；
   样例返回退出码 1。
3. 同一输入连跑两遍逐字节相同，换 `PYTHONHASHSEED` 也一样；排序不许依赖文件系统枚举顺序。
4. `report.html` 离线打开可读，数据与两份 TSV 一致，不出现 TSV 之外的语言或问题。
5. 测试用标准库 `unittest`，只读 `samples/`。

## 六、样例说明

`samples/packs/`：`en`（主语言）、`de`、`de-AT`（回退到 `de`）、`fr`、`zh-CN`；`_meta.json` 声明
主语言与回退表。`samples/expected/`：`issues.tsv`（18 行问题 + 表头）与 `matrix.tsv`
（5 × 16 = 80 行 + 表头），逐行与下表对应。

| 包 | 造出来的场景 | 期望 |
| --- | --- | --- |
| `de` | `checkout.*` 四个键整体缺失；`errors.network` 把 `{seconds}` 写成 `{sekunden}`；`settings.language` 是空串；`nav.legacyCart` 是多余键；`orders.list.updated` 用 `null` 删除；`legacy.promo` 用 `null` 清理多余键 | 4 条 `E_MISSING_KEY`、1 条 `E_PLACEHOLDER_MISMATCH`、1 条 `W_EMPTY_VALUE`、1 条 `W_EXTRA_KEY`、1 条 `E_DELETED`；`legacy.promo` 不报 |
| `de-AT` | 只覆盖 4 个键，其余继承 `de`；`checkout.summary.*` 连 `de` 也没有 | 2 条 `E_MISSING_KEY`，另加继承来的占位符、删除、空值各 1 条 |
| `fr` | `cart.itemCount` 写成字符串；`orders.list.updated` 缺 `one`；`checkout.summary.itemCount` 多出 `many` 且 `other` 的占位符不一致；`checkout.summary.total` 占位符不一致 | 1 条 `E_TYPE_MISMATCH`、2 条 `E_PLACEHOLDER_MISMATCH`、1 条 `E_PLURAL_MISSING`、1 条 `W_PLURAL_EXTRA` |
| `zh-CN` | 完整且干净 | 无问题 |

## 七、待补的文档

模块与函数划分、内部数据结构、stderr 文案；`--help` 文案与日志格式；页面的配色、布局与交互细节；
新增语言或新增问题码的扩展流程。以上自定，口径、格式与预算不变。
