# midea-lan（个人 fork）

上游原仓库：https://github.com/wuwentao/midea-lan

这是 [wuwentao/midea-lan](https://github.com/wuwentao/midea-lan) 的个人 fork，只为一个
问题而存在：[wuwentao/midea_ac_lan#658](https://github.com/wuwentao/midea_ac_lan/issues/658)。
美的「星光PRO」这类只回 BB 子协议的家电（型号 `223J6397`），在 Home Assistant 里一直
报 `NoSupportedProtocol`，只能手动重载集成才能短暂恢复。

库的完整文档、源码、issue 和发行版都在上游，本仓库不重复一遍：

- [上游仓库](https://github.com/wuwentao/midea-lan)
- [上游英文 README](https://github.com/wuwentao/midea-lan#readme)／[上游中文 README](https://github.com/wuwentao/midea-lan/blob/main/README_hans.md)
- [上游 issue 区](https://github.com/wuwentao/midea-lan/issues)／[上游 releases](https://github.com/wuwentao/midea-lan/releases)
- [Home Assistant 集成仓库](https://github.com/wuwentao/midea_ac_lan)

下面只说明本 fork 改了什么。完整背景、实测数据、安装与回滚步骤见
[PERSONAL_FIX.md](./PERSONAL_FIX.md)。

## 分支

| 分支                                                | 用途                                                                               |
| --------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `personal/ac-probe-fallback-fix（针对星光PRO修订）` | 自用的 pin 分支，Home Assistant 装的就是它；由自动化每 6 小时 rebase 到上游 `main` |
| `fix/probe-fallback-family`                         | 上游 PR [#113](https://github.com/wuwentao/midea-lan/pull/113) 的来源分支          |
| `main`                                              | 承载 `Keep the fix build current` 自动化，不参与发布                               |

## 改了什么

相对上游 `main` 只有 3 个提交，彼此独立，都可以单独 cherry-pick：

| 提交                                                             | 文件                                                    | 说明                                                                                                                                                                                                                                                    |
| ---------------------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fix: retry a timed-out protocol probe once before blacklisting` | `midealan/device.py`                                    | 协议探测时单次超时不再直接把命令拉黑，最多重试 `QUERY_PROBE_RETRIES = 2` 次；重发前清空 `_buffer`，避免半帧粘到重试回包导致解帧失败。重发的**写**超时属于链路故障，向上抛给连接恢复而不是拉黑                                                           |
| `fix: cap the reconnect backoff at one minute`                   | `midealan/device.py`                                    | 重连退避上限从 600 秒降到 `MAX_RECONNECT_SLEEP = 60`，设备恢复后最多 1 分钟自动接回，不用手动重载                                                                                                                                                       |
| `fix(ac): probe an alternative query family before giving up`    | `midealan/device.py`、`midealan/devices/ac/__init__.py` | 新增 `build_query_fallback()` 钩子与 `_probe_query_reply()`：主查询族（B5 `CapabilitiesQuery`／`0x41` 状态查询）整族静默时，在同一次探测里再试一次备用族；AC 用 BB 子协议（`SubProtocolQuery10/11/30`）实现它，基类默认返回空，所以其它机型行为完全不变 |

为什么算通用修法而不是打补丁：上游的 `AC_MODEL_CAPABILITIES` 是静态型号表，每来一个
BB-only 新机型都要再补一行；这里改成在探测阶段**运行时**发现"主族整体静默"，再由设备类
提供候选备用族。设备整体不可达时两族都不应答，不会误判、不会切换。

## 验证

`python -m pytest ./tests/`、`ruff check .`、`mypy midealan` 全绿；三个修复都做过
"回退源码即反向失败"的验证，测试里的 BB 回帧是用真机（223J6397）抓下来的三帧。
细节见 [PERSONAL_FIX.md](./PERSONAL_FIX.md)。

## 装到 Home Assistant

两条路（HACS 装 fork 集成，或直接往 HA 容器里装这个库）连同自动更新、回滚都写在
[PERSONAL_FIX.md](./PERSONAL_FIX.md) 里。上游合并等价修复并发布之后，把集成换回
[wuwentao/midea_ac_lan](https://github.com/wuwentao/midea_ac_lan) 即可，本 fork 可以停用。

## 许可证

沿用上游的 MIT 许可、作者仍是 wuwentao，见 [LICENSE](./LICENSE)。
