> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# mod 界面元素图库

> 查看 Claude Code mod 可以绘制的界面元素，例如文本、按钮、输入框、Markdown、代码和 diff，并附有示例代码和终端截图。

mod 通过元素来绘制界面：文本、框、按钮、输入框，以及几个可为您格式化内容的元素。本页的示例展示了绘制各元素的代码，其中大多数还附有结果在终端窗格中的截图，方便您根据外观来挑选元素。

要了解绘制的工作原理，请从[在界面中绘制](/docs/zh-CN/plugins/mods/interface)开始。有关主要 props 以及哪些应用会绘制各个元素，请参阅[元素参考](/docs/zh-CN/plugins/mods/reference#elements)。[类型声明](/docs/zh-CN/plugins/mods/create#get-the-types-for-your-build)列出了所有 props。

<h2 id="try-a-sample">
  试用示例
</h2>

本页的示例是代码片段，而非完整的 mod。每个示例都是一个元素及其内部嵌套内容的代码。

要在您自己的终端中查看示例，请按以下步骤创建一个小型 mod，并将示例粘贴进去。该 mod 会添加一个 `/gallery` 命令，用于打开一个窗格并在其中绘制示例。[窗格](/docs/zh-CN/plugins/mods/interface#pick-where-to-draw)在宽屏全屏终端中是会话记录旁边的侧边栏，在其他情况下则是输入框上方带边框的区域。

<Steps>
  <Step title="创建 mod">
    创建一个名为 `gallery` 的目录，并在其中创建 `.claude-plugin` 和 `hooks` 目录。[创建 mod](/docs/zh-CN/plugins/mods/create#write-a-mod-yourself) 介绍了这些文件。

    将清单保存为 `gallery/.claude-plugin/plugin.json`：

    ```json gallery/.claude-plugin/plugin.json theme={null}
    {
      "name": "gallery",
      "version": "0.1.0",
      "description": "Opens a pane that draws one sample",
      "author": { "name": "Your Name" }
    }
    ```

    在 `gallery/hooks/hooks.json` 中指定入口文件：

    ```json gallery/hooks/hooks.json theme={null}
    {
      "modules": ["./register.js"]
    }
    ```

    将代码保存为 `gallery/hooks/register.js`。它会添加一个打开窗格的 `/gallery` 命令，并在该窗格中绘制 `Plain text`：

    ```javascript gallery/hooks/register.js theme={null}
    // Stands in for your own callback in the samples that take one
    const noop = () => {}
    // The Select sample keeps its choice here
    let picked = 'md'

    // The Raster sample packs its cells with this function
    const DEFAULT_COLOR = 0x01000000
    function cellsOf(rows) {
      const numbers = rows.flat().flatMap(([char, color]) => [char.codePointAt(0), color, DEFAULT_COLOR])
      return new Uint8Array(Uint32Array.from(numbers).buffer).toBase64()
    }

    export function register(on) {
      on('session.start', async ($, e, next) => {
        await $.command.register({ name: 'gallery', description: 'Open the sample pane' })
        return next(e)
      })

      on('command.run', { command: 'gallery' }, async ($) => {
        await $.ui.open({ id: 'gallery', focus: true, closeOnEscape: true })
        return {}
      })

      on('ui.render', { component: 'Pane' }, async ($, e, next) => {
        if (e.requestId !== 'gallery') return next(e)
        const { Box, Text, Button, Input, Select, Link, Markdown, Code, Raster, Svg } = $.ui.resolve(e)
        // Replace the element after return with a sample
        return Text({ children: ['Plain text'] })
      })
    }
    ```
  </Step>

  <Step title="运行 mod">
    在 shell 中，从包含 `gallery` 的目录启动 Claude Code：

    ```bash theme={null}
    claude --plugin-dir ./gallery
    ```

    在 Claude Code 输入框中运行 `/gallery`。一个窗格随即打开，其中显示 `Plain text`。
  </Step>

  <Step title="替换为示例">
    从本页复制一个示例。在 `register.js` 中，将其粘贴以替换 `Text({ children: ['Plain text'] })`，使其位于 `return` 之后，然后保存文件。每次保存时 Claude Code 都会重新加载该模块，因此再次运行 `/gallery` 即可查看新示例。
  </Step>
</Steps>

<h2 id="pick-an-element">
  选择元素
</h2>

示例按您想在屏幕上显示的内容分组：

* **[显示文本](#show-text)**：`Text`、`Markdown` 和 `Link`
* **[显示代码和更改](#show-code-and-changes)**：`Code`
* **[排列元素](#arrange-elements)**：`Box`
* **[接收输入](#take-input)**：`Button`、`Input` 和 `Select`
* **[绘制图形](#draw-pictures)**：`Raster`、`Svg`、`Image` 和 `Client`

<h2 id="show-text">
  显示文本
</h2>

有三个元素可在屏幕上显示文字：`Text` 用于自定义样式，`Markdown` 用于已格式化的内容，`Link` 用于 URL。

<h3 id="text">
  `Text`
</h3>

`Text` 使用您指定的样式绘制字符串。此示例每种样式显示一行：

```javascript theme={null}
Box({
  flexDirection: 'column',
  children: [
    Text({ children: ['Plain text'] }),
    Text({ bold: true, children: ['bold'] }),
    Text({ italic: true, children: ['italic'] }),
    Text({ underline: true, children: ['underline'] }),
    Text({ strikethrough: true, children: ['strikethrough'] }),
    Text({ dimColor: true, children: ['dimColor'] }),
    Text({ inverse: true, children: ['inverse'] }),
    Text({ color: 'red', children: ["color: 'red'"] }),
    Text({ backgroundColor: 'blue', children: ["backgroundColor: 'blue'"] }),
  ],
})
```

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-text-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=90724ce3953d9347b61b6a9266fda451" className="dark:hidden" alt="一个包含九行文本的窗格，每行以其样式命名：plain、bold、italic、underline、strikethrough、灰色的 dimColor、inverse、红色的 color 以及蓝色的 backgroundColor。" width="1872" height="490" data-path="images/mods-el-text-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-text-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=790dd3465e06db0842639d0d8fcbbc96" className="hidden dark:block" alt="一个包含九行文本的窗格，每行以其样式命名：plain、bold、italic、underline、strikethrough、灰色的 dimColor、inverse、红色的 color 以及蓝色的 backgroundColor。" width="1872" height="490" data-path="images/mods-el-text-dark.png" />

`dimColor` 以灰色绘制文本。`backgroundColor` 的填充宽度仅与文本相同。

<h3 id="markdown">
  `Markdown`
</h3>

`Markdown` 按照 Claude 回复的格式来格式化文本。请通过 `text` 而非 `children` 传入内容：

```javascript theme={null}
Markdown({
  text: '## Release notes\n\nThis build has **two** fixes and one `flag`:\n\n- Faster start\n- Fewer prompts\n\n> Quoted text',
})
```

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-markdown-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=e40eacc4e3fbe06eb003f4463f552ec1" className="dark:hidden" alt="一个窗格，包含粗体标题 Release notes，随后是一个含有一个粗体词和一个彩色代码词的句子、一个两项列表，以及一段以斜体绘制且左侧带竖线的引用。" width="1872" height="452" data-path="images/mods-el-markdown-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-markdown-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=2cc342ed0c58e752c425fff3c95a33a6" className="hidden dark:block" alt="一个窗格，包含粗体标题 Release notes，随后是一个含有一个粗体词和一个彩色代码词的句子、一个两项列表，以及一段以斜体绘制且左侧带竖线的引用。" width="1872" height="452" data-path="images/mods-el-markdown-dark.png" />

标题以粗体绘制，不显示其 `#` 标记。行内代码以彩色绘制，不显示反引号。引用以斜体绘制，左侧带一条竖线。

<h3 id="link">
  `Link`
</h3>

`Link` 绘制一个标签，后跟其 URL：

```javascript theme={null}
Link({ href: 'https://code.claude.com/docs', label: 'Claude Code docs' })
```

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-link-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=8217bd2dfb561a8ba57023c6d89ea977" className="dark:hidden" alt="一个只有一行的窗格：标签 Claude Code docs，随后是灰色的 URL。" width="1872" height="186" data-path="images/mods-el-link-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-link-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=cd32f222a7d3530b070f58f5f189a1f8" className="hidden dark:block" alt="一个只有一行的窗格：标签 Claude Code docs，随后是灰色的 URL。" width="1872" height="186" data-path="images/mods-el-link-dark.png" />

终端将 URL 作为文本绘制在标签之后。点击能否打开它取决于用户的终端。

<h2 id="show-code-and-changes">
  显示代码和更改
</h2>

`Code` 使用 Claude Code 自己的语法配色绘制源代码文本或 diff。

<h3 id="code">
  `Code`
</h3>

指定 `language`，或传入 `path` 让 Claude Code 据此推断语言。使用 `startLine` 时，行号从该数字开始：

```javascript theme={null}
Code({
  language: 'javascript',
  startLine: 1,
  source: "const name = 'mods'\nconsole.log('hello ' + name)",
})
```

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-code-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=4e13f5d506d3fdcc59d6d9524f48d2a9" className="dark:hidden" alt="一个窗格，包含两行带行号、语法着色的 JavaScript 代码。" width="1872" height="224" data-path="images/mods-el-code-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-code-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=59f797371dd91b16b874a382b784b668" className="hidden dark:block" alt="一个窗格，包含两行带行号、语法着色的 JavaScript 代码。" width="1872" height="224" data-path="images/mods-el-code-dark.png" />

颜色来自用户的主题。

<h3 id="code-as-a-diff">
  以 diff 形式使用 `Code`
</h3>

使用 `format: 'diff'` 时，`source` 是一个或多个 unified diff hunk：

```javascript theme={null}
Code({
  format: 'diff',
  source: '@@ -1,3 +1,3 @@\n # Mods\n-A mod is a plugin.\n+A mod is a plugin that runs code.\n Read on.',
})
```

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-diff-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=4f12c9f7229ceccd9b4f47479a7235dd" className="dark:hidden" alt="一个包含四行 diff 的窗格。删除的行以红色底纹显示，添加的行以绿色底纹显示，每行都带有行号。在添加的行中，that runs code 这几个词的底纹更深。" width="1872" height="300" data-path="images/mods-el-diff-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-diff-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=9db96f34381bac8bebc32d41859b8808" className="hidden dark:block" alt="一个包含四行 diff 的窗格。删除的行以红色底纹显示，添加的行以绿色底纹显示，每行都带有行号。在添加的行中，that runs code 这几个词的底纹更深。" width="1872" height="300" data-path="images/mods-el-diff-dark.png" />

Claude Code 用行号代替 `@@` 行进行绘制。当删除的行与添加的行相似时，发生变化的词会以更深的底纹显示。

<h2 id="arrange-elements">
  排列元素
</h2>

<h3 id="box">
  `Box`
</h3>

`Box` 将其内部内容按行或列排布，并可绘制边框。此示例在一个带边框的框上方放置了一行文字：

```javascript theme={null}
Box({
  flexDirection: 'column',
  gap: 1,
  children: [
    Box({
      flexDirection: 'row',
      columnGap: 4,
      children: [Text({ children: ['a row'] }), Text({ children: ['of three'] }), Text({ children: ['items'] })],
    }),
    Box({
      borderStyle: 'round',
      paddingX: 1,
      children: [Text({ children: ["borderStyle: 'round'"] })],
    }),
  ],
})
```

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-box-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=cb9263bb02f5c8afb6e5b8450e0cdadc" className="dark:hidden" alt="一个窗格，包含一行三个间隔四列的词，然后是一个空行，再然后是围绕一行文本的圆角边框。边框横跨窗格的整个宽度。" width="1872" height="338" data-path="images/mods-el-box-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-box-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=dae83407af4d667a58717f68f13848e2" className="hidden dark:block" alt="一个窗格，包含一行三个间隔四列的词，然后是一个空行，再然后是围绕一行文本的圆角边框。边框横跨窗格的整个宽度。" width="1872" height="338" data-path="images/mods-el-box-dark.png" />

边框会延展至窗格的宽度。

<h2 id="take-input">
  接收输入
</h2>

`Button`、`Input` 和 `Select` 是控件：用户通过 Tab 键在它们之间移动，并操作获得焦点的那个控件。[键盘焦点和快捷键](/docs/zh-CN/plugins/mods/interface#know-which-keys-your-mod-can-receive)介绍了哪些按键会传递给它们。

使用 `focus: true` 打开窗格会让该窗格获得键盘焦点。`Input` 获得焦点后才能接收键入的字母，因此对于应在窗格打开后立即接收输入的输入框，请添加 `autoFocus: true`。

<h3 id="button">
  `Button`
</h3>

按钮会运行 `onPress`。此示例展示了默认形式、带快捷键的 `plain` 按钮，以及一个暗淡的按钮：

```javascript theme={null}
Box({
  flexDirection: 'column',
  children: [
    Button({ key: 'save', label: 'Save', onPress: noop }),
    Button({ key: 'next', label: 'Next', hotkey: 'n', plain: true, onPress: noop }),
    Button({ key: 'skip', label: 'Skip', dimColor: true, onPress: noop }),
  ],
})
```

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-button-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=8c638c15e12d9736f327dfcaa797b512" className="dark:hidden" alt="一个包含三个按钮的窗格，每行一个：带方括号的 Save，不带方括号且 n 为彩色的 n: Next，以及带方括号的灰色 Skip。" width="1872" height="262" data-path="images/mods-el-button-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-button-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=6a88a1fe18ba63b050366dcfab14be45" className="hidden dark:block" alt="一个包含三个按钮的窗格，每行一个：带方括号的 Save，不带方括号且 n 为彩色的 n: Next，以及带方括号的灰色 Skip。" width="1872" height="262" data-path="images/mods-el-button-dark.png" />

获得焦点的按钮以反色绘制。此处用户已按了两次 Tab：

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-button-plain-focused-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=a6224e58d10752a894615047d9824ad9" className="dark:hidden" alt="同样的三个按钮，其中第二个按钮 n: Next 以反色绘制。" width="1872" height="262" data-path="images/mods-el-button-plain-focused-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-button-plain-focused-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=9aeb7145615e77c84e7436180ed69e86" className="hidden dark:block" alt="同样的三个按钮，其中第二个按钮 n: Next 以反色绘制。" width="1872" height="262" data-path="images/mods-el-button-plain-focused-dark.png" />

<h3 id="input">
  `Input`
</h3>

`Input` 是单行文本输入框，用户按 Enter 时会运行 `onSubmit`：

```javascript theme={null}
Input({
  key: 'title',
  label: 'Title',
  placeholder: 'Type a title and press Enter',
  value: '',
  submitLabel: 'save',
  onSubmit: noop,
})
```

未获得焦点时，输入框显示其标签和占位文本：

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-input-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=7d580be1aaf5507032e1962b9167d725" className="dark:hidden" alt="一个只有一行的窗格：标签 Title，随后是灰色的占位文本 Type a title and press Enter。" width="1872" height="186" data-path="images/mods-el-input-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-input-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=9d254a8adc5959ba173717fb45eabd52" className="hidden dark:block" alt="一个只有一行的窗格：标签 Title，随后是灰色的占位文本 Type a title and press Enter。" width="1872" height="186" data-path="images/mods-el-input-dark.png" />

获得焦点后，标签变为粗体，出现光标，并在 `⏎` 之后显示 `submitLabel`：

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-input-focused-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=dcfcce0174480cc624d06c8521734fe7" className="dark:hidden" alt="同一个输入框，其标签为粗体，占位文本的第一个字母上有一个块状光标，并有一个回车符号，后跟单词 save。" width="1872" height="186" data-path="images/mods-el-input-focused-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-input-focused-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=2f5cf7d0e5f813983070e883009f528e" className="hidden dark:block" alt="同一个输入框，其标签为粗体，占位文本的第一个字母上有一个块状光标，并有一个回车符号，后跟单词 save。" width="1872" height="186" data-path="images/mods-el-input-focused-dark.png" />

键入内容会替换占位文本：

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-input-typed-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=4eb6d4443763a21415dad4e426bc747a" className="dark:hidden" alt="同一个输入框，其中包含已键入的字母 Rel，后跟回车符号和单词 save。" width="1872" height="186" data-path="images/mods-el-input-typed-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-input-typed-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=90fbac3c7a47ab26a4fb71401975eac6" className="hidden dark:block" alt="同一个输入框，其中包含已键入的字母 Rel，后跟回车符号和单词 save。" width="1872" height="186" data-path="images/mods-el-input-typed-dark.png" />

<h3 id="select">
  `Select`
</h3>

`Select` 让用户从多个选项中选择一个，并以该选项的 `value` 运行 `onSelect`：

```javascript theme={null}
Select({
  key: 'format',
  label: 'Format',
  value: picked,
  options: [
    { value: 'md', label: 'Markdown' },
    { value: 'html', label: 'HTML' },
    { value: 'txt', label: 'Plain text' },
  ],
  onSelect: (value) => {
    picked = value
  },
})
```

收起时，它显示其标签和当前选项：

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-select-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=4e3b18268ffd87d2cad06442e6d661a9" className="dark:hidden" alt="一个只有一行的窗格：标签 Format、当前选项 Markdown 以及一个小的向下箭头。" width="1872" height="186" data-path="images/mods-el-select-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-select-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=1e329e6aee748c2dd70e92dbe4960de0" className="hidden dark:block" alt="一个只有一行的窗格：标签 Format、当前选项 Markdown 以及一个小的向下箭头。" width="1872" height="186" data-path="images/mods-el-select-dark.png" />

展开时，它列出所有选项并标记其中一个：

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-select-moved-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=deb6ce0a49d213235feedaa79ac658c0" className="dark:hidden" alt="展开的选择器，三个选项列在标签下方。第二个选项 HTML 以反色绘制。" width="1872" height="300" data-path="images/mods-el-select-moved-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-select-moved-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=d69edeecd4f97dad4bdb39b908e3d98e" className="hidden dark:block" alt="展开的选择器，三个选项列在标签下方。第二个选项 HTML 以反色绘制。" width="1872" height="300" data-path="images/mods-el-select-moved-dark.png" />

用户选择一个选项后，列表会收起：

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-select-picked-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=f065ab5bc479868f03088dc736f17097" className="dark:hidden" alt="再次收起的选择器，当前选项显示为 HTML。" width="1872" height="186" data-path="images/mods-el-select-picked-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-select-picked-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=f084630e325825f3831219215a6a595e" className="hidden dark:block" alt="再次收起的选择器，当前选项显示为 HTML。" width="1872" height="186" data-path="images/mods-el-select-picked-dark.png" />

<h2 id="draw-pictures">
  绘制图形
</h2>

<h3 id="raster">
  `Raster`
</h3>

`Raster` 是由彩色字符单元格组成的网格，可用于热力图、迷你折线图或游戏棋盘。由终端负责绘制。此示例使用了起始模块中的 `cellsOf` 函数，该函数会将单元格打包成 `Raster` 所需的字符串。[绘制彩色单元格网格](/docs/zh-CN/plugins/mods/interface#draw-a-grid-of-colored-cells)对此进行了说明：

```javascript theme={null}
Raster({
  key: 'grid',
  columns: 3,
  rows: 2,
  cells: cellsOf([
    [['█', 0x2e7d32], ['█', 0xf9a825], ['█', 0xc62828]],
    [['█', 0x2e7d32], ['█', 0x2e7d32], ['█', 0xf9a825]],
  ]),
})
```

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-raster-light.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=5ed719481d3dd45b568bab696e57bf16" className="dark:hidden" alt="一个窗格，包含一个由彩色方块组成的小网格，共两行，每行三个：绿色、琥珀色和红色，然后是绿色、绿色和琥珀色。" width="1872" height="224" data-path="images/mods-el-raster-light.png" />

<img src="https://mintcdn.com/claude-code/EXkwf0kKMewZsyod/images/mods-el-raster-dark.png?fit=max&auto=format&n=EXkwf0kKMewZsyod&q=85&s=b381f05d4cc99e2abfacb505c77b17f5" className="hidden dark:block" alt="一个窗格，包含一个由彩色方块组成的小网格，共两行，每行三个：绿色、琥珀色和红色，然后是绿色、绿色和琥珀色。" width="1872" height="224" data-path="images/mods-el-raster-dark.png" />

`Raster` 会将每种颜色近似到一个较小的调色板，因此 `0x2e7d32` 会绘制为 `#337733`。

<h3 id="svg">
  `Svg`
</h3>

`Svg` 在 Desktop 应用中绘制 SVG 文档：

```javascript theme={null}
Svg({
  alt: 'Three bars of rising height',
  width: 120,
  height: 60,
  source:
    '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 120 60"><rect x="10" y="40" width="20" height="20" fill="#2e7d32"/><rect x="50" y="25" width="20" height="35" fill="#f9a825"/><rect x="90" y="5" width="20" height="55" fill="#c62828"/></svg>',
})
```

在终端中，仅返回 `Svg` 的窗格打开后是空的。要在那里绘制其他内容，请检查 [`e.surface`](/docs/zh-CN/plugins/mods/interface#pick-where-to-draw) 并返回不同的元素树。

<h3 id="image-and-client">
  `Image` 和 `Client`
</h3>

另有两个元素在此没有示例。`Image` 在终端中绘制 PNG 或原始像素。`Client` 是由您的另一个文件负责绘制的区域，用于动画和指针输入。[元素参考](/docs/zh-CN/plugins/mods/reference#elements)列出了它们的 props。

除非 Claude Code 检测到终端能够绘制使用 Unicode 占位符的 kitty 图形协议图像，否则用户看到的将是以暗色显示的 `Image` 的 `alt` 文本，而不是图片。请编写能够独立表达含义的 `alt` 文本。检测在启动时运行：在 kitty 0.28 或更高版本以及 Ghostty 中，一旦终端响应了 Claude Code 的图形查询，检测即会成功；在以下情况下检测会失败：

* **其他终端**：不属于上述两者的任何终端，或不响应该查询的终端。
* **tmux 和 screen**：在任何终端（包括 kitty 和 Ghostty）中运行于 tmux 或 screen 内的会话。
* **后台会话**：每个[后台会话](/docs/zh-CN/agent-view)，无论从哪个终端连接。

如果您的 mod 的用户在确实能够绘制这些占位符图像的终端中看到了暗色文本，可以将 [`CLAUDE_CODE_FORCE_TERMINAL_IMAGES`](/docs/zh-CN/env-vars) 设置为 `1`，以跳过检测。在 tmux 或 screen 中这样做并无帮助：`alt` 文本会消失，而 Claude Code 发送图片时不会为 tmux 或 screen 直通对其进行包装。

<h2 id="see-where-a-mod-can-draw">
  了解 mod 可以在哪里绘制
</h2>

这些示例都在窗格中绘制。mod 还可以在其他位置绘制，并可调用 Claude Code 为其显示内容：

* **窗格和横条**：[选择绘制位置](/docs/zh-CN/plugins/mods/interface#pick-where-to-draw)
* **Claude Code 自身的行，例如加载动画**：[更改 Claude Code 已绘制的内容](/docs/zh-CN/plugins/mods/interface#change-what-claude-code-already-draws)
* **Toast 通知、状态栏和日志行**：[在不开启轮次的情况下显示内容](/docs/zh-CN/plugins/mods/api#show-something-without-starting-a-turn)
* **问题对话框**：[暂停工具调用直到用户做出决定](/docs/zh-CN/plugins/mods/events#hold-a-tool-call-until-the-user-decides)

<h2 id="next-steps">
  后续步骤
</h2>

* [在界面中绘制](/docs/zh-CN/plugins/mods/interface)：逐步构建一个带标签页的窗格
* [测试绘制结果](/docs/zh-CN/plugins/mods/test#test-a-drawing)：在测试中按下您的按钮
* [元素参考](/docs/zh-CN/plugins/mods/reference#elements)：每个元素的主要 props 以及绘制它的应用
