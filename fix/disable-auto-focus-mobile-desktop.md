# 修复移动端桌面模式下输入框自动聚焦唤起键盘

## 问题描述

在 SiYuan 移动端桌面模式（手机/平板上使用桌面 UI）下，打开弹窗、切换面板等操作时，输入框会自动聚焦，导致虚拟键盘意外弹出，遮挡内容。

典型场景：

- 打开搜索弹窗（Ctrl+P / `/`）→ 搜索输入框自动聚焦 → 键盘弹出
- 打开文件重命名弹窗 → 输入框自动聚焦 → 键盘弹出
- 切换到大纲面板 → 筛选输入框自动聚焦 → 键盘弹出
- 新建笔记本、创建文档等弹窗 → 输入框自动聚焦 → 键盘弹出

## 修复方案

两个策略配合：

1. **移除 `autofocus` 属性**：通过 `MutationObserver` 实时监控 DOM 变化，删除所有新增元素的 `autofocus` 属性
2. **拦截 `focus()` 调用**：覆盖 `HTMLInputElement` / `HTMLTextAreaElement` / `HTMLSelectElement` 的 `focus()` 方法，只允许在用户手势（触摸/点击/按键）后 300ms 内的聚焦操作

### 修复代码

应用方式：**设置 → 外观 → 代码片段 → 添加 JavaScript 代码片段**

```javascript
/**
 * 修复移动端桌面模式下输入框自动聚焦导致键盘意外弹出
 *
 * 问题原因：弹窗/面板打开时 input/textarea 自动聚焦（autofocus 属性或
 * 程序化 .focus() 调用），触发系统键盘弹出遮挡内容。
 *
 * 修复策略：
 * 1. MutationObserver 移除所有 autofocus 属性
 * 2. 覆写 .focus() 方法，仅允许用户手势后的聚焦
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

    // 覆写输入类元素的 focus 方法
    var focusTargets = [
        { ctor: HTMLInputElement, proto: HTMLInputElement.prototype },
        { ctor: HTMLTextAreaElement, proto: HTMLTextAreaElement.prototype },
        { ctor: HTMLSelectElement, proto: HTMLSelectElement.prototype }
    ];

    focusTargets.forEach(function(item) {
        var originalFocus = item.proto.focus;
        item.proto.focus = function(options) {
            // 不在用户手势窗口内的 focus() 调用视为自动聚焦，忽略
            if (Date.now() - lastUserGesture > GESTURE_WINDOW) {
                return;
            }
            return originalFocus.call(this, options);
        };
    });
})();
```

## 原理详解

| 机制 | 说明 |
|------|------|
| `MutationObserver` | 监听 DOM 变化，新元素插入时立即移除 `autofocus` 属性，从源头阻止浏览器自动聚焦 |
| `.focus()` 覆写 | 拦截程序化调用，只有在用户交互后的 **300ms** 窗口内才放行 |
| 用户手势跟踪 | 监听 `touchstart` / `mousedown` / `keydown`（捕获阶段，优先于其他监听器） |
| `capture: true` | 捕获阶段确保手势标记优先于事件处理函数 |

## 事件流

```
用户点击/触摸
    ↓
touchstart/mousedown（捕获阶段 → 标记 lastUserGesture）
    ↓
弹窗打开，内部调用 input.focus()
    ↓
拦截检查：Date.now() - lastUserGesture <= 300ms ?
    ├── 是 → 放行（用户主动触发，确实是需要的）
    └── 否 → 忽略（自动聚焦，键盘不会弹出）
    ↓
用户手指离开屏幕 → touchend → 无额外处理
```

## 注意事项 / Trade-offs

- **300ms 窗口**：用户手势后 300ms 内的 `.focus()` 会被放行。如果某个操作有较长的异步延迟（如网络请求后聚焦输入框），窗口可能需要调整。实际使用中 300ms 覆盖绝大多数同步聚焦场景。
- **第三方输入法**：某些输入法（如 Gboard、搜狗）在输入框聚焦时可能会额外弹出工具栏，此修复通过阻止不必要的聚焦间接解决了这个问题。
- **桌面端无影响**：非触摸设备上 `ontouchstart` 检测为 false，代码不会执行。
