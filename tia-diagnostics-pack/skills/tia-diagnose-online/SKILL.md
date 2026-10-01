---
name: tia-diagnose-online
description: TIA 在线诊断剧本——CPU 行为异常、模块疑似故障、在线值与预期不符时,按固定判断树走「连接探活 → 编译状态 → 在线采样 → 一致性检查 → 诊断缓冲区分诊」,每步给出具体工具调用序列。当用户说「PLC 不对劲」「模块报故障」「在线看看」「诊断一下」时使用。
---

# TIA 在线诊断剧本

与 tia-operations(操作纪律)互补:本技能只管**已经在出问题的现场怎么查**。

## 0. 起手(任何诊断动作之前)

```text
1. tia_acquire                     ← 取 TIA 租约;被拒时错误里带持有方与剩余时间,等待或协调,不重试硬抢
2. tia_occupancy                   ← 看一眼谁在持租,避免盲抢
3. tia_bootstrap                   ← 环境状态 + recommendedNextTool
4. tia_connect                     ← 附着运行中的 TIA 实例(单实例契约:Openness/MCP 连的就是 GUI 那个)
```

**单会话纪律**:诊断全程一个 TIA 会话贯穿,默认不释放;多 agent 协作一律先租约后动手。

**降级顺序**(每步失效再走下一步):L0 own-host op → vendored 逃生门(`TIA_MCP_SERVER_PATH`)→ L2 注入桥观察 → L3 CU 兜底。L3 CU 默认禁用(`SEMA_CU_DISABLE=1`),每次使用须用户当场批准,弹窗仅限白名单(`SEMA_CU_POPUP_*` opt-in)。

## 1. Openness 盲区(先知道查不了什么)

| 想要 | Openness 能不能 | 正路 |
|---|---|---|
| 读 CPU RUN/STOP | ❌ | OPC UA 客户端,或 S7 直连 `tia_call GetPlcRunStateS7` |
| 读诊断缓冲区 | ❌ | OPC UA 或 S7 SZL 请求 |
| 切换 Run/Stop | ❌ | OPC UA 或 TIA GUI |
| 清除全部强制 | ❌ | 删强制表条目后重新下载 |

这些盲区**不是故障**,别在 Openness API 里反复重试。

## 2. 判断树第一层:目标 CPU 探活

```text
A. plcsim_status                 ← 目标是 PLCSIM?keeperAlive / apiFace 一目了然
B. tia_call GetPlcRunStateS7     ← S7 直连读 RUN/STOP(rack 0 slot 1)
C. tia_call ReadPlcLiveValuesS7  ← 读实时值,itemsJson 必须是纯地址字符串数组
                                    (["I0.0","Q0.0","M0.0"]),对象数组报 Unrecognized address
```

分诊:

- **TCP 连接错误 / 102 端口不通** → 目标根本没在监听。注意:**程序装载完成前 PLCSIM 实例的 102 端口不通**(S7 服务器随实例 RUN + 程序装载后才可用);物理机 102 由系统服务 s7oiehsx64 占用,连通≠实例活着,以 GetPlcRunStateS7 结果为准。
- **PLCSIM 相关** → 转 `tia-download-troubleshoot` 技能处理实例/网络问题。
- **S7 通但值不对** → 进第 3 层在线采样。

## 3. 判断树第二层:程序状态与编译一致性

```text
1. tia_project({action:'tree'})        ← 拿真实路径,绝不猜 PLC_1
2. tia_call CompileSoftware            ← 当前编译态;inconsistent 块不可导出/下载
3. tia_call compile_status / compile_report   ← L2 注入桥观察面(先探一次触发 hooks 惰性安装)
```

判读要点:

- **连续二次编译 = up-to-date 是 TIA 正常行为,不是编译器坏了**。要制造真编译:关工程重开——MCP `CloseProject` → `OpenProject(path:)`(参数名是 `path` 不是 `projectPath`)→ 首次 CompileSoftware 才是真编译。
- **在线模式下 Import/Save 全被拒**("not permitted in online mode")→ 先幂等 `GoOffline`,这不是故障。
- **在线/离线块不一致** → 症状是下载被拒或行为与源码不符;处置 = 重新编译 + 全量下载(Openness 只支持整包下载,无选择性下载)。

## 4. 判断树第三层:在线行为采样

```text
tia_online({action:'sample'})     ← 在线采样;复杂工况用 tia_online 的 read_vars/sample 通道
```

采样纪律(违反必误诊):

- **采样地板 ~50ms/样本**:寿命短于 ~50ms 的瞬态漏采**不是程序 bug**,别陷入「重算→又错→怀疑代码」死循环。
- 瞬态行为验**持久后果**(锁存/计数自增),不追脉冲穿行。
- 时序看形状(状态轮转顺序、单调推进),不断言「N 秒后正好 = X」。
- **OpenPLC 验证通过 ≠ S7 行为等价**(扫描周期、FB 背景 DB、TON 约定、RETAIN 都有语义差)——在真机上必须重采样确认。

## 5. 判断树第四层:模块/硬件状态

Openness 无直接模块健康读数,按序:

```text
1. tia_find_tools({query:'diagnostic buffer'}) / ({query:'module state'})
   → tia_call({name:'<命中工具>', args:{...}})     ← 208 工具面先搜再打
2. 诊断缓冲区走 S7 SZL 或 OPC UA(见第 1 节盲区表)
3. 项目树在线/离线设备项对比:tia_project({action:'tree'}) 后比对 device item 状态
```

硬件线索判读:

- 固件回退(如 CPU FW 4.0 不在本机目录回退 V2.9)属正常兼容行为,记录即可。
- fresh 工程硬件编译报 3 错(访问等级/机密数据密码/通信证书)→ 缺一次性 GUI 配置,见 tia-download-troubleshoot 第 6 节。

## 6. 收尾

```text
1. 结论写清:哪一层发现的问题、证据原文(错误码/应答尾部)、处置动作
2. 改动过工程的:tia_compile(0 errors) → tia_project({action:'save'})
3. tia_release(如会话不再需要;批跑场景保持持有)
```

**兜底重启纪律**:Openness 层关不干净工程/会话卡死时,杀 `Siemens.Automation.Portal`(及 TiaMcpServer)进程后重启,比任何 close/unregister API 都可靠。杀进程前确认宿主(headless TIA)没连着 GUI TIA——宿主被杀会连带杀掉自己拉起的 headless 实例,GUI 侧管道断裂挂死。
