---
name: browser-dynamic-loading
description: 在 OpenClaw 浏览器中可靠处理 JavaScript 动态渲染页面（如去哪儿机票、携程、OTA 等）。当用户需要在 URL 不变、结果通过 API 异步加载的网站上填写表单、等待渲染、抓取数据时使用。
metadata:
  {"openclaw":{"emoji":"🌐"}}
---

# 浏览器动态加载 (Browser Dynamic Loading)

在 `browser` 工具下，稳定完成「填写表单 → 提交搜索 → 等待 JS 渲染 → 抓取结果」的完整流程，适用于航班、酒店、OTA 等 URL 不变、结果通过 API 动态插入 DOM 的页面。

## 行为约束

**不要反复提醒动态加载的限制。** 用户已知晓去哪儿、携程等 OTA 为 JS 动态渲染。直接按流程操作，不必主动说「该网站是动态加载的」「可能不稳定」等，除非用户明确询问原因。

## 何时使用

- 用户需要搜索/抓取去哪儿、携程、飞猪等 OTA 的机票/酒店结果
- 页面上有搜索表单，但结果由 JS 异步加载，URL 不变化
- 需要「填写 → 点击搜索 → 等结果出现 → 读数据」的连贯操作
- 遇到 "Can't reach the OpenClaw browser control service" 或 snapshot 为空，怀疑是等待时机不当

## 核心原则

1. **提交搜索后不要立即 snapshot**：先 `browser wait`，再 snapshot。
2. **用 `wait` 等 DOM/网络稳定**：避免在 JS 还在渲染时抓取，减少超时和断连。
3. **拉长超时**：动态页网络请求多，默认 15s 易超时；建议 `--timeout-ms 30000`。

## 标准流程（按顺序）

### 1. 导航到搜索页

```
browser navigate <搜索页 URL>
```

### 2. Snapshot 获取表单 ref

```
browser snapshot --interactive --efficient
```

用返回的 `[ref=eN]` 对出发地、目的地、日期、搜索按钮等元素执行 `type` / `click`。

### 3. 填写表单并提交

```
browser type <ref> "北京"
browser type <ref> "上海"
browser click <搜索按钮 ref>
```

### 4. 等待结果渲染完成（关键）

**不要立即 snapshot**，先等待 DOM 或网络稳定：

**方式 A（等网络空闲）：**
```
browser wait --load networkidle --timeout-ms 30000
```

**方式 B（等结果 DOM 出现）：**
```
browser wait --fn "document.querySelectorAll('.flight-item').length > 0" --timeout-ms 30000
```

**方式 C（组合，更稳）：**
```
browser wait --load networkidle --fn "document.querySelector('.result-list')" --timeout-ms 30000
```

按页面结构选择或组合；去哪儿/携程等可用 `.flight-item`、`.list-item` 等常见类名。

### 5. 抓取结果

```
browser snapshot --interactive --efficient
```

此时 DOM 已包含航班/酒店数据，可提取价格、时间、航司等信息。

## 稳定性增强

- **控制服务断连**：若频繁 "Can't reach the OpenClaw browser control service"，优先检查：
  - 是否使用 snap Chromium（易失败）→ 改用 `google-chrome-stable`
  - 是否 `headless: true` + `noSandbox: true`（Linux 常见需要）
  - 是否 snapshot 前未 wait，导致重页面卡超时
- **超时配置**：可在 `~/.openclaw/openclaw.json` 增加 `browser.remoteCdpTimeoutMs`、`remoteCdpHandshakeTimeoutMs` 等，或通过 `browser wait --timeout-ms` 拉长单次等待。

## 示例：去哪儿机票搜索

1. `browser navigate https://flight.qunar.com/`
2. `browser snapshot --interactive --efficient` → 拿到出发地、目的地、日期、搜索按钮 ref
3. `browser type e1 "北京"`（按实际 ref 替换）
4. `browser type e2 "上海"`
5. `browser click e3`（搜索按钮 ref）
6. `browser wait --load networkidle --fn "document.querySelectorAll('[class*=\"flight\"]').length > 0" --timeout-ms 30000`
7. `browser snapshot --interactive --efficient` → 解析航班列表

## 注意事项

- ref 在导航/重渲染后可能失效，若报错需重新 snapshot 获取新 ref。
- 选择器（如 `.flight-item`）需根据目标站点 DOM 调整，可用 `browser snapshot --format aria` 查看结构后确定。
