---
name: tia-download-troubleshoot
description: TIA 下载失败分诊——「连接到模块失败」「找不到下载路由」「certificate not matching」「设备未信任弹窗」「not permitted in online mode」等下载链故障的排查树:PLCSIM 虚网卡/证书指纹白名单/接口选择/一次性 GUI 配置逐项定位,每步给出工具调用与配置键名。当用户说「下载失败」「下不进去」「连不上 PLC」时使用。
---

# TIA 下载失败分诊剧本

下载链标准四步(`tia-runbook` §3.4):

```text
1. tia_call CompileSoftware                      ← 0 errors,inconsistent 块不可下载
2. tia_call SetProfinetAddress {softwarePath, ipAddress, subnetMask}  → 再 CompileSoftware 硬件+软件生效
3. tia_call DownloadToPlc {softwarePath}         ← 自动路由;或经 tia_download(targetIpAddress)
4. S7 读回验证:tia_call ReadPlcLiveValuesS7      ← 前置:CPU 开 PUT/GET + 非优化 DB
```

**起手仍先 `tia_acquire` 租约**;全流程单 TIA 会话,不许中途换会话。

## 故障树(按报错原文对号入座)

### F1. 路由预检:「No download route reaches target IP ...」且网卡是 PLCSIM 虚网卡

- **根因**:PLCSIM Virtual Ethernet Adapter 走 APIPA,**无静态 IP**,提权配 IP 也静默失败(驱动不支持)——预检对显式 `targetIpAddress` 做 byIp 精确匹配永远 [no IP]。
- **处置**:**不传 `targetIpAddress`**,走默认自动路由(2026-09-26 起显式 IP 已支持:枚举无匹配时自动 `Create(ip)` + 5 参 Download,同址复用不堆积;但 PLCSIM 场景仍推荐不传)。
- **TCPIPSingleAdapter 模式下路由枚举地址恒为空是该模式固有形态**,不是反射缺陷;`pgPcRoute` 字段是「选中的候选」不是「实际生效链路」,不能当证据。
- 路由真实指向 PLCSIM IP 时,`tia_download` 会**自动调一次 `plcsim_ensure` 并重试一次**(结果里 `autoEnsured` 如实记录);显式外部 IP 绝不触发。

### F2. 下载执行期:「Connect to module PLC_1 failed」

按序查:

1. **PLCSIM 实例没起/没装载程序** → headless Openness **不会自动拉起标准 PLCSIM 实例**(GUI 下载向导的「启动仿真」一步不在 DownloadProvider 语义内)。用 `plcsim_ensure` 幂等拉起,顺序不可乱:
   ```text
   plcsim_ensure  → Manager 同 UAC → NetworkMode=TCPIPSingleAdapter(须零实例在跑时设置)
                  → RegisterInstance/attach → StoragePath(纯 ASCII,native 层 GBK 解析崩溃)
                  → PowerOn → SetIPSuite(必须在 PowerOn 之后,否则 -14)
                  → Run(-52 IsEmpty 空实例属正常)
   ```
   拉起后 keeper 保活,WMI 拉起已脱离 SemaTIA 进程树(应用重启不带走实例);收尾用 `plcsim_stop`(只杀该实例 keeper,绝不碰 Manager)。
2. **网络污染(物理机/Hub 场景)** → 手机 USB 共享网段(Remote NDIS 192.168.0.x)与 PLCSIM 网段同段会劫持路由;PLCSIM 操作期间禁用 USB 共享。多 Portal/AdapterConfigurator 孤儿进程占接口句柄 → 清孤儿后重试。
3. **全新 CPU 缺一次性 GUI 配置** → 见 F5。

### F3. 信任:「certificate not matching」/ 应答尾部带 `Trust prompts (1): plc=... fp=[...]`

- **根因**:设备证书不信任。信任失败提示载荷只有 `PlcName` + `VerificationInfo`,**无标记无 IP**,只能依据路由网卡名判定。
- **处置(标准流程)**:
  1. 随便配个不匹配的白名单值跑一次下载;
  2. 失败应答尾部抄出真指纹 `fp=[...]`;
  3. 写进工作区 `.sema/.mcp.json` 的 env 块:`SEMA_TIA_TRUST_FINGERPRINTS=<fp1,fp2>`(**sema-core 只认 .mcp.json 的 env,设置面板改动不落这里就不生效**);
  4. 开新 MCP 会话再下。下载成功/Error 态不透出 trustPrompts,只有信任失败路径有。
- **逃生门**:`SEMA_TIA_TRUST_ALL_SIM=1`(路由网卡为 Siemens PLCSIM Virtual Ethernet Adapter 时直通信任,适合 SIM 换证书);两者皆无 = Interactive(不挂订阅,同旧版行为)。
- 设备信任在 GUI 建立一次后**跨重启持久**。

### F4. 弹窗类:「Openness 访问」框 / 设备未信任弹窗 / 覆盖确认框

- **「Openness 访问」弹窗在宿主 exe 每次重建(hash 变)后必现**——点「全部确定」(Yestoall)保存授权,否则 download 第一步被拦。登记白名单:HKLM `...\Openness\AllowList\<exe>\Entry`(脚本 `register-openness-client.ps1`,**重建 exe 后必须重跑**)。
- 自动化代点弹窗走 L3 CU 白名单:`SEMA_CU_POPUP_{OPENNESS_ACCESS,DEVICE_TRUST,OVERWRITE_CONFIRM}` 逐类 opt-in(设置 → TIA Portal 页,物化进 .mcp.json env,新 MCP 会话生效)。CU 总闸默认禁用,每次须用户当场批准。

### F5. 其他高频报错

| 报错 | 根因 | 处置 |
|---|---|---|
| `not permitted in online mode` | 会话处于在线模式,DownloadProvider 拒下载 | 先幂等 `GoOffline` 再下 |
| fresh 工程硬件编译 3 错(访问等级/机密数据/通信证书) | 新 CPU 保护与安全默认策略 | GUI 一次性:保护与安全 → 访问等级=完全访问(无任何保护)+ 不勾选机密数据保护 |
| PLCSIM 下载报「连接到模块失败」且刚导入过 .scl | 项目未勾「块编译过程中支持仿真」 | GUI 一次性勾选;`.scl` 外部源**不支持** `{ S7_SimulationSupport := 'TRUE' }` 杂注(那是 S7DCL 头语法),别往源里塞 |
| S7 读回连不上 | CPU 未开 PUT/GET,或 DB 优化 | GUI 勾选 PUT/GET + 非优化 DB(`S7_Optimized := "FALSE"`) |
| `ready` 字段读不到(vendored 通道) | `CheckDownloadReadiness` 响应 `success` 不可靠,`ready` 在 `data.data` 下 | 按内容判定,别看 success |

## 收尾:下载成功必须读回才算完

```text
tia_call ReadPlcLiveValuesS7 {itemsJson:'["I0.0","Q0.0"]'}   ← 纯地址数组!对象数组报 Unrecognized address
```

读到值且与预期一致 = 闭环;只下不读 = 未验证。

**兜底重启纪律**:关不干净工程 / PLCSIM 实例被持(close 报 Failed closing、实例状态机卡死)时,杀 `Siemens.Automation.Portal`(及 TiaMcpServer)进程后重启,工程锁与实例持有即全部释放——比任何 close/unregister API 都可靠。杀宿主前确认它没连着 GUI TIA(会被连带挂死)。
