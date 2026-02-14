# WiFi 6 (802.11ax) Implementation in Xiaomi SDM845 Kernel

## 概述 (Overview)

本项目为小米SDM845设备实现了完整的WiFi 6 (802.11ax / HE - High Efficiency) 支持。本文档详细说明了为解决WiFi 6连接问题所做的工作。

This project implements complete WiFi 6 (802.11ax / HE - High Efficiency) support for Xiaomi SDM845 devices. This document details the work done to address WiFi 6 connection issues.

## 支持的硬件 (Supported Hardware)

本项目支持以下Qualcomm芯片组的WiFi 6功能：

The following Qualcomm chipsets have WiFi 6 support enabled:

- **QCA6290** - CONFIG_QCA6290_11AX := y
- **QCA6390** - CONFIG_WLAN_FEATURE_11AX := y  
- **QCA6490** - CONFIG_WLAN_FEATURE_11AX := y
- **QCA6750** - CONFIG_WLAN_FEATURE_11AX := y

配置文件位置：`drivers/staging/qcacld-3.0/configs/default_defconfig`

## WiFi 6核心实现 (Core WiFi 6 Implementation)

### 1. 驱动程序架构 (Driver Architecture)

WiFi 6支持主要通过以下组件实现：

#### Host Driver Layer (主机驱动层)
**文件**: `drivers/staging/qcacld-3.0/core/hdd/src/wlan_hdd_he.c`

主要功能 (Key Functions):
- `hdd_update_tgt_he_cap()` - 从固件更新HE能力
- `wlan_hdd_check_11ax_support()` - 检测beacon中的HE IE并设置802.11ax模式
- `hdd_update_he_cap_in_cfg()` - 配置UL MIMO/OFDMA能力
- `wlan_hdd_cfg80211_get_he_cap()` - 查询HE能力的vendor命令

#### Firmware Interface Layer (固件接口层)
**文件**: `drivers/staging/qcacld-3.0/core/wma/src/wma_he.c` (56KB)

关键实现 (Key Implementations):

**a) 能力处理 (Capability Processing)**
- `wma_update_target_ext_he_cap()` - 从固件提取并处理2G/5G频段的HE能力
- `wma_he_update_tgt_services()` - 启用固件的11ax服务
- `wma_get_he_capabilities()` - 向上层提供HE能力

**b) 对等端关联 (Peer Association)**  
- `wma_populate_peer_he_cap()` - 在关联期间填充对等端HE能力
- 处理MAC能力（TWT、分片、AMPDU等）
- 处理PHY能力（信道宽度、STBC、波束成形等）

**c) PPE阈值处理 (PPE Threshold Processing)**
- `wma_parse_he_ppet()` - 将固件PPE阈值格式转换为主机结构
- `wma_he_populate_ppet()` - 合并NSS/RU对的PPET8/PPET16
- 处理功率效率阈值的复杂位打包

**d) 操作管理 (Operations Management)**
- `wma_update_vdev_he_ops()` - 更新HE操作IE
- `wma_update_he_ops_ie()` - 向固件发送HE操作IE
- Debug日志函数用于能力验证

### 2. HE MAC能力支持 (HE MAC Capabilities)

```
✓ TWT (Target Wake Time) - 请求者和响应者
✓ 多TID聚合 (RX/TX)
✓ 动态分片支持
✓ OMI (操作模式指示) 控制
✓ OFDMA资源分配
✓ 全确认支持
✓ 32位BA位图
✓ BSR (Buffer Status Report)
✓ HE-LTF和GI组合
```

### 3. HE PHY能力支持 (HE PHY Capabilities)

```
✓ 信道宽度: 20/40/80/160/80+80 MHz
✓ LDPC编码
✓ HE-LTF支持: 1x, 2x, 4x
✓ GI支持: 0.8μs, 1.6μs, 3.2μs
✓ STBC TX/RX (≤80MHz和>80MHz)
✓ Doppler TX/RX
✓ UL MU-MIMO (全带宽和部分带宽)
✓ DCM (Dual Carrier Modulation) 编码/解码
✓ 波束成形 (SU/MU，多种NSTS配置)
✓ 1024-QAM支持
```

### 4. 配置参数 (Configuration Parameters)

**文件**: `drivers/staging/qcacld-3.0/components/mlme/dispatcher/inc/cfg_mlme_he_caps.h`

重要的INI配置 (Important INI Configurations):

#### MCS映射配置 (MCS Map Configuration)
```c
he_rx_mcs_map_lt_80 = 0xFFFA  // ≤80MHz接收HE-MCS映射
he_tx_mcs_map_lt_80 = 0xFFFA  // ≤80MHz传输HE-MCS映射
he_rx_mcs_map_160 = 0xFFFA    // 160MHz接收HE-MCS映射
he_tx_mcs_map_160 = 0xFFFA    // 160MHz传输HE-MCS映射
```

MCS值说明 (MCS Value Meanings):
- 0 = 支持HE-MCS 0-7
- 1 = 支持HE-MCS 0-9
- 2 = 支持HE-MCS 0-11
- 3 = 不支持此空间流

