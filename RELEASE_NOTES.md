# OpenHub v3.2.1 动态流闪退修复版（Release Notes）

## 修复背景
原版 OpenHub v3.2.1 在「动态（Activities）」信息流渲染时**必现崩溃**，实测在鸿蒙 4.2 / Android 12 上冷启动即崩。
根因：GitHub 部分 `PushEvent` 的 payload **缺少 `commits` 字段**，代码直接调用 `getCommits().size()` 未判空，触发 `NullPointerException`；
此外 GitHub 后续新增的事件类型与 action 值（如 `PullRequestReviewEventActionType.created`、`DiscussionEvent` 等）不在 2018 年的枚举内，
`Enum.valueOf()` 与 `switch(ordinal())` 也会抛异常。

## 修复内容（5 处，已实机验证 0 崩溃）
| # | 位置 | 修复 |
|---|---|---|
| 1 | `ActivitiesAdapter.java` PushEvent 分支 | `commits` 判空（`commits == null ? 0 : size()`），空数据时显示 0 条提交而非崩溃 |
| 2 | `ActivitiesAdapter.java` 4 个枚举辅助方法（Issue / Member / PullRequestReview / ReviewComment） | `valueOf()` 加 `try/catch`，未知 action 值兜底返回空文案 |
| 3 | `ActivitiesAdapter.java` `setActionAndDesc` | `switch(model.getType())` 前加 null 保护，未知事件类型（如 `DiscussionEvent` / `SponsorshipEvent`）不再 NPE |
| 4 | `ActivityRedirectionModel.java` `generateForCommit` | `commits` 为空 / 索引越界时降级为仓库级跳转，避免点击提交行崩溃 |
| 5 | 字节码层（APK） | 以上等价改动已在 APK（smali）层实机验证，覆盖同类隐患 |

## 发布 APK 信息
- **包名**：`com.thirtydegreesray.openhub`
- **versionName**：`3.2.1`
- **versionCode**：`44`（为绕过鸿蒙按包名复用旧 AOT 编译产物而提升，功能与原版一致；非版本号变更）
- **签名**：自签名（调试证书），安装需允许「未知来源」
- **验证**：启动 → 自动登录 → 动态流渲染 → 滚动 10+ 屏 → 下拉刷新，全程 0 FATAL，进程存活 10 分钟+

## 安装说明
1. 下载 `openhub_v3.2.1_fixed_crash.apk`。
2. 允许「未知来源」安装（自签名证书）。
3. 若覆盖安装后异常，请先**卸载旧版**再安装（清除鸿蒙 AOT 缓存）。

## 源码改动
- 提交：`dc9c1df`「fix: 修复动态流闪退（空 commits / 未知枚举值 / 未知事件类型）」
- 分支：`fix/crash-null-safety`（已合并至 `master`）
- 改动文件：`app/src/main/java/com/thirtydegreesray/openhub/ui/adapter/ActivitiesAdapter.java`、`app/src/main/java/com/thirtydegreesray/openhub/mvp/model/ActivityRedirectionModel.java`（+46 / -10）
