# 07 `@earendil-works/pi-tui`

## 1. 职责

`pi-tui` 是一个轻量终端 UI 框架，核心目标：

- **差分渲染**：只重绘变化的行/视口行，避免全屏刷新。
- **同步输出**：使用 CSI 2026，把一帧更新原子化，消除闪烁。
- **两种渲染器**：主屏（保留终端滚动历史）与备用屏（应用自管视口、鼠标/键盘滚动）。
- **组件化**：统一 `Component.render(width): string[]` 接口。
- **终端能力适配**：Kitty 键盘协议、Kitty/iTerm2 图片、真彩色/256 色、终端调色板查询、剪贴板。

源码：[packages/tui/src](../packages/tui/src/)

## 2. 模块

| 文件/目录 | 职责 |
|---|---|
| [tui.ts](../packages/tui/src/tui.ts) | `Component`、`Container`、`TUI` 接口、`TuiBase`、覆盖层、鼠标分发、焦点 |
| [tui-main-screen.ts](../packages/tui/src/tui-main-screen.ts) | 主屏差分渲染器 |
| [tui-alt-screen.ts](../packages/tui/src/tui-alt-screen.ts) | 备用屏视口渲染器、滚动、搜索、选择 |
| [terminal.ts](../packages/tui/src/terminal.ts) | `Terminal` 接口与 `ProcessTerminal` |
| [layout.ts](../packages/tui/src/layout.ts)、[layout-node.ts](../packages/tui/src/layout-node.ts) | 约束布局计算与渲染 |
| [components/](../packages/tui/src/components/) | Text、Box、Editor、Markdown、SelectList、ScrollView、VStack/HStack 等 |
| [keys.ts](../packages/tui/src/keys.ts) | 键位解析、Kitty 协议、`matchesKey()` |
| [keybindings.ts](../packages/tui/src/keybindings.ts) | 可配置键位、冲突检测 |
| [colors.ts](../packages/tui/src/colors.ts)、[oklab.ts](../packages/tui/src/oklab.ts) | sRGB/ANSI/OKLCH/OKHSL 颜色与混色 |
| [terminal-image.ts](../packages/tui/src/terminal-image.ts) | 图片协议检测、编码、尺寸、单元格计算 |
| [utils.ts](../packages/tui/src/utils.ts) | ANSI 宽度、截断、按列切片、换行 |
| [stdin-buffer.ts](../packages/tui/src/stdin-buffer.ts) | 批量输入拆分、Kitty 序列处理 |
| [alt-screen-search.ts](../packages/tui/src/alt-screen-search.ts) | 备用屏搜索面板与匹配 |
| 其他 | fuzzy、autocomplete、kill-ring、undo-stack、word-navigation、wheel-scroll、latex、native clipbboard |

## 3. Component 模型

```ts
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  handleMouse?(event: TuiMouseEvent): TuiMouseEventResult | undefined;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

约定：

- `render(width)` 返回若干终端行，每行可包含 ANSI 序列与图片协议行。
- 宽度按终端单元格计算，CJK/emoji/组合字符需用 `visibleWidth()` 与 `sliceByColumn()`。
- 状态变化后调用 `tui.requestRender()`；主题变化调用 `invalidate()`。
- 可聚焦组件实现 `Focusable` 并在光标位置输出 `CURSOR_MARKER`，TUI 会把硬件光标放到该位置（IME 候选窗定位）。

`Container` 顺序渲染子组件，并缓存子组件高度用于鼠标命中测试。

## 4. TUI 接口与实现

### 4.1 共享接口

```ts
interface TUI extends Component {
  readonly mode: "regular" | "fullscreen";
  addChild/removeChild/clear
  setFocus(component)
  showOverlay(component, options?): OverlayHandle
  hideOverlay() / hasOverlay()
  start() / stop(options?)
  renderNow(force?) / requestRender(force?)
  addInputListener(listener)
  setLayoutRoot(component)
  queryTerminalColors(...)
}
```

### 4.2 `TuiMainScreen`

- 渲染到主终端缓冲区，保留 scrollback。
- 维护 `previousLines`、`previousWidth/Height`、光标行、硬件光标行、已渲染最大行数。
- 更新时只写变化行；必要时向上滚动，增量重绘。
- 适合聊天式应用：历史由终端保留，应用只管理当前附加区域。

### 4.3 `TuiAltScreen`

- 进入备用屏（`?1049h`），应用拥有固定高度视口。
- 支持：
  - `VStack`/`HStack` 约束布局与 `ScrollView`。
  - 鼠标滚轮、拖拽、文本选择、双击选词、复制。
  - 键盘滚动、PageUp/PageDown、OSC 133 提示符跳转。
  - `Ctrl+Shift+F` 搜索面板。
  - 大图缓存、Kitty 图片放置与删除。
- 停止时恢复主屏并打印完整最终文档。

### 4.4 `TuiBase`

`TuiBase` 实现两种渲染器共用的：

- 子组件容器、焦点管理。
- 覆盖层栈、隐藏/聚焦/恢复、范围查询。
- 输入监听、终端颜色方案监听、终端颜色查询。
- `start/stop`、渲染调度（合并同一帧的多次 `requestRender`）。
- 鼠标事件分发与坐标变换。
- 渲染节流/日志/调试。

## 5. 差分渲染与同步输出

主屏渲染大致流程：

```text
requestRender()
  → 合并到下一帧
  → render(width) 得到目标行
  → 与 previousLines 逐行比较
  → 生成最小 ANSI 更新（移动光标、清行、写变化段）
  → 用 CSI 2026 包裹一帧（BEGIN/END synchronized output）
  → 更新 previousLines / cursor 状态
