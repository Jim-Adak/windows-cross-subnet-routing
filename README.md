# Windows Server 跨子网访问异常：静态路由修复实录

> 记录日期：2026-11-04  
> 关键词：Windows Server、静态路由、CIDR、最长前缀匹配、默认网关、PowerShell  
> 结果：**已解决**

## 相关配置教程与原理分析

- [RRAS 配置 /17 路由：覆盖范围、操作步骤与 CIDR 二进制计算](docs/rras-route-17-and-cidr.md) — 新增；包含 `172.16.128.0/17`、`255.255.128.0` 与按位 AND 的详细解释。**该 /17 方案尚未在原网络实测。**
- [Windows Server 静态路由：RRAS 图形界面与 PowerShell](docs/gui-vs-cli-static-route.md) — 单网段 `/24` 路由配置。
- [静态路由、动态路由与子网规划比较](docs/static-vs-dynamic-routing.md) — OSPF 和最长前缀匹配的分析。

## 1. 故障现象

校园网中的 Windows 11 客户端无法按预期访问 Windows Server：

- Windows Server（`172.16.76.3`）主动 Ping Windows 11（`172.16.157.222`）成功；
- Windows 11 主动 Ping Windows Server 超时；
- Windows 11 对 `172.16.76.3:3389` 的 TCP 测试显示 `TcpTestSucceeded : True`；
- 同一服务器的 BMC 管理地址 `172.16.76.14` 可以访问。

这些结果说明，**不能仅凭 Ping 失败判断所有通信都被阻断**。BMC 和主机操作系统使用不同的网络管理路径，TCP 端口连通也不等于远程桌面登录必然成功。

## 2. 网络环境

| 参数 | Windows Server | Windows 11 |
| --- | --- | --- |
| IPv4 地址 | `172.16.76.3` | `172.16.157.222` |
| 子网掩码 | `255.255.0.0` | `255.255.255.0` |
| 前缀长度 | `/16` | `/24` |
| 默认网关 | `172.16.76.1` | `172.16.157.1` |
| 物理网卡 | 以太网，接口索引 `16` | WLAN |

两台设备还配置了 ZeroTier 等虚拟网络适配器，但本次修复针对的是物理网络中的 `172.16.x.x` 地址。

## 3. 排查：默认网关存在，但没有被选中

在 Windows Server 管理员 PowerShell 中执行：

```powershell
Get-NetIPConfiguration -InterfaceIndex 16
route print -4
Find-NetRoute -RemoteIPAddress 172.16.157.222
```

服务器原有默认路由：

```text
网络目标        网络掩码        网关           接口
0.0.0.0         0.0.0.0         172.16.76.1    172.16.76.3
```

同时存在一条直连路由：

```text
网络目标        网络掩码        网关           接口
172.16.0.0      255.255.0.0    在链路上       172.16.76.3
```

`Find-NetRoute` 显示，到 `172.16.157.222` 选中的是：

```text
DestinationPrefix : 172.16.0.0/16
NextHop           : 0.0.0.0
InterfaceIndex    : 16
```

### 为什么会这样？

Windows 路由选择遵循**最长前缀匹配**。对于 `172.16.157.222`，`172.16.0.0/16` 比 `0.0.0.0/0` 更具体，因此优先匹配。服务器把目标视为本地链路上的地址，通常会尝试通过 ARP 直接寻找邻居，而不是把数据交给默认网关 `172.16.76.1`。

客户端使用 `/24`，则通常认为 `172.16.76.3` 是异网段地址，向默认网关 `172.16.157.1` 发送流量。双方对子网边界的判断不同，可能导致不对称路径或 ARP 可达性问题。

**注意：** 子网掩码不一致并不必然造成故障。实际网络是否存在 Proxy ARP、VLAN 路由或 ACL 也会影响通信。本例有直接证据证明服务器选择了 `/16` 直连路由，且后续更具体的静态路由方案解决了问题。

## 4. 解决：添加更具体的静态路由

