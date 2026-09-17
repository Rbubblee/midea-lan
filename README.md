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

## 怎么用

这个 fork 改的是 Home Assistant 集成所依赖的 `midea-lan` 库，集成本体还是官方的
`midea_ac_lan`。所以用法只有一件事：让 HA 用上这个库。装好之后不需要额外配置，
之前加过的集成和实体照常接上，只是原先一直不可用的那台会开始有数据。

前提是 HA 里已经添加过 `midea_ac_lan` 集成，而且设备就是 #658 里那类只回 BB 子协议的
家电；其它机型不需要这个 fork。

### 方案 A：HACS 自定义仓库（推荐，之后能自动更新）

1. HACS → 右上角 ⋯ → _Custom repositories_ → 填 `https://github.com/Rbubblee/midea_ac_lan`，
   类型选 _Integration_ → Add；
2. 列表里会出现两个同名的 `Midea AC LAN`（域名都是 `midea_ac_lan`）。先对原来那个
   （`wuwentao/midea_ac_lan`）点 _Remove_：这一下只删 `custom_components/midea_ac_lan/`
   目录和 HACS 自己的记录，HA 的 config entry 和实体注册表都会保留；
3. 安装刚添加的 `Rbubblee/midea_ac_lan`（HACS 会装它最新的 release，例如 `v2026.9.1`）；
4. 重启 HA core。

之后上游发新版本时，在 HACS 里点一次 _Update_ 再重启即可：fork 的集成每 6 小时检查上游
release，会自己重打 pin 并发布新 release；库这边也有自动化每 6 小时把分支 rebase 到上游。

### 方案 B：直接把库装进 HA 容器（最快，几分钟）

需要 `Advanced SSH & Web Terminal` 插件，并在插件配置里关掉 protection mode（要用
`docker` 命令）。

```bash
docker exec homeassistant python3 -m uv pip install --system --reinstall --no-deps \
  "midea-lan @ https://github.com/Rbubblee/midea-lan/archive/refs/heads/personal/ac-probe-fallback-fix%EF%BC%88%E9%92%88%E5%AF%B9%E6%98%9F%E5%85%89PRO%E4%BF%AE%E8%AE%A2%EF%BC%89.zip"

# 容器里没有 uv 模块时，换成 pip：
# docker exec homeassistant python3 -m pip install --no-cache-dir --force-reinstall --no-deps \
#   "midea-lan @ https://github.com/Rbubblee/midea-lan/archive/refs/heads/personal/ac-probe-fallback-fix%EF%BC%88%E9%92%88%E5%AF%B9%E6%98%9F%E5%85%89PRO%E4%BF%AE%E8%AE%A2%EF%BC%89.zip"

ha core restart
```

分支名里有中文，URL 里用的是百分号编码，别手动换成中文，否则 pip 拉不到。
HA core 更新会重建容器、把装进去的库冲掉，重跑一遍上面这条命令即可。

### 怎么确认生效

- 把 `midealan` logger 调到 `debug`，日志里应出现
  `no reply to [...], probing the alternative query family [...]`，随后设备 `available: True`；
- 实体恢复有值：`climate.*_climate`、`sensor.*_indoor_temperature`、`sensor.*_indoor_humidity`；
- 关机状态下 `current_energy_consumption` 为 `unknown` 属正常。

### 更新与回滚

- 更新：方案 A 在 HACS 里点 _Update_；方案 B 重跑上面的 `docker exec` 命令；
- 回滚：`docker exec homeassistant python3 -m uv pip install --system --reinstall "midea-lan==2026.9.1"`，
  然后 `ha core restart`；HACS 那边把仓库换回 `wuwentao/midea_ac_lan` 即可，PyPI 上的官方版本会自动装回；
- 上游合并等价修复并发布后，升级官方集成就行，本 fork 可以停用。

更完整的背景、协议细节和实测数据见 [PERSONAL_FIX.md](./PERSONAL_FIX.md)。

## 许可证

沿用上游的 MIT 许可、作者仍是 wuwentao，见 [LICENSE](./LICENSE)。
