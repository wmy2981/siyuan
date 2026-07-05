# 修复移动端桌面模式下输入框自动聚焦唤起键盘

## 问题描述

在 SiYuan 移动端桌面模式（手机/平板上使用桌面 UI）下，打开弹窗、切换面板、选中笔记等操作时，输入框或编辑器区域会自动聚焦，导致虚拟键盘意外弹出，遮挡内容。

典型场景：

- 打开搜索弹窗（Ctrl+P / `/`）→ 搜索输入框自动聚焦 → 键盘弹出
- 打开文件重命名弹窗 → 输入框自动聚焦 → 键盘弹出
- 切换到大纲面板 → 筛选输入框自动聚焦 → 键盘弹出
- **选中一篇笔记打开 → 编辑器（contenteditable）自动聚焦 → 键盘弹出**
- 新建笔记本、创建文档等弹窗 → 输入框自动聚焦 → 键盘弹出

## 修复方案

两个策略配合：

1. **移除 `autofocus` 属性**：通过 `MutationObserver` 实时监控 DOM 变化，删除所有新增元素的 `autofocus` 属性
2. **拦截 `focus()` 调用**：覆写 `HTMLElement.prototype.focus`，对所有可输入元素（`input`、`textarea`、`select`、`[contenteditable]`）做用户手势检测——只允许在用户手势（触摸/点击/按键）后 300ms 内的聚焦操作

### 修复代码

应用方式：**设置 → 外观 → 代码片段 → 添加 JavaScript 代码片段**

```javascript
/**
 * 修复移动端桌面模式下所有可输入区域自动聚焦导致键盘意外弹出
 *
 * 问题原因：弹窗/面板打开或选中笔记时，input/textarea/编辑器等
 * 自动聚焦（autofocus 属性或程序化 .focus() 调用），
 * 触发系统键盘弹出遮挡内容。
 *
 * 修复策略：
 * 1. MutationObserver 移除所有 autofocus 属性
 * 2. 覆写 HTMLElement.prototype.focus，拦截可输入元素的自动聚焦
 */
(function() {
    "use strict";

    // 仅在触摸设备上生效（移动端 + 触摸屏笔记本）
    if (!("ontouchstart" in window) && navigator.maxTouchPoints <= 0) return;

    // ════════════════════════════════════════════
    // 策略一：移除 autofocus 属性
    // ════════════════════════════════════════════

    // 清理已存在的 autofocus
    var cleanupAutofocus = function() {
        document.querySelectorAll("[autofocus]").forEach(function(el) {
            el.removeAttribute("autofocus");
        });
    };
    cleanupAutofocus();

    // 监控新加入的 DOM 元素，移除 autofocus
    var observer = new MutationObserver(function(mutations) {
        mutations.forEach(function(m) {
            m.addedNodes.forEach(function(n) {
                if (n.nodeType !== 1) return;
                if (n.matches && n.matches("[autofocus]")) {
                    n.removeAttribute("autofocus");
                }
                if (n.querySelectorAll) {
                    n.querySelectorAll("[autofocus]").forEach(function(el) {
                        el.removeAttribute("autofocus");
                    });
                }
            });
        });
    });
    var targetNode = document.body || document.documentElement;
    if (targetNode) {
        observer.observe(targetNode, { childList: true, subtree: true });
    }

    // ════════════════════════════════════════════
    // 策略二：拦截程序化 .focus() 调用
    // ════════════════════════════════════════════

    // 匹配所有可以输入文本的元素
    var INPUT_SELECTOR = "input, textarea, select, [contenteditable]";

    // 记录最近一次用户手势时间
    var lastUserGesture = 0;

    var markGesture = function() {
        lastUserGesture = Date.now();
    };
    document.addEventListener("touchstart", markGesture, true);
    document.addEventListener("mousedown", markGesture, true);
    document.addEventListener("keydown", markGesture, true);

    // 用户手势有效窗口（毫秒）
    var GESTURE_WINDOW = 300;

    // 覆写 HTMLElement.prototype.focus
    // 比分别覆写 HTMLInputElement / HTMLTextAreaElement / HTMLSelectElement 更全面，
    // 可以覆盖 [contenteditable] 编辑器区域等所有可能的输入元素
    var originalFocus = HTMLElement.prototype.focus;
    HTMLElement.prototype.focus = function(options) {
        // 检查该元素是否为可输入元素
        if (this.matches && this.matches(INPUT_SELECTOR)) {
            // 不在用户手势窗口内 → 视为自动聚焦，忽略
            if (Date.now() - lastUserGesture > GESTURE_WINDOW) {
                return;
            }
        }
        // 非输入元素或手势窗口内的聚焦 → 正常放行
        return originalFocus.call(this, options);
    };
})();
```

## 原理详解

| 机制 | 说明 |
|------|------|
| `MutationObserver` | 监听 DOM 变化，新元素插入时立即移除 `autofocus` 属性，从源头阻止浏览器自动聚焦 |
| `HTMLElement.prototype.focus` 覆写 | 拦截所有元素的 `.focus()` 调用，通过 `matches(INPUT_SELECTOR)` 判断是否为可输入元素 |
| `INPUT_SELECTOR` | `input, textarea, select, [contenteditable]` — 覆盖标准表单控件 + SiYuan 编辑器区域 |
| 用户手势跟踪 | 监听 `touchstart` / `mousedown` / `keydown`（捕获阶段，优先于其他监听器） |
| `capture: true` | 捕获阶段确保手势标记优先于事件处理函数 |

## 事件流

```
用户点击/触摸
    ↓
touchstart/mousedown（捕获阶段 → 标记 lastUserGesture）
    ↓
弹窗打开 / 笔记加载 → 内部调用 element.focus()
    ↓
HTMLElement.prototype.focus（已覆写）
    ├── this.matches(INPUT_SELECTOR) ? → 否 → 放行（非输入元素）
    │
    └── 是 → Date.now() - lastUserGesture <= 300ms ?
         ├── 是 → 放行（用户主动触发）
         └── 否 → 忽略（自动聚焦，键盘不会弹出）
```

## 注意事项 / Trade-offs

- **300ms 窗口**：用户手势后 300ms 内的 `.focus()` 会被放行。如果某个操作有较长的异步延迟（如网络请求后聚焦输入框），窗口可能需要调整。实际使用中 300ms 覆盖绝大多数同步聚焦场景。
- **编辑器聚焦**：打开笔记时编辑器（`[contenteditable]`）的自动聚焦也会被拦截，用户需要手动点击编辑器区域开始输入。这是预期的行为——防止键盘意外弹出。
- **第三方输入法**：某些输入法（如 Gboard、搜狗）在输入框聚焦时可能会额外弹出工具栏，此修复通过阻止不必要的聚焦间接解决了这个问题。
- **桌面端无影响**：非触摸设备上 `ontouchstart` 检测为 false，代码不会执行。
