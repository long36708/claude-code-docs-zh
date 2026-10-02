> ## Documentation Index
> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# 使用 mod 在界面中绘制

> 从 Claude Code mod 中绘制窗格、提示符上方的条带、按钮和文本字段，处理按键和输入，并在重绘和会话之间保持状态。

mod 可以在 Claude Code 中绘制自己的界面，并更改 Claude Code 已经绘制的界面部分。mod 可以绘制的每个位置称为[渲染站点](/docs/zh-CN/plugins/mods/reference#render-sites)，例如窗格、提示符上方的条带或加载指示器。Claude Code 在即将绘制渲染站点时会触发 [`ui.render`](/docs/zh-CN/plugins/mods/reference#interface) 事件，你的该事件钩子返回在那里绘制的内容。

此地图显示 mod 可以在终端会话中的绘制位置：

<img src="https://mintcdn.com/claude-code/dgiVO_Od1X1faduV/images/mods-screen-map.svg?fit=max&auto=format&n=dgiVO_Od1X1faduV&q=85&s=5fda26b6609c62b68c6f9e528c1590ea" className="dark:hidden" alt="Claude Code 终端会话的地图。mod 可以在右侧添加窗格作为侧边栏，在记录的右上角添加 toast，在记录中添加日志行，在提示符上方添加条带，以及在提示符下方添加状态行。mod 可以重绘消息、工具调用行和加载指示器。提示符是 Claude Code 自己的。" width="600" height="336" data-path="images/mods-screen-map.svg" />

<img src="https://mintcdn.com/claude-code/dgiVO_Od1X1faduV/images/mods-screen-map-dark.svg?fit=max&auto=format&n=dgiVO_Od1X1faduV&q=85&s=5b4161581a1bd2c0450b0c8b57bc1225" className="hidden dark:block" alt="Claude Code 终端会话的地图。mod 可以在右侧添加窗格作为侧边栏，在记录的右上角添加 toast，在记录中添加日志行，在提示符上方添加条带，以及在提示符下方添加状态行。mod 可以重绘消息、工具调用行和加载指示器。提示符是 Claude Code 自己的。" width="600" height="336" data-path="images/mods-screen-map-dark.svg" />

在较窄的终端中，窗格位于提示符上方而不是记录旁边。

在开始之前，请构建你的[第一个 mod](/docs/zh-CN/plugins/mods/create)。从工作示例开始，该示例构建一个具有两个选项卡和计数器的窗格，然后阅读你想要更改的每个部分的部分。

<Note>
  要查找一个属性或限制，请参阅[参考](/docs/zh-CN/plugins/mods/reference#render-sites)。
</Note>

<h2 id="build-a-pane-with-tabs">
  构建带有选项卡的窗格
</h2>

在本部分中，你将构建一个 mod，该 mod 添加 `/hello-tabs` 命令，该命令打开一个窗格。窗格是在宽全屏终端中记录旁边的侧边栏，或在其他情况下是提示符上方的框架区域。此窗格显示两个选项卡，第二个选项卡有一个按钮，可以将计数器加一。重新启动 Claude Code 后，计数仍然存在。

完成的 mod 看起来像这样。录制打开窗格，切换到第二个选项卡，按几次按钮，然后返回到第一个选项卡：

<Frame>
  <video autoPlay muted loop playsInline controls className="w-full dark:hidden" src="https://mintcdn.com/claude-code/dgiVO_Od1X1faduV/images/mods-hello-tabs-light.mp4?fit=max&auto=format&n=dgiVO_Od1X1faduV&q=85&s=49d520094d87b5b44bfe50fa49677f06" aria-label="在 Claude Code 提示符处键入 /hello-tabs 命令，一个框架窗格在其上方打开，顶部显示&#x22;1: One&#x22;和&#x22;2: Two&#x22;，文本为&#x22;This is the first tab.&#x22;。第二个选项卡显示&#x22;Add one&#x22;按钮，旁边是&#x22;Count: 1&#x22;，计数上升到 3。窗格然后返回到第一个选项卡。" data-path="images/mods-hello-tabs-light.mp4" />

  <video autoPlay muted loop playsInline controls className="w-full hidden dark:block" src="https://mintcdn.com/claude-code/dgiVO_Od1X1faduV/images/mods-hello-tabs-dark.mp4?fit=max&auto=format&n=dgiVO_Od1X1faduV&q=85&s=ff7a14d713d6e5d3b0000efa8522ea4b" aria-label="在 Claude Code 提示符处键入 /hello-tabs 命令，一个框架窗格在其上方打开，顶部显示&#x22;1: One&#x22;和&#x22;2: Two&#x22;，文本为&#x22;This is the first tab.&#x22;。第二个选项卡显示&#x22;Add one&#x22;按钮，旁边是&#x22;Count: 1&#x22;，计数上升到 3。窗格然后返回到第一个选项卡。" data-path="images/mods-hello-tabs-dark.mp4" />
</Frame>

Claude Code 没有内置的 tabs 元素，所以选项卡是一行中的两个按钮。mod 跟踪哪一个是活动的，并在该行下方绘制该选项卡的内容。

<Steps>
  <Step title="创建插件">
    mod 是一个具有清单、指向你的代码的 `hooks.json` 和代码文件的插件。[创建 mod](/docs/zh-CN/plugins/mods/create#write-a-mod-yourself) 解释了每一个。创建一个名为 `hello-tabs` 的目录，其中包含 `.claude-plugin` 和 `hooks` 目录，然后保存前两个文件。

    将清单保存为 `hello-tabs/.claude-plugin/plugin.json`：

    ```json hello-tabs/.claude-plugin/plugin.json theme={null}
    {
      "name": "hello-tabs",
      "version": "0.1.0",
      "description": "Opens a pane with two tabs and a counter",
      "author": { "name": "Your Name" }
    }
    ```

    在 `hello-tabs/hooks/hooks.json` 中命名你的入口点：

    ```json hello-tabs/hooks/hooks.json theme={null}
    {
      "modules": ["./register.js"]
    }
    ```
  </Step>

  <Step title="编写代码">
    代码执行三个任务，每个钩子一个：

    * 添加 `/hello-tabs` 命令
    * 运行该命令时打开窗格
    * 绘制窗格的内容：选项卡行和打开的选项卡的主体

    两个模块级变量 `tab` 和 `count` 保存窗格的状态。

    将其保存为 `hello-tabs/hooks/register.js`：

    ```javascript hello-tabs/hooks/register.js theme={null}
    // 窗格的 id，用于打开窗格和在绘制时识别它
    const PANE = 'hello-tabs'

    // 窗格显示的内容：哪个选项卡是打开的，以及计数器的值
    let tab = 'one'
    let count = 0

    export function register(on) {
      // 在你的第一个提示符之前运行，以及重新加载后再次运行
      on('session.start', async ($, e, next) => {
        await $.command.register({ name: 'hello-tabs', description: 'Open the hello-tabs pane' })
        // 加载早期会话保存的计数（如果有的话）
        const saved = await $.store.get('count')
        if (typeof saved === 'number') count = saved
        return next(e)
      })

      // 当你键入 /hello-tabs 时运行
      on('command.run', { command: 'hello-tabs' }, async ($) => {
        // 打开窗格，给它键盘焦点，让 Esc 关闭它
        await $.ui.open({ id: PANE, title: 'Hello tabs', focus: true, closeOnEscape: true })
        // 在记录中不打印任何内容
        return {}
      })

      // 每次 Claude Code 绘制窗格时运行
      on('ui.render', { component: 'Pane' }, async ($, e, next) => {
        // 不理其他 mod 的窗格
        if (e.requestId !== PANE) return next(e)
        // 获取此应用可以绘制的元素
        const { Box, Text, Button } = $.ui.resolve(e)
        // 要求 Claude Code 再次运行此钩子
        const redraw = () => $.ui.invalidate('ui.render')

        // 一个选项卡：一个按钮，按下时切换到其选项卡
        const tabButton = (name, label, hotkey) =>
          Button({
            key: 'tab-' + name,
            label,
            hotkey,
            plain: true,
            // 使不是打开的选项卡变暗
            dimColor: tab !== name,
            onPress: () => {
              tab = name
              redraw()
            },
          })

        // 根据打开的选项卡，在选项卡下方显示的内容
        const body =
          tab === 'one'
            ? [Text({ children: ['This is the first tab.'] })]
            : [
                Box({
                  flexDirection: 'row',
                  columnGap: 2,
                  children: [
                    Button({
                      key: 'more',
                      label: 'Add one',
                      hotkey: 'a',
                      onPress: async () => {
                        count += 1
                        redraw()
                        // 保存计数，以便在重新启动后仍然存在
                        await $.store.set('count', count)
                      },
                    }),
                    Text({ children: ['Count: ' + count] }),
                  ],
                }),
              ]

        // 整个窗格：选项卡行、空白行，然后是主体
        return Box({
          flexDirection: 'column',
          children: [
            Box({
              flexDirection: 'row',
              columnGap: 3,
              children: [tabButton('one', 'One', '1'), tabButton('two', 'Two', '2')],
            }),
            Text({ children: [' '] }),
            ...body,
          ],
        })
      })
    }
    ```

    每个钩子也做代码没有明确说明的事情：

    * **[`session.start`](/docs/zh-CN/plugins/mods/reference#session)** 也从 [`$.store`](#keep-state) 读取保存的计数，这是一个在会话之间持久化的键值存储。
    * **[`command.run`](/docs/zh-CN/plugins/mods/api#add-a-command)** 只告诉 Claude Code 窗格存在。打开窗格本身不绘制任何内容：Claude Code 然后触发 `ui.render` 来询问在其中放入什么。
    * **`ui.render`** 返回元素树，一个 `Box`，它保存其他框、文本和按钮，并从 `tab` 和 `count` 每次运行时重新构建它。

    按下按钮会运行其 `onPress` 回调，该回调更改变量并调用 `redraw`。Claude Code 然后再次运行 `ui.render` 钩子，该钩子从新值构建新树。每个交互式绘制都使用该渲染周期：回调更改状态，钩子从新状态重新渲染。
  </Step>

  <Step title="打开窗格">
    在你的 shell 中，使用 `claude --plugin-dir ./hello-tabs` 启动 Claude Code。在 Claude Code 提示符处，运行 `/hello-tabs`。一个窗格打开，顶部显示 `1: One` 和 `2: Two`。按 `2`，然后按 `a`，**Add one** 的快捷键，几次。计数上升。
  </Step>

  <Step title="检查计数是否已保存">
    按 Esc 关闭窗格，然后退出会话。在你的 shell 中，使用相同的 `claude --plugin-dir ./hello-tabs` 命令再次启动 Claude Code，在 Claude Code 提示符处运行 `/hello-tabs`。计数在你离开的地方。

    要清除计数，让 mod 调用 `$.store.delete('count')`。[保持状态](#keep-state) 涵盖每种值持续多长时间。
  </Step>
</Steps>

<h2 id="pick-where-to-draw">
  选择绘制位置
</h2>

`ui.render` 钩子为每个渲染站点运行，除非你将其缩小到你想要绘制的站点。要选择渲染站点，请将称为[匹配器](/docs/zh-CN/plugins/mods/events#filter-which-events-a-hook-handles)的过滤器作为第二个参数传递给 `on`。`{ component: 'Pane' }` 仅为窗格运行钩子。在钩子中，`e.component` 命名站点，`e.surface` 说明哪个应用在绘制，`e.props` 保存站点自己的数据。对于窗格，`e.requestId` 是你用来打开它的 `id`。

两个站点是空的，直到 mod 填充它们，窗格和条带。选择一个选项卡以查看每个是什么以及如何在其中绘制：

<Tabs>
  <Tab title="Pane">
    窗格是在宽全屏终端中记录旁边的侧边栏，或在其他情况下是提示符上方的框架区域。打开多个窗格时，每个窗格都会获得一个显示其标题的选项卡。

    当你的 mod 使用你选择的 `id` 调用 `$.ui.open` 时，窗格出现，如 `$.ui.open({ id: 'hello-tabs' })`。[在正确的时间打开窗格](#open-a-pane-at-the-right-time) 涵盖其他字段以及窗格何时等待更宽的终端。

    要在你的窗格中绘制，请过滤 `{ component: 'Pane' }` 并检查 `e.requestId` 是否是你的 `id`。
  </Tab>

  <Tab title="Band above the prompt">
    条带是直接在提示符输入上方的条纹。它始终存在，每个 mod 都共享它。

    你的钩子返回一棵树以在条带中显示某些内容，或返回 `next(e)` 以不显示任何内容。一棵树替换 mod [在你之后](/docs/zh-CN/plugins/mods/events#the-order-mods-run-in) 在那里绘制的内容。要保留他们的，将 `await next(e)` 的结果放在你的树中的 [`Box`](#build-a-tree-from-elements) 的子项中。

    要在条带中绘制，请过滤 `{ component: 'AbovePrompt' }`。
  </Tab>
</Tabs>

<h3 id="change-what-claude-code-already-draws">
  更改 Claude Code 已经绘制的内容
</h3>

Claude Code 自己绘制大部分界面：消息、工具调用行、加载指示器等。这些部分中的每一个也是一个渲染站点，所以 mod 可以重新设置样式或替换它。要更改一个，请在你的 `ui.render` 钩子上过滤此表中的其名称：

| 站点 | 它是什么 |
| :- | :- |
| `UserMessage`, `AssistantMessage` | 记录中的消息 |
| `ToolUse`, `ToolResult`, `ToolGroup` | 工具调用的行、其结果和折叠的调用运行 |
| `CommandOutput` | 命令打印的行 |
| `AskUserQuestion` | Claude 打开的对话框以询问你一个问题 |
| `Spinner`, `ToolProgress`, `TurnDuration` | 轮次的状态行：在 Claude 工作时动画的行、运行工具的实时进度行以及关闭轮次的行 |
| `InfoNotice`, `SessionMode`, `PromptHint` | 徽标下的状态行、页脚中的模式标签以及提示符下的提示行 |

在 Claude Code 已经绘制的站点，你的钩子有三个选择：更改详细信息、替换绘制或不理它。选择一个选项卡以查看每一个应用于加载指示器。示例读取另一个钩子计数的 `calls` 变量，如[教程 mod](/docs/zh-CN/plugins/mods/create#write-a-mod-yourself) 中所示。

<Tabs>
  <Tab title="Change a detail">
    要保留 Claude Code 的绘制并更改其一部分，请将 `next` 传递给更改了 `props` 的事件副本。此钩子更改加载指示器单词后的文本：

    ```javascript theme={null}
    on('ui.render', { component: 'Spinner' }, async ($, e, next) => {
      // 保留 Claude Code 的加载指示器，并更改其单词后的文本
      return next({ ...e, props: { ...e.props, suffix: ' · tool calls: ' + calls + '…' } })
    })
    ```

    加载指示器保留其动画和单词，你的文本跟在单词后面：

    ```text theme={null}
    Thinking · tool calls: 2…
    ```
  </Tab>

  <Tab title="Replace the drawing">
    要在站点的位置绘制你自己的内容，请返回一棵树，不要调用 `next`。此钩子在加载指示器所在的位置绘制一行文本：

    ```javascript theme={null}
    on('ui.render', { component: 'Spinner' }, async ($, e) => {
      const { Text } = $.ui.resolve(e)
      // 没有对 next 的调用，所以这一行在加载指示器的位置绘制
      return Text({ children: ['Claude has made ' + calls + ' tool calls'] })
    })
    ```

    当 Claude 工作时，你的行显示，Claude Code 的加载指示器不显示：

    ```text theme={null}
    Claude has made 2 tool calls
    ```
  </Tab>

  <Tab title="Leave it alone">
    要将站点保留为 Claude Code 绘制的方式，请返回 `next(e)`。钩子通常对某些事件这样做，对其他事件不这样做。此钩子在有要计数的调用之前保留加载指示器：

    ```javascript theme={null}
    on('ui.render', { component: 'Spinner' }, async ($, e, next) => {
      // 还没有什么要显示的，所以不变地传递事件
      if (calls === 0) return next(e)
      return next({ ...e, props: { ...e.props, suffix: ' · tool calls: ' + calls + '…' } })
    })
    ```

    在第一个工具调用之前，加载指示器看起来就像没有 mod 的样子：

    ```text theme={null}
    Thinking…
    ```
  </Tab>
</Tabs>

权限提示不是渲染站点，所以 mod 无法更改它显示的内容。问题对话框 `AskUserQuestion` 是一个，所以 mod 可以更改它。

终端和桌面应用不会触发所有相同的站点。`Pane`、`AbovePrompt`、`Spinner` 和记录站点在两者中都有效。其他一些状态行仅在终端中触发。[渲染站点表](/docs/zh-CN/plugins/mods/reference#render-sites) 列出了每个站点在哪里触发。

<h3 id="open-a-pane-at-the-right-time">
  在正确的时间打开窗格
</h3>

窗格仅在你的 mod 打开它时出现。你如何以及何时打开它决定了它是否获得键盘焦点、它要求多少空间，以及它是否在狭窄的终端中显示。

要打开窗格，请使用你选择的 `id` 调用 [`$.ui.open`](/docs/zh-CN/plugins/mods/reference#mods-api-methods)。`id` 是窗格的名称：你的 `ui.render` 钩子检查它，你再次传递它来关闭窗格。

```javascript theme={null}
await $.ui.open({ id: 'hello-tabs', title: 'Hello tabs', focus: true })
```

要关闭窗格，请使用你打开它的 `id` 调用 `$.ui.close`：

```javascript theme={null}
await $.ui.close({ id: 'hello-tabs' })
```

除了 `id`，`$.ui.open` 还接受这些可选字段：

| 字段 | 它做什么 |
| :- | :- |
| `title` | 打开多个窗格时窗格的选项卡标签 |
| `focus` | 请求[键盘焦点](#know-which-keys-your-mod-can-receive) |
| `closeOnEscape` | 使 Esc 关闭窗格 |
| `holdToasts` | 保持 toast，来自 [`$.ui.toast`](/docs/zh-CN/plugins/mods/api#show-something-without-starting-a-turn) 的小通知，直到窗格关闭 |
| `rows` | 当窗格位于提示符上方时要求的高度。默认值是空间的三分之一。 |
| `columns` | 当窗格位于记录旁边时要求的宽度 |

`focus`、`closeOnEscape` 和 `holdToasts` 是可选的，仅接受 `true`。要省略其中一个，请忽略它。传递 `false` 会抛出错误，例如 `ui.open: focus is true or left out`。要有条件地设置其中一个，仅在条件成立时添加字段。此调用仅在 `items` 不为空时请求键盘焦点：

```javascript theme={null}
const pane = { id: 'hello-tabs', title: 'Hello tabs' }
await $.ui.open(items.length > 0 ? { ...pane, focus: true } : pane)
```

要让命令在 Claude 工作时打开窗格，请在[注册命令](/docs/zh-CN/plugins/mods/api#add-a-command)时添加 `immediate: true`。没有它，在轮次期间键入的命令会等待轮次结束。

<h4 id="when-a-pane-waits-for-a-wider-terminal">
  当窗格等待更宽的终端时
</h4>

你的 mod 打开的窗格而不被要求不会在狭窄的终端中出现，所以它无法接管小屏幕。它是否出现取决于打开它的内容：

* **由用户做的事情打开**，例如他们运行的命令或他们按下的按钮，窗格在任何宽度出现
* **由你的 mod 自己打开**，例如从计时器或 [`turn.start`](/docs/zh-CN/plugins/mods/events#follow-a-turn) 钩子，窗格仅在至少 144 列宽的终端中出现。用户自己打开该窗格一次后，110 列就足够了。

当窗格出现时，`$.ui.open` 解析为 `{ isPlaced: true }`。当窗格在等待时，`isPlaced` 是 `false`，`reason` 是一个说明原因的字符串。等待的窗格在用户打开它或拓宽终端时出现。要说某些内容可用而不打开窗格，请调用 `$.ui.toast('Your message')`，它显示一个在几秒后消失的小通知。

<h2 id="build-a-tree-from-elements">
  从元素构建树
</h2>

`ui.render` 钩子返回的是一个元素树：对要绘制的内容的描述，由相互嵌套的框、文本和控件组成。你描述绘制，Claude Code 在终端或桌面应用中呈现它。

要获取元素，请在你的钩子中调用 `$.ui.resolve(e)`，如 `const { Box, Text, Button } = $.ui.resolve(e)`。每个元素都是一个函数。你传递它属性，你把在其中的元素和字符串放在 `children` 中。

大多数绘制使用四个元素。选择一个选项卡以查看每一个以及终端如何绘制它：

<Tabs>
  <Tab title="Text">
    `Text` 绘制一个字符串，带有可选的样式，如 `bold` 和 `color`：

    ```javascript theme={null}
    Text({ children: ['This is the first tab.'] })
    ```

    ```text theme={null}
    This is the first tab.
    ```
  </Tab>

  <Tab title="Box">
    `Box` 排列其中的内容，在一行或一列中。这个把一个按钮和一行文本并排放在一起，相隔两列：

    ```javascript theme={null}
    Box({
      flexDirection: 'row',
      columnGap: 2,
      children: [
        Button({ key: 'more', label: 'Add one', onPress: addOne }),
        Text({ children: ['Count: 0'] }),
      ],
    })
    ```

    ```text theme={null}
    [ Add one ]  Count: 0
    ```
  </Tab>

  <Tab title="Button">
    `Button` 是用户可以按下的控件。它运行你的 `onPress` 回调。使用 `plain: true` 它没有括号并显示其快捷键：

    ```javascript theme={null}
    Button({ key: 'more', label: 'Add one', onPress: addOne })
    Button({ key: 'tab-one', label: 'One', hotkey: '1', plain: true, onPress: showTabOne })
    ```

    ```text theme={null}
    [ Add one ]
    1: One
    ```
  </Tab>

  <Tab title="Input">
    `Input` 是一个文本字段。当用户按 Enter 时，它使用文本运行你的 `onSubmit` 回调：

    ```javascript theme={null}
    Input({
      key: 'new-note',
      label: 'Note',
      placeholder: 'Type a note and press Enter',
      value: '',
      submitLabel: 'add',
      onSubmit: addNote,
    })
    ```

    ```text theme={null}
    Note: Type a note and press Enter
    ```
  </Tab>
</Tabs>

[界面图库](/docs/zh-CN/plugins/mods/gallery)提供了大多数元素的示例和屏幕截图。此表列出了每个元素：

| 元素 | 它绘制什么 | 位置 |
| :- | :- | :- |
| `Box` | 一个 flex 容器。接受布局属性，如 `flexDirection`、`columnGap`、`padding`、`borderStyle` 和 `width`。 | 到处 |
| `Text` | 样式化文本。接受 `color`、`bold`、`dimColor`、`italic` 和 `wrap`。`color` 是主题键或颜色，如 `'red'`。`wrap` 是 `'wrap'`、`'truncate'`、`'truncate-start'`、`'truncate-middle'` 或 `'truncate-end'`。 | 到处 |
| `Button` | 调用 `onPress` 的控件 | 到处 |
| `Link`, `Code`, `Markdown` | 带有 `href` 和可选 `label` 的链接、代码块和格式化为 Claude 回复方式的文本。`Markdown` 在 `text` 属性中而不是在 `children` 中获取其内容，当你传递 `onLinkPress` 时需要 `key`。 | 到处 |
| `Input`, `Select` | 文本字段和选择器 | 终端、桌面 |
| `Svg` | SVG 文档 | 桌面 |
| `Client` | 由你的第二个文件绘制的区域，用于动画和指针输入。该文件没有 mod API。它仅通过发布数据到达你的钩子，该数据作为 `ui.message` 事件到达。 | 终端、桌面 |
| `Raster`, `Image` | [彩色单元格网格](#draw-a-grid-of-colored-cells)和图片 | 终端 |

如果你的模块是 `.tsx` 或 `.jsx` 文件，你可以将树写成 JSX。首先从 `$.ui.resolve(e)` 解构元素，因为钩子模块没有元素全局。

如果树使用应用没有的元素、元素不接受的属性或没有子项的位置，Claude Code 绘制其自己的站点版本。

在使用 `--plugin-dir` 启动的会话中，记录行说明这一点，例如 `ui.render (Pane) refused: Text prop "bogusProp" is not allowed; the engine drew its own`。[调试日志](/docs/zh-CN/plugins/mods/troubleshoot#read-the-debug-log) 将其记录为 `ui.render (Pane): a hook returned a tree that does not validate` 并带有相同的原因。会话中没有其他内容出现，所以当绘制不显示时，检查该行或日志。

<h3 id="draw-a-grid-of-colored-cells">
  绘制彩色单元格网格
</h3>

对于热力图、迷你图或终端中的游戏板，绘制一个 `Raster` 而不是每个单元格的 `Box`。`Raster` 接受 `key`、其大小（以 `columns` 和 `rows` 为单位）和 `cells`，它将每个单元格打包到一个字符串中。每个单元格是三个数字：字符的代码点、其颜色和其背景颜色。颜色是十六进制数字，红、绿、蓝各两位，例如 `0xc62828` 表示红色，或 `0x01000000` 表示终端的默认值。

桌面应用没有 `Raster`，所以检查 `e.surface` 并在那里绘制文本。此窗格主体绘制一个三乘二的热力图：

```javascript theme={null}
// 表示"使用终端的默认颜色"的值
const DEFAULT_COLOR = 0x01000000

// 将 [character, color] 对的行打包到 Raster 接受的一个字符串中
// 一个单元格是三个数字：字符的代码点、其颜色和其背景
function cellsOf(rows) {
  const numbers = rows.flat().flatMap(([char, color]) => [char.codePointAt(0), color, DEFAULT_COLOR])
  return new Uint8Array(Uint32Array.from(numbers).buffer).toBase64()
}

on('ui.render', { component: 'Pane' }, async ($, e, next) => {
  // 仅在使用 id 'heat' 打开的窗格中绘制
  if (e.requestId !== 'heat') return next(e)
  const { Box, Text, Raster } = $.ui.resolve(e)
  // 三个单元格的两行，每个都是一个块字符及其颜色
  const rows = [
    [['█', 0x2e7d32], ['█', 0xf9a825], ['█', 0xc62828]],
    [['█', 0x2e7d32], ['█', 0x2e7d32], ['█', 0xf9a825]],
  ]
  if (e.surface !== 'terminal') {
    return Text({ children: ['The heat map needs the terminal.'] })
  }
  return Box({
    flexDirection: 'column',
    children: [Raster({ key: 'grid', columns: 3, rows: 2, cells: cellsOf(rows) })],
  })
})
```

在终端中，窗格显示网格：

<img src="https://mintcdn.com/claude-code/dgiVO_Od1X1faduV/images/mods-heat-map.svg?fit=max&auto=format&n=dgiVO_Od1X1faduV&q=85&s=b91bcce3bad74bc851149133d4acc5d5" alt="终端中的一个窗格，包含一个小的彩色块网格，两行三列。顶行是绿色、琥珀色和红色。底行是绿色、绿色和琥珀色。" width="360" height="132" data-path="images/mods-heat-map.svg" />

`rows` 数组是你要更改的部分，`cellsOf` 将其转换为打包的字符串。钩子仅在 `id` 为 `heat` 的窗格中绘制，所以从命令中使用 `$.ui.open({ id: 'heat' })` 打开一个，如 [`hello-tabs` 示例](#build-a-pane-with-tabs) 打开其窗格。

每个字符必须是一个单元格宽。要动画化已经在屏幕上的 `Raster`，请使用窗格的 `id` 作为 `requestId`、`Raster` 的 `key`、相同的大小和新单元格调用 `$.ui.blit`。对于此示例，这是 `$.ui.blit({ requestId: 'heat', key: 'grid', columns: 3, rows: 2, cells: cellsOf(newRows) })`。它重新绘制该一个元素而不再次运行你的 `ui.render` 钩子。

<h2 id="respond-to-presses-and-typing">
  响应按键和输入
</h2>

当用户按下你绘制的按钮、输入字段或从列表中选择时，Claude Code 调用你给该控件的函数，它在你的模块中运行。每个控件接受其自己的回调：

* **`Button`**：接受 `onPress(e)`，其中 `e.surface` 是按键来自的应用
* **`Input`**：接受 `onSubmit(value)` 和 `onInput(value)`
* **`Select`**：接受 `onSelect(value)`，其选择在 `options` 中，至少一个选择的列表，具有唯一值，例如 `[{ value: 'sm', label: 'Small' }, { value: 'lg', label: 'Large' }]`

测试通过其 `key` 按下或输入到控件中，所以给每个控件一个。控件的每次使用也会触发 [`ui.press`、`ui.input` 或 `ui.select`](/docs/zh-CN/plugins/mods/reference#interface)，其中 `key` 在 `e.element` 中，另一个 mod 可以钩住这些事件。其钩子在你的回调之前运行，所以它看到用户输入到你的 `Input` 中的内容，可以更改它或代替你的回调回答。mod API 没有按下另一个 mod 的按钮的方法。

<h3 id="know-which-keys-your-mod-can-receive">
  键盘焦点和快捷键
</h3>

你的 mod 永远不会自己读取键盘。用户按下一个键，Claude Code 决定它是为你的哪个控件，该控件的回调运行。除了[条带上的数字快捷键](/docs/zh-CN/plugins/mods/reference#elements)，这仅在你的窗格或条带有键盘焦点时发生。其余时间，键进入提示符。

<h4 id="how-a-pane-gets-keyboard-focus">
  窗格如何获得键盘焦点
</h4>

窗格通过以下三种方式之一获得键盘焦点：

* 你的 mod 从命令或按键使用 `focus: true` 打开它
* 用户按 Ctrl+X 然后 Tab
* 用户点击它

Claude Code 仅在提示符为空且没有其他内容有键盘焦点时授予 `focus: true`。在用户输入时打开的窗格不会获取他们的按键。

<h4 id="what-each-key-does">
  每个键做什么
</h4>

此表列出了当你的窗格或条带有键盘焦点时每个键做什么：

| 键 | 它做什么 |
| :- | :- |
| Tab | 移动到下一个控件 |
| 上和下 | 在你的绘制适合时在控件之间移动。当窗格或条带的行数超过它可以显示的行数时，它们会滚动它。 |
| Enter | 按下焦点 `Button`、提交焦点 `Input` 或在 `Select` 中选择 |
| 按钮的快捷键 | 按下该按钮。当 `Input` 有焦点时，每个可打印键都进入字段。 |
| Esc | 将键盘焦点返回到提示符。使用 `closeOnEscape: true`，它也关闭窗格。 |

mod 无法将 Tab 或箭头键绑定到其他任何东西，所以游戏用 `w`、`a`、`s` 和 `d` 操舵。

<h4 id="set-a-hotkey-and-the-first-focus">
  设置快捷键和第一个焦点
</h4>

控件上的两个属性决定了键盘如何到达它：

* **`hotkey`**：要让用户用一个键按下 `Button`，给它一个 `hotkey`，一个数字或一个小写字母，如 `hotkey: 'a'`
* **`autoFocus`**：要选择窗格打开时哪个控件有焦点，向它添加 `autoFocus: true`。在其他上省略属性，因为 Claude Code 拒绝 `autoFocus: false`。

快捷键的显示方式取决于按钮和应用：

| 按钮 | 在终端中 | 在桌面应用中 |
| :- | :- | :- |
| 带括号，默认 | `[ Add one ]`，没有显示快捷键 | 标签，旁边有一个小键 |
| 使用 `plain: true` | `1: One` | 标签，旁边有一个小键 |

在终端中，在括号按钮的标签中命名键，或使用 `plain: true`，所以用户可以看到要按什么。[元素参考](/docs/zh-CN/plugins/mods/reference#elements) 有其他 `Button` 规则：`action`、条带上的数字快捷键和一个快捷键上的两个按钮。

<h3 id="take-typed-input-and-draw-a-row-for-each-item">
  获取输入的文本并为每个项目绘制一行
</h3>

许多窗格是一个文本字段，下面有一个列表。本部分中的示例是一个笔记窗格：你输入一个笔记并按 Enter 添加它，每个笔记都有一个删除它的 `x` 按钮。添加两个笔记后，终端这样绘制窗格：

```text theme={null}
╭──────────────────────────────────────────────────────────╮
│ Note: Type a note and press Enter ⏎ add                ✕ │
│ x buy milk │
│ x call bob │
╰──────────────────────────────────────────────────────────╯
```

示例使用两种技术：

* **获取输入的文本**：当用户按 Enter 时，`Input` 使用字段的文本调用 `onSubmit(value)`，在每次更改时调用 `onInput(value)`
* **绘制列表**：将你的数据映射到每个一行，并给每行的按钮其自己的 `key`

此钩子绘制窗格的内容：

```javascript theme={null}
// 窗格绘制的列表
let notes = []

on('ui.render', { component: 'Pane' }, async ($, e, next) => {
  // 仅在使用 id 'notes' 打开的窗格中绘制
  if (e.requestId !== 'notes') return next(e)
  const { Box, Text, Button, Input } = $.ui.resolve(e)
  const redraw = () => $.ui.invalidate('ui.render')

  return Box({
    flexDirection: 'column',
    children: [
      Input({
        key: 'new-note',
        label: 'Note',
        placeholder: 'Type a note and press Enter',
        // 每次绘制字段为空，这在提交后清除它
        value: '',
        submitLabel: 'add',
        autoFocus: true,
        // 当你在字段中按 Enter 时运行
        onSubmit: async (value) => {
          // 忽略空行
          if (!value.trim()) return
          notes = [...notes, value.trim()]
          redraw()
          await $.store.set('notes', notes)
        },
      }),
      // 每个笔记一行：一个删除按钮，然后是笔记的文本
      ...notes.map((note, i) =>
        Box({
          flexDirection: 'row',
          columnGap: 1,
          children: [
            Button({
              // 它自己的键，所以每行的按钮可以区分
              key: 'delete-' + i,
              label: 'x',
              plain: true,
              onPress: async () => {
                notes = notes.filter((_, j) => j !== i)
                redraw()
                await $.store.set('notes', notes)
              },
            }),
            Text({ children: [note] }),
          ],
        }),
      ),
    ],
  })
})
```

要尝试窗格：

* **添加笔记**：输入一行并按 Enter。该行显示为新行，字段清空。
* **删除笔记**：按 Tab 直到笔记的 `x` 按钮有焦点，然后按 Enter。`x` 是按钮的标签，不是快捷键，所以输入字母不会按下它。

每个更改遵循与 `hello-tabs` 相同的渲染周期：回调更改 `notes`，调用 `redraw`，并将列表保存到 `$.store`。

字段在每次提交后清空，因为其 `value` 属性。`value` 是绘制字段时保存的文本，用户的输入替换它，直到你的钩子再次绘制字段。示例总是用 `''` 绘制字段。

示例保存笔记而不加载它们。要在下一个会话中将它们带回，请在 `session.start` 钩子中读取它们，就像 `hello-tabs` 读取 `count` 的方式一样。

三个属性组成字段的行，`Note: Type a note and press Enter ⏎ add`：

| 属性 | 在示例中 | 它是什么 |
| :- | :- | :- |
| `label` | `Note` | 字段前的文本。终端在其后绘制 `: `。 |
| `placeholder` | `Type a note and press Enter` | 当字段为空时显示的暗文本 |
| `submitLabel` | `add` | `⏎` 后的单词，说明 Enter 做什么 |

提交 `Input` 不会启动轮次，除非你的回调调用 [`$.prompt.submit`](/docs/zh-CN/plugins/mods/api#start-a-turn-from-a-background-job)。

<h2 id="redraw-when-something-changes">
  重绘站点
</h2>

绘制是一个快照：它显示你的 `ui.render` 钩子上次运行时返回的内容。要显示新内容，钩子必须再次运行。Claude Code 为某些更改再次运行它，你的 mod 要求其余的。

<h3 id="when-claude-code-redraws-without-being-asked">
  当 Claude Code 在不被要求时重绘
</h3>

当站点的属性更改或终端的宽度更改时，Claude Code 再次运行你的 `ui.render` 钩子。它不在计时器上运行钩子，也无法判断你的模块中的变量何时更改。

<h3 id="redraw-when-your-data-changes">
  当你的数据更改时重绘
</h3>

要在你自己的数据更改后再次绘制你的站点，请调用 `$.ui.invalidate('ui.render')`。此窗格计数按键。按钮的回调更改 `count`，然后要求重绘：

```javascript theme={null}
let count = 0

on('ui.render', { component: 'Pane' }, async ($, e, next) => {
  if (e.requestId !== 'counter') return next(e)
  const { Box, Text, Button } = $.ui.resolve(e)
  return Box({
    flexDirection: 'row',
    columnGap: 2,
    children: [
      Button({
        key: 'more',
        label: 'Add one',
        onPress: () => {
          count += 1
          // 数据更改了，所以要求 Claude Code 再次绘制窗格
          $.ui.invalidate('ui.render')
        },
      }),
      Text({ children: ['Count: ' + count] }),
    ],
  })
})
```

每次按键都会提高窗格中的数字。[`hello-tabs` 示例](#build-a-pane-with-tabs) 将相同的调用包装在其 `redraw` 函数中。

你在 [`$.state`](#keep-a-value-in-\$-state) 中保存的值不需要调用，因为写入值会重绘读取它的站点。

<h3 id="redraw-on-a-timer">
  在计时器上重绘
</h3>

要保持时钟、倒计时或来自会话外部的值最新，请按计划重绘。在模块的 `session.start` 钩子中启动计时器。如果模块已经有一个，如 `hello-tabs` 所做的，请将 [`$.clock.every`](/docs/zh-CN/plugins/mods/api#run-work-in-the-background) 行添加到它：

```javascript theme={null}
on('session.start', async ($, e, next) => {
  // 每 1000 毫秒，要求 Claude Code 再次绘制你的站点
  $.clock.every(1000, () => $.ui.invalidate('ui.render'))
  return next(e)
})
```

Claude Code 现在每秒运行你的 `ui.render` 钩子一次。当模块重新加载时计时器停止，新副本启动其自己的。

<h3 id="how-often-a-site-can-redraw">
  站点可以重绘的频率
</h3>

Claude Code 限制重绘的频率，所以你的 mod 可以在其数据更改时调用 `$.ui.invalidate`。可见窗格和条带的限制比其他站点更高，[限制表](/docs/zh-CN/plugins/mods/reference#limits) 中有具体数字。

比限制更快的调用被合并为一次重绘。该重绘运行你的钩子一次，钩子读取你的数据，因为它在那一刻的样子，所以最新值显示，中间的值不显示。动画无法比限制运行得更快。

<h2 id="keep-state">
  保持状态
</h2>

mod 有三个地方可以保存值，它们在值持续多长时间方面有所不同：直到模块重新加载、直到会话结束或从一个会话到下一个会话。根据值必须持续多长时间选择：

| 在其中保存 | 它持续到 | 用于 |
| :- | :- | :- |
| 模块级变量 | 模块重新加载，这在开发期间每次保存文件时发生 | 你可以丢失的值，如 `hello-tabs` 中的 `tab` |
| `$.state` | 会话结束，或用户运行 `/clear`、`/resume` 或 `/branch` | 绘制依赖的值，应该在重新加载后存活 |
| `$.store` | 你的 mod 删除它，或没有会话在 [`cleanupPeriodDays`](/docs/zh-CN/settings-reference#cleanupperioddays) 内读取或写入存储。存储是一个键值存储，保存为你的插件自己的 JSON 文件，位于 `~/.claude/plugins/store/` 下。 | 设置、历史记录、用户期望下次找到的任何内容 |

`$.store.get(key)` 解析为值或 `undefined`，`$.store.set(key, value)` 接受任何 JSON 值。

<h3 id="keep-a-value-in-state">
  在 `$.state` 中保存值
</h3>

`$.state` 为会话的长度保存值，并为你重绘。它是反应式状态：读取值的 `ui.render` 钩子订阅它，所以 Claude Code 每次你写入值时重绘该站点，你不调用 `$.ui.invalidate`。`$.state` 中的值也在模块重新加载后存活，变量不会。

要设置它，声明你的值，将你的清单指向声明，然后定义和使用每个值。示例将 `count` 从 `hello-tabs` 移到 `$.state`。

<h4 id="declare-the-values">
  声明值
</h4>

在类型文件中声明值。外键是你的插件的名称，其下的每个条目是一个值及其类型。将其保存为 `hello-tabs/types/index.d.ts`：

```typescript hello-tabs/types/index.d.ts theme={null}
declare module 'claude-code' {
  interface PluginState {
    'hello-tabs': {
      tab: 'one' | 'two'
      count: number
    }
  }
}
```

<h4 id="point-the-manifest-at-the-declaration">
  将清单指向声明
</h4>

要让 `claude plugin validate` 根据该文件检查你的代码，请向清单添加 `types` 字段及其路径：

```json hello-tabs/.claude-plugin/plugin.json theme={null}
{
  "name": "hello-tabs",
  "version": "0.1.0",
  "description": "Opens a pane with two tabs and a counter",
  "author": { "name": "Your Name" },
  "types": "./types/index.d.ts"
}
```

<h4 id="define-read-and-write-a-value">
  定义、读取和写入值
</h4>

在你的模块中，定义每个值及其默认值，在绘制时读取它，并从回调中写入它。`atom` 命名值及其默认值，`read` 返回它，`update` 写入它。三个帮助程序为你调用 `$.state.get` 和 `$.state.set`：

```javascript theme={null}
import { atom, read, update } from 'claude-code'

// 在模块顶部：命名值并给出其默认值
const count = atom({ plugin: 'hello-tabs', key: 'count' }, 0)

// 在 ui.render 钩子中：读取值以绘制它
const n = await read($, count)

// 在按钮中：从旧值写入新值
onPress: () => update($, count, (value) => value + 1)
```

因为 `ui.render` 钩子读取了 `count`，Claude Code 每次按钮写入它时再次运行钩子。

三个规则适用于代码：

* **将 `plugin` 和 `key` 写成字面字符串**：`claude plugin validate` 从你的源代码中读取它们
* **在类型文件中声明每个值**：否则验证失败，出现 `hello-tabs.count is not declared`
* **从回调或另一个事件的钩子中写入**：`ui.render` 钩子可以读取状态，不能写入它，所以从 `onPress`、`onSubmit` 或另一个事件的钩子中写入

<h4 id="change-hello-tabs-to-use-state">
  更改 `hello-tabs` 以使用 `$.state`
</h4>

要将 `hello-tabs` 中的 `count` 移到 `$.state`，请更改使用它的每一行：

* **在模块顶部**：添加 `import` 行，并用 `atom` 行替换 `let count = 0`
* **在 `ui.render` 钩子中**：在 `tabButton` 之前添加 `read` 行，并在 `Text` 中绘制 `'Count: ' + n`
* **在 Add one 按钮中**：用[从多个会话保存](#save-from-more-than-one-session)中的按钮替换 `onPress`，它保存计数以及写入它
* **在 `session.start` 钩子中**：用[在 `/clear` 后再次加载保存的值](#load-a-saved-value-again-after-clear)中的 `loadCount` 调用替换读取 `saved` 的两行

为选项卡按钮保留 `redraw`，因为 `tab` 仍然是一个变量。

<h3 id="load-a-saved-value-again-after-clear">
  在 `/clear` 后再次加载保存的值
</h3>

如果你的 mod 在 `session.start` 时将保存的值从 `$.store` 复制到 `$.state`，它必须在 `/clear`、`/resume` 或 `/branch` 后再次复制。这些命令将每个 `$.state` 值放回其默认值，`session.start` 不再触发。[`classic.SessionStart`](/docs/zh-CN/plugins/mods/events#hook-the-settings-hook-events) 在每个之后触发，`e.source` 设置为 `clear`、`resume` 或 `fork`，所以在其上的钩子中再次复制值。否则你的绘制显示默认值，保存 `$.state` 值的回调将默认值写入你存储的内容。

此代码从两个钩子加载 `count`。它基于 `hello-tabs` 的 `$.state` 版本，其中 `count` 是原子，`update` 被导入。将 `loadCount` 放在 `register` 上方，并将 `loadCount` 调用添加到你已经拥有的 `session.start` 钩子。`classic.SessionStart` 也在启动和压缩后触发，这不会重置 `$.state`，所以对 `source` 的过滤将钩子保留到三个重置：

```javascript theme={null}
// 将保存的计数从 $.store 复制到 $.state，如果没有保存任何内容则为 0
async function loadCount($) {
  const saved = Number((await $.store.get('count')) ?? 0)
  await update($, count, () => saved)
}

// 在你的第一个提示符之前运行，以及重新加载后再次运行
on('session.start', async ($, e, next) => {
  await loadCount($)
  return next(e)
})

// 在 /clear、/resume 和 /branch 后再次运行，报告 fork
on('classic.SessionStart', { source: ['clear', 'resume', 'fork'] }, async ($, e, next) => {
  await loadCount($)
  return next(e)
})
```

两个钩子就位后，窗格在 `/clear` 后显示保存的计数，而不是 `0`，**Add one** 的下一次按键添加到保存的计数。

`loadCount` 将存储的值写入 `$.state` 中的值，`session.start` 每次模块重新加载时再次触发。要保持存储不落后，请在每次更改时保存，如 **Add one** 按钮所做的。

要在不会话的情况下检查重新加载，请[在 `/clear` 后测试绘制](/docs/zh-CN/plugins/mods/test#test-a-drawing-after-clear)。

<h3 id="save-from-more-than-one-session">
  从多个会话保存
</h3>

你的机器上运行你的 mod 的每个会话共享一个 `$.store`。`get` 后跟 `set` 不是原子的。当两个会话各自读取值、更改它并写回时，它们竞争，第二次写入替换第一次。

两个选择使这种情况不太可能：

* **给每个项目其自己的键**：`set` 仅更改其自己的键，所以写入不同键的会话不会相互覆盖
* **在写入前再次读取**：对于多个会话更改的值，在回调中 `get` 键，并从该值构建新值，而不是从你在 `session.start` 加载的副本。如果另一个会话的写入落在你的 `get` 和 `set` 之间，它仍然会丢失。

此按钮将一个添加到存储现在保存的任何内容，然后更新绘制：

```javascript theme={null}
onPress: async () => {
  // 读取存储现在保存的内容，另一个会话可能已更改
  const saved = Number((await $.store.get('count')) ?? 0)
  // 保存新计数，然后显示它
  await $.store.set('count', saved + 1)
  await update($, count, () => saved + 1)
}
```

如果第二个会话自此会话启动以来按下了其自己的按钮三次，此按键显示并保存包括这三个的计数。

<h2 id="next-steps">
  后续步骤
</h2>

* [对事件做出反应](/docs/zh-CN/plugins/mods/events)：从工具调用和轮次提供你的绘制
* [使用 mod API](/docs/zh-CN/plugins/mods/api)：从计时器和模型调用提供你的绘制
* [测试绘制](/docs/zh-CN/plugins/mods/test#test-a-drawing)：从测试按下你的按钮，在多个表面上
* [渲染站点](/docs/zh-CN/plugins/mods/reference#render-sites)和[元素](/docs/zh-CN/plugins/mods/reference#elements)：每个站点的属性和每个元素的属性