前提：`172.16.76.1` 能够转发到 `172.16.157.0/24`，且网络访问控制允许相关流量。远程修改路由前，建议保留 BMC 等带外管理方式。

在 Windows Server 的管理员 PowerShell 中添加临时路由：

```powershell
New-NetRoute `
  -DestinationPrefix "172.16.157.0/24" `
  -InterfaceIndex 16 `
  -NextHop "172.16.76.1" `
  -PolicyStore ActiveStore
```

参数说明：

| 参数 | 含义 |
| --- | --- |
| `DestinationPrefix` | 目标客户端所在网段 |
| `InterfaceIndex 16` | Windows Server 物理以太网接口 |
| `NextHop 172.16.76.1` | 下一跳网关 |
| `PolicyStore ActiveStore` | 先在活动路由表中临时测试 |

新增 `172.16.157.0/24` 路由比原有的 `172.16.0.0/16` 更具体，因此访问 `172.16.157.x` 时会优先通过 `172.16.76.1` 转发。

```text
修复前：172.16.157.222 → 命中 172.16.0.0/16 → 在链路上
修复后：172.16.157.222 → 命中 172.16.157.0/24 → 172.16.76.1
```

## 5. 验证

服务器端：

```powershell
Find-NetRoute -RemoteIPAddress 172.16.157.222
Get-NetRoute -DestinationPrefix "172.16.157.0/24"
```

预期关键字段：

```text
DestinationPrefix : 172.16.157.0/24
NextHop           : 172.16.76.1
InterfaceIndex    : 16
```

Windows 11 客户端：

```powershell
ping 172.16.76.3
Test-NetConnection 172.16.76.3 -Port 3389
mstsc /v:172.16.76.3
```

**实际结果：用户确认静态路由方案成功解决访问问题。** 不对未提供的具体 Ping 或 RDP 结果作额外推断。

如果静态路由正确而 Ping 仍失败，需要另查 Windows 防火墙的入站 ICMP 规则。TCP 3389 可连接不等于 ICMP 必然可达。

## 6. 持久化配置

临时测试成功后，可以在管理员 PowerShell 中写入持久路由：

```powershell
New-NetRoute `
  -DestinationPrefix "172.16.157.0/24" `
  -InterfaceIndex 16 `
  -NextHop "172.16.76.1" `
  -PolicyStore PersistentStore
```

也可以使用传统命令，**两种持久化方式择一**：

```cmd
route -p add 172.16.157.0 mask 255.255.255.0 172.16.76.1 if 16
```

检查配置：

```powershell
route print -4
```

## 7. 回滚

撤销临时路由：

```powershell
Remove-NetRoute `
  -DestinationPrefix "172.16.157.0/24" `
  -InterfaceIndex 16 `
  -NextHop "172.16.76.1" `
  -PolicyStore ActiveStore `
  -Confirm:$false
```

若另行创建了持久路由，确认不再需要后可删除：

```powershell
Remove-NetRoute `
  -DestinationPrefix "172.16.157.0/24" `
  -InterfaceIndex 16 `
  -NextHop "172.16.76.1" `
  -PolicyStore PersistentStore `
  -Confirm:$false
```

## 8. 经验总结

1. **不同子网可以通信**：前提是三层路由和访问控制允许。
2. **有默认网关不代表一定走默认网关**：更具体的路由优先匹配。
3. **先查看实际选路**：`route print -4` 和 `Find-NetRoute` 比猜测网关更可靠。
4. **BMC 与操作系统网络分开分析**：BMC 可达不能代替主机网络检查。
5. **Ping 和业务端口分开验证**：ICMP、TCP、RDP 身份验证是不同层次。
6. **先临时测试，再持久化**：修改远程服务器路由前保留回滚通道。

---

**一句话结论：** 本例并非缺少默认网关，而是服务器的 `172.16.0.0/16` 直连路由优先匹配了目标地址；通过为 `172.16.157.0/24` 添加经 `172.16.76.1` 转发的更具体静态路由，成功解决跨子网访问异常。
