# 现场记录

来源：用户反馈（原始记录，未做整理）。

- 用户点了一个带两个斜杠的链接，形如 `https://cdn.example.com//assets//img/logo.png`。
- 页面里显示的地址只剩主机部分：`https://cdn.example.com`，后面的 `//assets//img/logo.png` 整段不见了。
- 换成 `https://cdn.example.com/assets/img/logo.png`（单斜杠）就正常，说明问题出在连续斜杠上。
- 同一批链接里还有只写了 `//cdn.example.com/lib/app.js` 的，没有协议前缀，行为是否一致没人确认过。
- 这类输入已经收进 `urls` 批次的样例里，口径以 README 为准。