#### UL MU配置 (UL MU Configuration)
```c
he_ul_mumimo = 0           // 0:无支持 1:全带宽 2:部分带宽 3:全部和部分带宽
enable_ul_mimo = 1         // 启用上行MIMO
enable_ul_ofdma = 1        // 启用上行OFDMA
```

#### OBSS PD配置 (OBSS PD Configuration)
```c
he_sta_obsspd = 0x15b8c2ae
// Bit 7:0   - OBSS PD min (主信道): -82 dBm (0xae)
// Bit 15:8  - OBSS PD max (主信道): -62 dBm (0xc2)
// Bit 23:16 - 辅助信道Ed: -72 dBm (0xb8)
// Bit 31:24 - TX_PWR(ref): 21 dBm (0x15)
```

## WiFi 6连接流程 (WiFi 6 Connection Flow)

```
1. 固件广播11ax服务
   ↓
   wma_he_update_tgt_services() 启用服务

2. 接收目标能力
   ↓
   wma_update_target_ext_he_cap() 按频段处理

3. 检测到带HE IE的AP beacon
   ↓
   wlan_hdd_check_11ax_support() 设置11ax模式

4. 站点关联
   ↓
   wma_populate_peer_he_cap() 配置对等端能力

5. 同步HE操作
   ↓
   wma_update_he_ops_ie() 保持固件更新
```

## PPE阈值处理 (PPE Threshold Handling)

PPE (Packet Extension) 阈值对于WiFi 6性能优化至关重要：

- 支持每个NSS最多4个RU分配索引
- 支持多个NSS配置
- 处理紧凑存储的位级打包/解包
- 合并PPET8和PPET16值以获得完整的门限信息

## 已知问题和解决方案 (Known Issues and Solutions)

### Issue 1: UL MIMO/OFDMA配置冲突
**问题**: INI配置和固件能力之间的不匹配可能导致连接失败

**解决方案**: `hdd_update_he_cap_in_cfg()` 函数确保:
```c
// 仅在固件支持时启用UL MIMO
if (val & 0x1 || (val >> 1) & 0x1)
    val1 = enable_ul_mimo & 0x1;

// 仅在固件支持时启用UL OFDMA  
if ((val >> 1) & 0x1)
    val1 |= ((enable_ul_ofdma & 0x1) << 1);
```

### Issue 2: HE能力协商
**问题**: 不正确的能力交换可能导致关联失败

**解决方案**: 
- 实现按频段的能力处理（2G/5G分开）
- 正确的MAC和PHY能力打包
- PPE阈值的适当解析

### Issue 3: FTM模式兼容性
**问题**: 在工厂测试模式(FTM)下，HE命令不应执行

**解决方案**:
```c
if (QDF_GLOBAL_FTM_MODE == hdd_get_conparam()) {
    hdd_err("Command not allowed in FTM mode");
    return -EPERM;
}
```

## 调试和验证 (Debugging and Verification)

### 启用HE调试日志
驱动程序包含详细的调试日志功能：
- `wma_print_he_cap()` - 打印完整的HE能力
- `wma_print_he_mac_cap_w1()` / `wma_print_he_mac_cap_w2()` - MAC能力详情
- `wma_print_he_phy_cap()` - PHY能力详情

### 验证WiFi 6连接
```bash
# 检查HE能力是否已启用
dmesg | grep "11AX\|HE"

# 验证固件支持
dmesg | grep "DOT11AX"

# 检查关联状态
iw dev wlan0 link
```

## 兼容性 (Compatibility)

### 芯片组支持 (Chipset Support)
- ✓ QCA6290 (完全支持)
- ✓ QCA6390 (完全支持)
- ✓ QCA6490 (完全支持)
- ✓ QCA6750 (完全支持)

### Android版本 (Android Version)
- 基于Linux内核4.19
- 设计用于Android 10+（基于内核4.19的设备）
- 具体Android版本支持取决于设备制造商的实现

### AP兼容性 (AP Compatibility)
驱动程序设计为与所有标准兼容的WiFi 6 AP配合使用：
- 正确解析HE IE
- 支持所有强制性HE功能
- 可选功能的适当协商

## 技术规范参考 (Technical Specification References)

- IEEE 802.11ax-2021 标准
- Qualcomm WLAN Host Driver (qcacld-3.0)
- WMI (Wireless Module Interface) 规范
- cfg80211无线配置API

## 结论 (Conclusion)

本项目为小米SDM845设备提供了全面的WiFi 6支持实现。通过适当的能力协商、按频段配置和PPE阈值处理，驱动程序可以成功连接到WiFi 6 AP。实现包括所有必要的MAC和PHY功能，以确保与标准兼容的WiFi 6网络的互操作性。

This project provides a comprehensive WiFi 6 support implementation for Xiaomi SDM845 devices. Through proper capability negotiation, band-specific configuration, and PPE threshold handling, the driver can successfully connect to WiFi 6 APs. The implementation includes all necessary MAC and PHY features to ensure interoperability with standards-compliant WiFi 6 networks.

## 进一步阅读 (Further Reading)

- `drivers/staging/qcacld-3.0/README.txt` - 驱动程序概述
- `drivers/staging/qca-wifi-host-cmn/README.txt` - 通用WiFi组件
- IEEE 802.11ax标准文档
