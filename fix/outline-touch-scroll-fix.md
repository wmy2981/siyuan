# 修复移动端桌面模式下大纲面板滑动被判定为拖动标题

## 问题描述

在 SiYuan 移动端桌面模式（手机/平板上使用桌面 UI）下，大纲面板中上下滑动操作被判定为拖动标题，导致无法正常滚动大纲面板。

## 根因分析

事件链路如下：

1. **`touchDragBridge.ts` → `handleManualTouchStart`**
   - 触摸大纲面板时，将 `touchstart` 转换为合成 `mousedown` 事件分发
   - 这是为了让桌面拖拽排序功能在触摸设备上能工作

2. **`Outline.ts` → `bindSort()`**
   - 接收到合成 `mousedown` 后，设置 `document.onmousemove` 开始跟踪拖拽
   - 拖拽阈值仅为 **3px**，极小移动即可触发

3. **`touchDragBridge.ts` → `handleManualTouchMove`**
   - 检测到 `document.onmousemove` 已设置，分发合成 `mousemove` 事件
   - `bindSort` 收到后判定为拖拽，创建拖拽幽灵元素，阻止正常滚动

问题的核心：**移动端垂直滑动被 3px 阈值的桌面拖拽机制劫持**。

## 代码位置

| 文件 | 作用 |
|------|------|
| `app/src/util/touchDragBridge.ts` | 触摸→鼠标事件桥接 |
| `app/src/layout/dock/Outline.ts` (line 348: `bindSort`) | 大纲面板拖拽排序 |
| `app/src/layout/dock/Outline.ts` (line 60: `.sy__outline` class) | 大纲面板 CSS 选择器 |

## 修复方案

在 `touchmove` **捕获阶段**检测垂直滚动意图，及时清除 `bindSort` 设置的拖拽监听状态。

### 修复代码

应用方式：**设置 → 外观 → 代码片段 → 添加 JavaScript 代码片段**

```javascript
/**
 * 修复移动端桌面模式下大纲面板滑动被判定为拖动标题
 *
 * 问题原因：touchDragBridge 将触摸转为鼠标事件后，
 * Outline.bindSort 的 mousedown 监听器会启动拖拽跟踪（阈值仅 3px），
 * 导致上下滑动被误判为拖动标题。
 *
 * 修复方案：在 touchmove 捕获阶段检测垂直滚动意图，
 * 及时清除 bindSort 设置的拖拽监听，恢复正常滚动。
 */
(function() {
    "use strict";
    // 非触摸设备不处理
    if (!("ontouchstart" in window) && navigator.maxTouchPoints <= 0) return;

    let touchStartY = 0;
    let touchStartX = 0;
    let cleaned = false;

    // 捕获阶段：在 handleManualTouchStart 之前记录触摸起点
    document.addEventListener("touchstart", function(e) {
        if (!e.target.closest(".sy__outline")) return;
        const t = e.touches[0];
        touchStartY = t.clientY;
        touchStartX = t.clientX;
        cleaned = false;
    }, {capture: true, passive: true});

    // 捕获阶段：在 handleManualTouchMove 之前检测垂直滚动意图
    document.addEventListener("touchmove", function(e) {
        if (cleaned || !touchStartY) return;
        if (!e.target.closest(".sy__outline")) return;

        const t = e.touches[0];
        const dy = Math.abs(t.clientY - touchStartY);
        const dx = Math.abs(t.clientX - touchStartX);

        // 垂直滑动 > 水平滑动 × 1.5 且超过阈值 → 判定为滚动而非拖动
        if (dy > 8 && dy > (dx || 1) * 1.5) {
            cleaned = true;

            // 清除 bindSort 设置的拖拽状态，阻止后续合成 mousemove
            document.onmousemove = null;
            document.onmouseup = null;
            document.ondragstart = null;
            document.onselectstart = null;
            document.onselect = null;

            // 移除拖拽幽灵元素
            const ghost = document.getElementById("dragGhost");
            if (ghost) ghost.remove();

            // 恢复被拖拽变透明的列表项样式
            const outline = document.querySelector(".sy__outline");
            if (outline) {
                outline.querySelectorAll('.b3-list-item[style*="opacity"]').forEach(function(el) {
                    el.style.opacity = "";
                });
                // 清理拖拽高亮类
                outline.querySelectorAll(
                    ".dragover__top, .dragover__bottom, .dragover, .dragover__current"
                ).forEach(function(el) {
                    el.classList.remove("dragover__top", "dragover__bottom", "dragover", "dragover__current");
                });
            }
        }
    }, {capture: true, passive: true});

    // 清理状态
    const reset = function() {
        touchStartY = 0;
        touchStartX = 0;
        cleaned = false;
    };
    document.addEventListener("touchend", reset, {capture: true, passive: true});
    document.addEventListener("touchcancel", reset, {capture: true, passive: true});
})();
```

## 原理详解

| 机制 | 说明 |
|------|------|
| `capture: true` | 在**捕获阶段**拦截事件，优先于 `touchDragBridge` 的冒泡阶段处理 |
| 垂直/水平比 > 1.5 | 区分"垂直滚动"和"任意方向拖拽" |
| 8px 阈值 | 远大于 bindSort 的 3px 阈值，避免误杀短距离滑动 |
| 清除 `document.onmousemove` | 阻止 `handleManualTouchMove` 分发合成 mousemove |
| `passive: true` | 不干扰浏览器原生滚动性能 |

## 事件修复时序

```
touchstart:
  ┌─ 捕获 (我们的 handler) ─→ 记录触摸起点
  ├─ 目标
  └─ 冒泡 (handleManualTouchStart) ─→ 分发合成 mousedown → bindSort 设置跟踪

touchmove:
  ┌─ 捕获 (我们的 handler) ─→ 检测到垂直滚动 → 清除 document.onmousemove ✅
  ├─ 目标
  └─ 冒泡 (handleManualTouchMove) ─→ document.onmousemove 为空 → 不分发 mousemove ✅

touchend:
  ┌─ 捕获 (我们的 handler) ─→ 重置状态
  ├─ 目标
  └─ 冒泡 (handleManualTouchEnd) ─→ 正常清理
```

## 已知影响

- **拖拽排序**：此修复会阻止触摸设备上的大纲面板拖拽排序功能（因为垂直滑动在有触摸屏的设备上难以区分"滚动"和"拖拽排序"——两者都是上下移动）
- **桌面端**：无影响，桌面端不使用触摸事件，`ontouchstart` 检测为 false
- **移动端原生模式**：无影响，移动端使用 `MobileOutline.ts`（无 `bindSort`）

如需在触摸设备上重新排序标题，建议在编辑器正文中拖拽块标操作。
