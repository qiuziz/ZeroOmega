# ZeroOmega fork: proxyman-pac 实时同步

本 fork 在官方 ZeroOmega 3.5.2 基础上把规则列表的刷新从「闹钟轮询」改成
**service worker 内 2 秒轮询 + 标签页活动触发**，让 `proxyman-pac` 生成新快照后
数秒内生效，而不是等 15 分钟 ~ 24 小时。

## 改动点

| 文件 | 改动 |
| --- | --- |
| `omega-target-chromium-extension/src/module/options.coffee` | 新增 `setupFastSync()` |
| `omega-target-chromium-extension/overlay/manifest.json` | 增加 `offscreen` 权限（历史遗留，可移除） |

`setupFastSync()` 做的事：

1. 每 2 秒 `fetch('http://127.0.0.1:8899/health')`，比较 `generated_at`
2. 变化时调用 `updateProfile(null, true)` 重新拉取 `/rules.txt` 并重编译 PAC
3. `chrome.tabs.onUpdated` / `onActivated` 触发同一检查（500ms 防抖），
   保证用户切回浏览器时立即同步
4. 原有 `chrome.alarms` 机制保留，作为 SW 休眠期间的兜底

## 为什么不用 offscreen / 为什么不是纯定时器

实测结论（Chrome 153）：

- **offscreen document 会被 Chrome 回收**，回收后定时器静默停止，服务端再也收不到轮询。
- **service worker 空闲后休眠**，`setInterval` 随之停止；ZeroOmega 自带的
  `setInterval(getPlatformInfo, 25s)` 保活并不足以维持 2 秒轮询。
- 因此「纯后台秒级轮询」在 MV3 下不可靠，必须叠加**用户操作触发**。

实际行为：SW 活跃期内是 2 秒轮询；SW 休眠后，用户切换标签或刷新页面会立即触发一次检查。

## 为什么不用源码构建

`npm run deps` 在 `bower install` 阶段全部 `ETIMEDOUT`（bower 从 github.com 拉 archive，
本机网络不可达），`omega-web/build` 无法生成。

`dist/zeroomega-pac` 是基于官方 `chromium-release.zip` (v3.5.2) 打**同逻辑补丁**得到的产物：

- `js/omega_target_chromium_extension.min.js` 注入与源码一致的 `setupFastSync`
- manifest 增加 `offscreen` 权限

网络恢复后执行 `cd omega-build && npm run deps && npm run build` 可从源码重建。

## 加载

Chrome 153 已禁用 `--load-extension`，必须手动加载：

1. `chrome://extensions` → 开启「开发者模式」
2. 「加载已解压的扩展程序」→ 选择 `dist/zeroomega-pac`
3. 禁用商店版 ZeroOmega（两个扩展同时控制代理会冲突）
4. 新扩展「导入/导出」→「从备份文件恢复」

## 验证结果

2026-10-26 实测（Chrome 153）：

| 场景 | 服务端快照生成 | 扩展拉取 | 端到端 |
| --- | --- | --- | --- |
| 新增规则 + 切换标签 | 1.07s | +3.18s | 4.52s |
| 清空规则 + 切换标签 | ~1s | +7.6s | 8.64s |

规则条数 3 → 7 → 3 双向同步均生效，扩展存储与页面显示一致。
