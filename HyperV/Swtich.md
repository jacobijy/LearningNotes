# 📘 Hyper-V 虚拟机网络配置完全指南

> 适用系统：Windows 10/11 专业版/企业版、Windows Server 2016+
> 注意：所有 PowerShell 命令请**以管理员身份运行**

---

## 1. 四种网络模式对比

| 模式 | 通信范围 | 能否上网 | 宿主机影响 | 推荐场景 |
|:---|:---|:---|:---|:---|
| **外部 (External)** | VM ↔ 局域网/互联网 | ✅ 是 | 物理网卡 IP 会迁移到虚拟网卡 | VM 需独立局域网 IP、对外提供服务 |
| **内部 (Internal)** | VM ↔ 宿主机 | ❌ 否 (需手动配 NAT) | 新增虚拟网卡，物理网络不变 | 隔离开发、固定内网段、端口映射 |
| **专用 (Private)** | VM ↔ VM | ❌ 否 | 无影响 | 纯内网集群、安全演练 |
| **Default Switch** | VM ↔ 宿主机/互联网 | ✅ 是 (系统内置 NAT) | 无影响，自动分配 IP | **日常推荐**：开箱即用、零配置 |

---

## 2. 宿主机配置 (PowerShell 管理员)

### 🟢 场景一：外部交换机 (External)
```powershell
# 1. 查看物理网卡名称
Get-NetAdapter | Where-Object {$_.Status -eq 'Up'} | Format-Table Name

# 2. 创建外部交换机（将 "Ethernet" 替换为你的物理网卡名）
# AllowManagementOS $true 确保宿主机不断网
New-VMSwitch -Name "ExtSwitch" -NetAdapterName "Ethernet" -AllowManagementOS $true

# 3. 验证
Get-VMSwitch | Format-Table Name, SwitchType
ipconfig | Select-String "vEthernet" -Context 0,3
```

### 🔵 场景二：内部交换机 + NAT (Internal + NAT)
```powershell
# 1. 创建内部交换机
New-VMSwitch -Name "IntSwitch" -SwitchType Internal

# 2. 配置宿主机虚拟网卡 IP（作为 VM 网关）
#    虚拟网卡需要作为网关设置192.168.x.1 IP
#    连接该虚拟网卡的则要设置成同网段的其他ip
$Adapter = "vEthernet (IntSwitch)"
New-NetIPAddress -InterfaceAlias $Adapter -IPAddress 192.168.100.1 -PrefixLength 24

# 3. 开启 NAT 转发
New-NetNat -Name "MyNAT" -InternalIPInterfaceAddressPrefix "192.168.100.0/24"

# 4. 验证
Get-NetIPAddress -InterfaceAlias $Adapter | Format-Table IPAddress
Get-NetNat | Format-Table Name, InternalIPInterfaceAddressPrefix
```

### 🟠 场景三：专用交换机 (Private)
```powershell
New-VMSwitch -Name "PrivSwitch" -SwitchType Private
```

### 🟡 场景四：Default Switch（微软默认）
```powershell
# 无需创建，系统已预置。直接查看：
Get-VMSwitch -Name "Default Switch"
```

---

## 3. Ubuntu 虚拟机配置

### 第一步：绑定交换机
Hyper-V 管理器 → 右键虚拟机 → 设置 → 网络适配器 → 选择对应交换机 → 应用 → 启动。

### 第二步：配置 Netplan
编辑文件：`sudo nano /etc/netplan/00-installer-config.yaml`

#### 选项 A：DHCP 自动获取（推荐 Default Switch / 外部交换机）
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp-identifier: mac
```

#### 选项 B：静态 IP（推荐内部交换机）
> 网关必须指向宿主机 vEthernet 的 IP
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.100.10/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 223.5.5.5]
```

**生效命令**：
```bash
sudo netplan generate
sudo netplan apply
ip -4 addr show eth0
ping -c 4 8.8.8.8
```

---

## 4. 端口转发（外网访问虚拟机）
适用于内部交换机+NAT场景，将宿主机端口映射到 VM：
```powershell
Add-NetNatStaticMapping -NatName "MyNAT" -Protocol TCP `
  -ExternalIPAddress 0.0.0.0 -ExternalPort 8080 `
  -InternalIPAddress 192.168.100.10 -InternalPort 80
```
访问方式：浏览器输入 `http://宿主机IP:8080` 即可访问 VM 的 80 端口。

---

## 5. 故障排查速查表

| 现象 | 原因 | 解决 |
|:---|:---|:---|
| 创建外部交换机后宿主机断网 | 驱动未加载 / IP 冲突 | `Disable-NetAdapter "vEthernet (xxx)"` 后 `Enable` |
| VM ping 不通宿主机 | 宿主机虚拟网卡无 IP | `ipconfig` 确认 `vEthernet` 有 `192.168.x.1` |
| 能 ping IP 但无法上网 | NAT 规则缺失 / DNS 错误 | `Get-NetNat` 查规则；VM 内配 DNS `8.8.8.8` |
| `Destination Host Unreachable` | ARP 解析失败 / 链路不通 | 确认 VM 绑定正确交换机；临时关闭防火墙测试 |
| Ubuntu 报错 `INCOMPLETE` | 内部交换机驱动卡死 | 重建交换机或直接改用 `Default Switch` |

---

## 6. 常用维护命令
```powershell
# 查看虚拟机网络绑定状态
Get-VMNetworkAdapter -All | Format-Table VMName, SwitchName, Connected

# 临时关闭防火墙（排查用）
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# 恢复防火墙
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# 重置网络栈（终极修复）
netsh winsock reset
netsh int ip reset
Restart-Computer
```