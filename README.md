# 地址解析与相对解析（从 0 实现）

起始环境里只有规格与样例。实现要新写：Python 3.13 标准库，单文件，不引第三方依赖、不加构建步骤、不联网。

## 1. 范围

要做：

- 把一条地址原文拆成六个字段：协议、主机、端口、路径、查询、片段，并按本文规则规范化。
- 以一条基准地址解析相对引用：父级跳转、仅查询、仅片段、绝对地址覆盖、空引用等。
- 批量跑 samples/ 下的素材，产出可逐行核对的 JSONL，以及一份数据内联的单文件 HTML 报告。

不做：

- 不解析 IPv6 字面量（方括号形式）和用户信息（@ 前缀），出现即拒绝。
- 不做 IDNA 与 Punycode、不做 DNS、不做 HTTP 客户端、不渲染页面、不联网、不读环境变量。
- 对外部文件只读 samples/，只写 var/。
- 不追求与浏览器或 RFC 3986 逐字兼容，冲突以本文为准。

## 2. 口径与公式

### 2.1 字符与百分号编码

- 每条输入先去掉首尾 ASCII 空白（空格、\t、\r、\n、\f、\v）。
- 正文里出现控制字符（U+0000–U+001F、U+007F）、空格、反斜杠、尖括号、英文双引号（U+0022）一律非法，记 illegal_char。
- 百分号后面必须跟两位十六进制数字，否则非法，记 bad_percent_encoding。
- 六个字段同一口径：百分号三元组里编码的是 unreserved（A-Z a-z 0-9 - . _ ~）就还原成原字符，其余三元组原样保留，并把十六进制字母规范成大写，例如 %2f 保留为 %2F、%41 还原为 A。

### 2.2 规范化

- 大小写：协议与主机小写；路径、查询、片段保持原样。
- 默认端口：http 80、https 443、ws 80、wss 443、ftp 21。端口等于所属协议的默认值就记 null；没有协议时不套默认端口。
- 端口：只允许十进制数字，允许前导零（0080 与 80 等价），取值 1–65535，0 和越界都非法。
- 点段折叠：路径以 / 开头时按 RFC 3986 第 5.2.4 节去掉 . 段与 .. 段；不以 / 开头时原样保留。
- 顺序：先还原百分号，再折叠点段，所以 %2E 会被当成点段：/a/%2E%2E/b 折叠为 /b。
- 空串与缺失不同：有 authority 但没写路径记空串，不补 /；查询与片段要区分不存在（null）与存在但为空（空串），例如 http://a/? 的查询是空串，而 http://a/ 的查询是 null。

### 2.3 相对解析（基准必须带协议）

按引用原文的形态选分支，结果同样填六个字段：

- 空引用：六个字段全部取自基准，片段记 null，基准自己的片段被丢弃。
- 仅片段（以 # 开头）：其余字段取自基准，片段取引用。
- 仅查询（以 ? 开头）：路径等取自基准，查询取引用（单独一个 ? 就是空查询），片段取引用或 null。
- 带协议：整条引用按它自己解析，基准不参与。
- 以 // 开头：协议取自基准，主机、端口、路径、查询、片段取自引用，并按基准协议重新套默认端口规则。
- 其余：路径合并为「基准路径最后一个 / 之前的部分」加「引用路径」，引用路径以 / 开头就直接用它，合并结果再折叠点段；查询在引用写了 ? 时取引用的（含空查询），否则继承基准；片段取引用或 null。
- 基准不可解析或没有协议记 invalid_base；基准合法、引用非法时报引用自己的原因。

## 3. 状态机与数据结构

流水线，任一步不通过就停在该步并给出原因，顺序即优先级：

修剪 → 字符校验 → 切片段（#）→ 切查询（?）→ 切协议（形如 scheme: 才认）→ 切 authority 与路径（协议后紧跟 // 才算 authority，其余整段是路径）→ 主机（先判空，再看字符集）→ 端口 → 路径 → 查询 → 片段 → 小写化与点段折叠 → 组装。

字段：scheme、host、port、path、query、fragment。path 恒为字符串，其余可为 null。失败原因只有这些取值：illegal_char、bad_percent_encoding、missing_host、bad_host_char、empty_port、non_digit_port、port_out_of_range、unsupported_userinfo、unsupported_ipv6、invalid_base，相对批次另有 bad_line。authority 为空（http://、http:///x、//、//:8080/x）记 missing_host，且优先于端口类原因；主机允许的字符是字母、数字、- . _ ~ %，其余记 bad_host_char。

## 4. 输入输出与文件格式

- samples/urls/*.txt：每行一条地址原文，整行空白则忽略，行序即记录序。
- samples/relative/*.txt：每行用第一个制表符分成两段，左边是基准、右边是引用，两侧各自去掉首尾空白，右边可以为空串（空引用）；整行空白则忽略；没有制表符的行记 bad_line（base 写整行原样，ref 为 null）。
- 处理顺序：先 urls 后 relative；每批内部按文件名字典序，文件内部按行序。
- 新写的入口是仓库根目录的 solution.py。
- 输出 JSONL：程序写 var/urls.jsonl 与 var/relative.jsonl，每行一个对象，键顺序严格是 input、ok、scheme、host、port、path、query、fragment，失败行只有 input、ok、error；relative 批次用 base、ref 两个键替换 input（base 在前）。序列化固定用 json.dumps(obj, ensure_ascii=False, separators=(',', ':'))，UTF-8，LF 换行，文件末尾一个换行。
- 成功行 ok 为 true，六个字段照写，缺值写 null；失败行 ok 为 false，error 取第 3 节的取值。input、base、ref 写去掉首尾空白之后的文本。
- var/report.html：单文件 HTML5，无外部资源（不出现 link、img、iframe、script src、@import、url(...)），样式内联，两批各一张表，每条记录一行 tr，失败行要能看出 error。
- 命令行：python solution.py 跑样例并写 var/ 下三个文件（目录不存在就创建）；python solution.py --urls 文件 和 python solution.py --relative 文件 只处理一个文件，把 JSONL 写到标准输出。

## 5. 性能与验收

- 批量预算：--urls 模式处理 50000 行（每行不超过 2 KiB）单线程端到端不超过 3 秒，峰值内存不超过 256 MiB。
- 验收：
  1. var/urls.jsonl、var/relative.jsonl 与 samples/expected/ 下的同名文件逐字节相同，顺序敏感。
  2. var/report.html 存在、单文件、内联全部记录。
  3. 只用标准库；不联网；无第三方依赖与构建步骤。
  4. 非法输入必须按第 3 节的取值拒绝，不能静默接受或猜测修复。

## 6. 样例说明

- urls 批次：01-basic 常规绝对地址；02-double-slash 双斜杠与空路径；03-dot-segments 点段与 %2E；04-encoded 编码字符与十六进制大小写；05-case 协议与主机大小写混写；06-missing-host 缺主机；07-port 端口与默认端口；08-query-fragment 查询与片段的取舍；09-reject 各类拒绝原因。
- relative 批次：01-parent 父级跳转；02-query-only 仅查询；03-fragment-only 仅片段；04-absolute 绝对地址覆盖与 // 引用；05-empty 空引用；06-reject 非法基准、非法引用与坏行。
- samples/expected/urls.jsonl 与 samples/expected/relative.jsonl 是逐行确定的期望结果，与输入一一对应，行数相同。
- samples/notes.md 是现场记录。

## 7. 待补的文档

- IPv6 字面量、用户信息、IDNA 与非 ASCII 主机、更多协议的默认端口暂不在范围内；要扩展先改本文，再补 samples 与期望结果。
