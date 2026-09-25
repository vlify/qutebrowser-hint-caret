# qutebrowser-hint-caret

[English](README.md)

用 hint 选中一个段落，光标直接落到该段段首 —— 一个 [qutebrowser](https://qutebrowser.org/)
userscript，无需修改 qutebrowser 本体。Tridactyl 式的手感：先指到你要开始的地方，再从那里选。

    按 v  →  敲 hint 标签  →  光标落在该段段首，且已处于 caret 选择态，
                             直接 hjkl 就能从此处扩选

## 为什么做这个

用鼠标选文字是一次上下文切换；而单独用 caret mode 时，光标总是从网页顶部开始，
你得一步步走到刚才读的位置。这个脚本补上这一段：对你正在读的段落打 hint，光标就在那里。

这个想法并不新：

* **VimFx** 的 caret mode 早就这么做了 ——「每个直接子节点含非空 `TextNodes` 的元素都有一个
  hint，激活 hint 就把光标放到该元素开头」（见
  [qutebrowser#1453](https://github.com/qutebrowser/qutebrowser/issues/1453) 中的记录），
* [qutebrowser#1453](https://github.com/qutebrowser/qutebrowser/issues/1453)
  (「Make caret mode more efficient by adding easymotion mode」，2016 年开至今仍 open)
  要的就是这种无鼠标的光标导航，
* [qutebrowser#5035](https://github.com/qutebrowser/qutebrowser/issues/5035)
  (「Hints + paragraphs = mouseless copy action」) 想要让 hint 作用于段落。

本脚本是其中一小块可用的实现：hint 选中一个段落，光标从该段开头开始。

## 安装

```sh
mkdir -p ~/.local/share/qutebrowser/userscripts
cp caret-anchor ~/.local/share/qutebrowser/userscripts/
chmod +x ~/.local/share/qutebrowser/userscripts/caret-anchor
```

如果你改过 `userscripts` 路径，请确认它包含 `~/.local/share/qutebrowser/userscripts`。

## 配置

在 `config.py` 中加入：

```python
c.hints.selectors['para'] = ['p', 'li', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
                             'blockquote', 'dd', 'dt', 'pre', 'td', 'th',
                             'figcaption']
config.bind('v', 'hint para userscript caret-anchor')
```

选择器列表可按需调整 —— 它决定"什么算一个段落"。

## 工作原理

qutebrowser 的 userscript 通过 `QUTE_*` 环境变量拿到被选中的元素，并可以通过 `QUTE_FIFO`
把命令发回 qutebrowser。本脚本：

1. 从 `QUTE_SELECTED_HTML` 取出元素标签名，并把标签名与被选文本**以 base64 编码**传给页面 ——
   页面文本可能含换行、引号或 `$`，内联进命令串会破坏 qutebrowser 的命令解析
   （文本会被当成命令自己执行），
2. 在 `document`、同源 iframe 以及所有**开放**的 shadow root 中重新定位该元素
   （与 qutebrowser 自己的 hint 使用同一套容器列表），并优先选择"文本包含所选项、
   且嵌套最深（文本最短）"的元素，
3. 构造一个**非空**选区，其 *focus* 端位于段落开头（选中一个字符、方向朝后），
4. 向 `QUTE_FIFO` 写命令：设置选区的 `jseval`、`mode-enter caret`、
   折叠选区的 `jseval`，以及 `selection-toggle`（回到"只移光标"模式）。

第 3 步是整个技巧的核心。`caret.js` 的 `setInitialCursor()` 会执行：

```js
const len = window.getSelection().toString().length;
if (len === 0) positionCaret();          // 光标被甩回网页顶部
...
if (len > 0) selectionState = NORMAL;    // 进入选择态
```

所以"非空选区"既能保住我们放好的光标位置，又能顺带打开选择态 —— **无需给 qutebrowser 打补丁**。
折叠选区（`len === 0`）会被当成"没有选区"，光标就会跳回文档顶部。

`Range` 对象没有方向，所以 `createRange()` + `setStart(t, 1)` + `setEnd(t, 0)`
会被规范化成折叠选区、静默失败。`Selection.setBaseAndExtent()` 是 DOM 中唯一能表达
"哪一端是 focus"的 API，这就是此处用"反向选择"的原因。

## 已知限制

在 qutebrowser 3.7.0 / QtWebEngine 6.11.2 / Qt 6.11.2 上验证。已在 Arch Wiki、ChatGPT、
Gemini 上测试通过。文本位于**开放** shadow root 内的页面（如 B 站评论区）同样可用 ——
但请留意下面的注意事项，那些页面上光标可能不可见。

* **段落是按文本内容匹配的。** 脚本用"文本包含所选内容、优先取最短者（最内层元素）"
  的方式重新定位元素。若页面存在多段完全相同的文本，可能定位到另一个。
* **只能选中 `c.hints.selectors['para']` 中列出的元素。**
* **脚本会短暂选中一个字符**（为满足 caret.js 的判据，见《工作原理》），随后立即折叠，
  所以不会有文字一直处于高亮状态。
* **只搜索开放的 shadow root 与同源 iframe** —— 这与 qutebrowser 自己的 hint 行为一致
  （`javascript/webelem.js` 中的 `find_css()`）。**封闭**的 shadow root 内的文本，
  userscript 无法访问，这是浏览器层面的限制，不是本脚本的缺陷。
* **页面 CSP 严格时光标可能不可见。** qutebrowser 通过向页面插入 `<style>` 元素来绘制光标；
  若页面的 `Content-Security-Policy` 限制了 `style-src` 且不含 `'unsafe-inline'`、
  nonce 或 hash，该样式会被拒绝（表现为 `Refused to apply inline style ...`）。
  此时**锚定、选区、光标移动全部正常**，只是那个闪烁的光标看不见 ——
  可以借助选区高亮判断当前位置。
* **在 shadow root 内部，用 `o` 反转选区时视觉效果可能不准。** 逻辑选区反转是正确的，
  但光标在屏幕上的位置可能指向别处，因为 qutebrowser 是按 shadow *host* 计算光标坐标的。
  建议改用 `hjkl` 来扩选和确认。
* 本脚本依赖 `caret.js` 当前 `setInitialCursor()` 的行为（见上文）。若上游改变
  了"如何判断已有选区"的逻辑，本脚本需要跟进更新。

## 许可证

GPL-3.0-or-later，与 qutebrowser 本体一致。
