# 修复移动端桌面模式下输入框自动聚焦唤起键盘

## 问题描述

在 SiYuan 移动端桌面模式（手机/平板上使用桌面 UI）下，打开弹窗、切换面板、选中笔记等操作时，输入框或编辑器区域会自动聚焦，导致虚拟键盘意外弹出，遮挡内容。

典型场景：

- 打开搜索弹窗（Ctrl+P / `/`）→ 搜索输入框自动聚焦 → 键盘弹出
- 打开文件重命名弹窗 → 输入框自动聚焦 → 键盘弹出
- 切换到大纲面板 → 筛选输入框自动聚焦 → 键盘弹出
- **选中一篇笔记打开 → 编辑器（contenteditable）自动聚焦到标题 → 键盘弹出**
- 新建笔记本、创建文档等弹窗 → 输入框自动聚焦 → 键盘弹出

## 根因分析

SiYuan 中程序化聚焦有两条路径：

| 路径 | 方式 | 适用元素 | 是否被旧版修复覆盖 |
|------|------|----------|-------------------|
| `.focus()` 方法 | `element.focus()` | input / textarea / select | ✅ |
| Selection API | `selection.addRange(range)` | `[contenteditable]` 编辑器 | ❌ 绕过 `.focus()` 覆写 |

编辑器打开文档时调用 `focusBlock()` → `focusByRange()` → `selection.addRange(range)`，浏览器自动将焦点移至 contenteditable 容器，此过程不经过 `.focus()` 方法。

## 修复方案

**统一使用 `focusin` 事件捕获阶段拦截**——这是唯一能覆盖所有聚焦来源的机制：

1. **`MutationObserver` 移除 `autofocus` 属性**（从源头阻止浏览器自动聚焦）
2. **`focusin` 捕获阶段监听 + `element.blur()`**——拦截一切聚焦来源后立即失焦

### 修复代码

应用方式：**设置 → 外观 → 代码片段 → 添加 JavaScript 代码片段**

```javascript
/**
 * 修复移动端桌面模式下所有可输入区域自动聚焦导致键盘意外弹出
 *
 * 问题原因：弹窗/面板打开或选中笔记时，input/textarea/编辑器等
 * 自动聚焦（autofocus 属性、程序化 .focus() 调用或
 * Selection API contenteditable 聚焦），触发系统键盘弹出。
 *
 * 修复策略：
 * 1. MutationObserver 移除所有 autofocus 属性
 * 2. focusin 捕获阶段拦截 × blur，覆盖所有聚焦来源
 */
(function() {
    "use strict";

    // 仅在触摸设备上生效（移动端 + 触摸屏笔记本）
    if (!("ontouchstart" in window) && navigator.maxTouchPoints <= 0) return;

    // 匹配所有可输入文本的元素
    var INPUT_SELECTOR = "input, textarea, select, [contenteditable]";

    // ════════════════════════════════════════════
    // 策略一：移除 autofocus 属性
    // ════════════════════════════════════════════

    // 清理已存在的 autofocus
    document.querySelectorAll("[autofocus]").forEach(function(el) {
        el.removeAttribute("autofocus");
    });

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
    // 策略二：拦截一切聚焦来源（focusin 捕获阶段）
    // ════════════════════════════════════════════

    // 用户手势跟踪
    var lastUserGesture = 0;

    var markGesture = function() {
        lastUserGesture = Date.now();
    };
    document.addEventListener("touchstart", markGesture, true);
    document.addEventListener("mousedown", markGesture, true);
    document.addEventListener("keydown", markGesture, true);

    // 用户手势有效窗口（毫秒）
    var GESTURE_WINDOW = 300;

    // focusin 在捕获阶段拦截，优先于所有业务监听器
    // 覆盖：.focus() 调用、Selection API、浏览器 autofocus 等所有聚焦来源
    document.addEventListener("focusin", function(e) {
        var target = e.target;
        if (!target.matches || !target.matches(INPUT_SELECTOR)) return;
        if (Date.now() - lastUserGesture <= GESTURE_WINDOW) return;

        // 非用户手势触发的聚焦 → 立即失焦，阻止键盘弹出
        // 在捕获阶段 blur，浏览器尚未提交键盘显示，无闪烁
        target.blur();
    }, true); // capture: true
})();
```

## 原理详解

| 机制 | 说明 |
|------|------|
| `MutationObserver` | 监听 DOM 变化，新元素插入时立即移除 `autofocus` 属性 |
| `focusin` 捕获阶段 | **唯一能拦截所有聚焦来源的统一入口**（`.focus()`、Selection API、浏览器 autofocus） |
| 捕获阶段 blur | 在事件传播到业务代码前失焦，浏览器未提交键盘 UI，无视觉闪烁 |
| 用户手势跟踪 | 监听 `touchstart` / `mousedown` / `keydown`，300ms 窗口内放行 |

## 覆盖的聚焦路径

```
.focus() 调用
    └── 元素聚焦 → focusin 事件 → 捕获阶段拦截 → 检查手势窗口
                                            ├── 窗口内 → 放行
                                            └── 窗口外 → blur()

Selection API (selection.addRange)
    └── 浏览器聚焦 contenteditable → focusin 事件 → 同上

Browser autofocus (HTML autofocus 属性)
    ├── MutationObserver 移除属性（预防）
    └── 即使漏网 → focusin 事件 → 同上

用户手动点击/触摸输入区域
    └── touchstart → 标记手势 → 聚焦事件 → 窗口内 → 放行
```

## 注意事项

- **编辑器自动聚焦被拦截**：打开笔记时编辑器不会自动获得焦点，键盘不会弹出。用户需要**手动点击编辑器区域**开始输入。这是预期的行为。
- **无闪烁**：`blur()` 在 `focusin` 捕获阶段执行，浏览器尚未提交键盘 UI。
- **桌面端无影响**：非触摸设备上 `ontouchstart` 检测为 false，代码不会执行。
- **手势窗口 300ms**：若某项操作需要较长的异步延迟后聚焦（如网络请求完成后），可适当调大 `GESTURE_WINDOW` 值。