```

注意点：

- 超长帧按 1 MiB 分块写入，避免 V8 字符串上限。
- 图片行不可与普通文本直接合并；删除/重绘时需调用 Kitty 图片删除序列。
- `clearOnShrink` 控制内容变短时是否清除旧行。
- 终端 resize 通过 `SIGWINCH` 刷新尺寸。

## 6. 布局

备用屏支持显式布局：

```ts
tui.setLayoutRoot(new VStack([
  { component: new ScrollView(transcript, { follow: "end", primary: true }), basis: 0, grow: 1, minSize: 1 },
  { component: editorAndFooter, basis: "auto", shrink: 1, minSize: 1 },
]));
```

- 栈条目支持 `basis`、`grow`、`shrink`、`minSize`、`maxSize`、响应式 `visible`。
- `ScrollView` 拥有滚动状态；`primary` 接收键盘导航与空白区域滚轮。
- 布局每帧重建，但有状态组件与其渲染缓存保留。
- 主屏模式下这些布局语义不可用，因为滚动由终端拥有。

## 7. 覆盖层

```ts
const handle = tui.showOverlay(component, {
  width: "80%", maxHeight: "50%",
  anchor: "center", offsetX: 2, offsetY: -1,
  margin: 2, nonCapturing: false,
});
```

`OverlayHandle` 支持 `hide()`、`setHidden()`、`focus()`、`unfocus()`、`getBounds()`。

覆盖层栈支持：

- 多个覆盖层按 focusOrder 管理。
- 非捕获覆盖层不抢焦点。
- 覆盖层内的组件仍可接收鼠标。
- 焦点恢复策略：`clear` / `preserve`，并可显式指定恢复目标。

## 8. 输入与键位

- `parseKey(data)` 把原始输入解析为 `KeyId`。
- `matchesKey(data, "ctrl+c")` 判断按键。
- `KeybindingsManager` 支持默认键位、用户配置、冲突检测；应用键位与编辑键位分开。
- Kitty 键盘协议开启后支持 key release/repeat；`wantsKeyRelease` 决定组件是否接收 release。
- `StdinBuffer` 处理：
  - 批量粘贴拆分。
  - 超长粘贴标记（`[paste #1 +123 lines]`）。
  - 终端查询响应与普通输入的区分。
- `Editor` 组件实现多行编辑、撤销、kill ring、模糊补全、路径/斜杠补全、粘贴标记、鼠标选择。

## 9. 颜色

`Color` 可以是：

- 索引色（0-15 跟随终端调色板）。
- sRGB。
- OKLCH / OKHSL。

API：

```ts
parseColor("oklch(70% 0.12 220)")
mixColors(a, b, ratio)
colorToRgb(color)
foregroundAnsi(color, mode)
backgroundAnsi(color, mode)
styleText(text, style, mode)
getTerminalColorMode()
```

`styleText()` 根据终端能力输出 truecolor 或 256 色。OKLCH 转换较贵，频繁渲染的颜色应预先转成 sRGB。

## 10. 终端图片

支持：

- Kitty graphics protocol。
- iTerm2 inline images。
- 终端能力探测与缓存。
- PNG/JPEG/GIF/WebP 尺寸读取。
- 单元格尺寸、图片行数计算、占位/回退。
- 备用屏缓存屏幕外图片的传输与解码字节上限。

主要 API：`detectCapabilities()`、`getCapabilities()`、`renderImage()`、`encodeKitty()`、`encodeITerm2()`、`deleteKittyImage()`、`setCellDimensions()`。

## 11. 组件清单

内置组件：

`Text`、`TruncatedText`、`Input`、`Editor`、`Markdown`、`Loader`、`CancellableLoader`、`SelectList`、`SettingsList`、`MouseRegion`、`Spacer`、`Image`、`Box`、`Container`、`VStack`、`HStack`、`ScrollView`。

其中：

- `Markdown` 使用 `marked` 解析并渲染 ANSI 样式、代码高亮、mermaid（由上层接入）。
- `Editor` 是最复杂组件，包含补全、撤销、历史、粘贴、鼠标、软换行、光标。
- `SelectList`/`SettingsList` 提供键盘/鼠标选择、分页、搜索。

## 12. 原生集成

`native/` 提供可选原生模块：

- macOS/Linux/Windows 剪贴板。
- 原生修饰键检测（Shift+Enter 等）。
- 预构建 `.node` 随包发布；缺失时回退纯 JS。

## 13. 依赖与使用方

依赖：`marked`、`get-east-asian-width`。无内部依赖。

主要使用方：`pi-coding-agent` 的交互模式与扩展 UI。

## 14. 测试

`packages/tui/test/` 使用虚拟终端（`@xterm/headless`）验证渲染、布局、覆盖层、键位、颜色、图片、鼠标、截断、宽字符等。

```bash
cd packages/tui
node --test test/tui-render.test.ts
```
