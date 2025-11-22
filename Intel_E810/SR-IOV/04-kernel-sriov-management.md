# Linux Kernel SR-IOV 管理機制

## 概述

本文檔整理 Linux kernel 官方文檔中關於 SR-IOV 的管理機制，包含 PCI 層面的 API、網絡設備的特殊處理、以及 sysfs 介面的使用方式。

**文檔來源**:
- `Documentation/PCI/pci-iov-howto.rst` - SR-IOV 驅動開發指南
- `Documentation/networking/sriov.rst` - 網絡設備 SR-IOV API
- `Documentation/networking/devlink/ice.rst` - Intel E810 devlink 介面

---

## 1. SR-IOV 基本概念（Kernel 視角）

### 1.1 定義

**Single Root I/O Virtualization (SR-IOV)** 是 PCI Express 擴展能力，讓一個物理設備可以呈現為多個虛擬設備。

- **Physical Function (PF)**: 物理設備，擁有完整的控制能力
- **Virtual Function (VF)**: 虛擬設備，功能受限但可獨立運作

### 1.2 Kernel 處理方式

從 kernel 的角度，VF 被視為 **熱插拔 PCI 設備**。這意味著：

1. VF 有自己的 Bus, Device, Function Number (BDF)
2. VF 有自己的 PCI Configuration Space
3. VF 有自己的 PCI Memory Space (BAR)
4. VF driver 操作 VF 的寄存器集，就像操作真實 PCI 設備一樣

**關鍵區別**: VF 的能力範圍由 PF 通過 SR-IOV Capability 寄存器動態控制。

---

## 2. SR-IOV Kernel API

### 2.1 啟用 SR-IOV

#### 方法 1: Driver 控制（不推薦）

在 PF driver 中調用：

```c
int pci_enable_sriov(struct pci_dev *dev, int nr_virtfn);
```

**參數**:
- `dev`: PF 的 pci_dev 結構指針
- `nr_virtfn`: 要啟用的 VF 數量

**返回值**: 成功返回 0，失敗返回負數錯誤碼

**使用場景**: 舊式 driver，需要在 `probe()` 時立即啟用所有 VF

**缺點**:
- VF 數量固定（或由模塊參數決定）
- 無法動態調整
- 所有相同型號的設備都使用相同配置

#### 方法 2: sysfs 控制（推薦）

實現 `.sriov_configure` callback：

```c
static int dev_sriov_configure(struct pci_dev *dev, int numvfs)
{
    if (numvfs > 0) {
        /* 執行資源檢查和分配 */
        return pci_enable_sriov(dev, numvfs);
    }
    if (numvfs == 0) {
        /* 清理資源 */
        pci_disable_sriov(dev);
        return 0;
    }
    return -EINVAL;
}

static struct pci_driver dev_driver = {
    .name = "my-driver",
    .probe = dev_probe,
    .remove = dev_remove,
    .sriov_configure = dev_sriov_configure,  // 關鍵：註冊此callback
};
```

**用戶操作**:
```bash
# 啟用 4 個 VF
echo 4 > /sys/bus/pci/devices/<BDF>/sriov_numvfs

# 禁用所有 VF
echo 0 > /sys/bus/pci/devices/<BDF>/sriov_numvfs
```

**優點**:
- 每個 PF 可獨立配置
- 動態調整 VF 數量
- 用戶空間可控
- Kernel 自動驗證（如 numvfs <= totalvfs）

**Ice driver 實現**:
```c
// ice_sriov.c:1039
int ice_sriov_configure(struct pci_dev *pdev, int num_vfs)
{
    struct ice_pf *pf = pci_get_drvdata(pdev);

    // 檢查是否允許 SR-IOV
    if (ice_check_sriov_allowed(pf))
        return -EBUSY;

    if (!num_vfs) {
        // 禁用 VF
        if (!pci_vfs_assigned(pdev)) {
            ice_free_vfs(pf);
            return 0;
        }
        return -EBUSY;  // VF 正被 VM 使用
    }

    // 啟用 VF
    return ice_pci_sriov_ena(pf, num_vfs);
}
```

### 2.2 禁用 SR-IOV

```c
void pci_disable_sriov(struct pci_dev *dev);
```

**行為**:
1. 通知所有 VF driver 準備移除
2. 移除 VF 的 PCI 設備
3. 寫入 SR-IOV Capability 的 NumVFs = 0
4. 清除 VF Enable bit

**注意**: 如果任何 VF 被分配給 VM (passthrough)，禁用會失敗。使用 `pci_vfs_assigned()` 檢查。

### 2.3 VF Driver Auto-Probing

控制 VF 創建後是否自動加載 driver：

```bash
# 啟用 auto-probe (默認)
echo 1 > /sys/bus/pci/devices/<PF_BDF>/sriov_drivers_autoprobe

# 禁用 auto-probe
echo 0 > /sys/bus/pci/devices/<PF_BDF>/sriov_drivers_autoprobe
```

**使用場景**:
- `autoprobe=1`: 傳統用法，VF 給 host 使用
- `autoprobe=0`: VF 要 passthrough 給 VM，不需要 host driver

**重要**: 修改此選項不影響已經 probe 的 VF，只影響新創建的 VF。

---

## 3. 網絡設備 SR-IOV API

### 3.1 Legacy NDO API (已凍結)

Network Device Operations (NDO) 提供了一組 callback 用於配置 VF：

```c
struct net_device_ops {
    // VF 配置相關 callback
    int (*ndo_set_vf_mac)(struct net_device *dev, int vf, u8 *mac);
    int (*ndo_set_vf_vlan)(struct net_device *dev, int vf,
                           u16 vlan, u8 qos, __be16 proto);
    int (*ndo_set_vf_rate)(struct net_device *dev, int vf,
                           int min_tx_rate, int max_tx_rate);
    int (*ndo_set_vf_spoofchk)(struct net_device *dev, int vf, bool setting);
    int (*ndo_set_vf_trust)(struct net_device *dev, int vf, bool setting);
    int (*ndo_set_vf_link_state)(struct net_device *dev, int vf, int link_state);
    int (*ndo_get_vf_config)(struct net_device *dev, int vf,
                             struct ifla_vf_info *ivi);
    int (*ndo_get_vf_stats)(struct net_device *dev, int vf,
                            struct ifla_vf_stats *vf_stats);
};
```

**Ice driver 實現**:

```c
// ice_main.c
static const struct net_device_ops ice_netdev_ops = {
    .ndo_set_vf_mac        = ice_set_vf_mac,
    .ndo_set_vf_trust      = ice_set_vf_trust,
    .ndo_set_vf_vlan       = ice_set_vf_port_vlan,
    .ndo_set_vf_spoofchk   = ice_set_vf_spoofchk,
    .ndo_set_vf_link_state = ice_set_vf_link_state,
    .ndo_set_vf_rate       = ice_set_vf_bw,
    .ndo_get_vf_config     = ice_get_vf_cfg,
    .ndo_get_vf_stats      = ice_get_vf_stats,
};
```

**用戶空間工具**: `ip link` 使用這些 API

```bash
# 設置 VF MAC
ip link set eth0 vf 0 mac 00:11:22:33:44:55

# 設置 VF VLAN
ip link set eth0 vf 0 vlan 100

# 設置 VF 頻寬限制
ip link set eth0 vf 0 rate 1000  # 1000 Mbps

# 設置 VF 信任模式
ip link set eth0 vf 0 trust on

# 查看 VF 配置
ip link show eth0
```

### 3.2 Modern Switchdev API (推薦)

Kernel 官方文檔明確指出：

> Modern NICs are strongly encouraged to focus on implementing the switchdev
> model to configure forwarding and security of SR-IOV functionality.

**理由**:
1. Legacy API 已凍結，不接受新功能
2. Switchdev 與網絡棧其他部分整合更好
3. 提供更靈活的轉發規則配置

**Ice driver 狀態**: 目前仍使用 Legacy API，未來可能遷移到 Switchdev。

---

## 4. Sysfs 介面詳解

### 4.1 PF Sysfs 結構

```
/sys/bus/pci/devices/<PF_BDF>/
├── sriov_totalvfs        # 硬件支持的最大 VF 數量（只讀）
├── sriov_numvfs          # 當前啟用的 VF 數量（讀寫）
├── sriov_drivers_autoprobe  # 是否自動 probe VF driver（讀寫）
└── virtfn<N>/            # 指向第 N 個 VF 的符號鏈接
    └── -> ../0000:XX:YY.Z
```

**示例**:
```bash
$ cat /sys/bus/pci/devices/0000:01:00.0/sriov_totalvfs
256

$ cat /sys/bus/pci/devices/0000:01:00.0/sriov_numvfs
4

$ ls -l /sys/bus/pci/devices/0000:01:00.0/virtfn0
lrwxrwxrwx ... virtfn0 -> ../0000:01:10.0
```

### 4.2 VF Sysfs 結構

```
/sys/bus/pci/devices/<VF_BDF>/
├── physfn/               # 指向 PF 的符號鏈接
│   └-> ../0000:01:00.0
├── driver/               # VF driver (如果已加載)
├── resource              # VF 的 BAR 資源
└── ... (標準 PCI 設備屬性)
```

**檢查 VF 關係**:
```bash
# 找到 VF 對應的 PF
$ readlink /sys/bus/pci/devices/0000:01:10.0/physfn
../0000:01:00.0
```

### 4.3 網絡設備 Sysfs

```
/sys/class/net/<ethX>/device/
├── sriov_totalvfs
├── sriov_numvfs
└── virtfn<N>/
```

**創建 VF（網絡設備視角）**:
```bash
# 方法 1: 通過 PCI BDF
echo 4 > /sys/bus/pci/devices/0000:01:00.0/sriov_numvfs

# 方法 2: 通過網絡介面名稱
echo 4 > /sys/class/net/eth0/device/sriov_numvfs
```

---

## 5. Intel E810 特定功能 (Devlink)

Intel E810 driver 使用 `devlink` 框架提供額外的 SR-IOV 管理功能。

### 5.1 Devlink Port Representation

每個 VF 可以有一個 **representor port**，用於從 PF 側管理 VF 的流量。

**創建 port representor**:
```bash
# 啟用 switchdev mode
devlink dev eswitch set pci/0000:01:00.0 mode switchdev

# VF representor 自動創建為網絡介面
# eth0: PF
# eth0_0: VF 0 的 representor
# eth0_1: VF 1 的 representor
```

**用途**:
- 從 PF 側監控 VF 流量
- 設置 TC (Traffic Control) 規則
- 實現 VF 間的流量控制

### 5.2 Devlink Params

```bash
# 查看 devlink 參數
devlink dev param show pci/0000:01:00.0

# 設置 VF 相關參數（E810 特定）
devlink dev param set pci/0000:01:00.0 \
    name max_macs value 256 cmode runtime
```

---

## 6. 實際操作流程

### 6.1 創建和配置 VF

```bash
#!/bin/bash
PF_BDF="0000:01:00.0"
PF_DEV="eth0"

# 步驟 1: 檢查最大 VF 數量
MAX_VFS=$(cat /sys/bus/pci/devices/$PF_BDF/sriov_totalvfs)
echo "Max VFs: $MAX_VFS"

# 步驟 2: 創建 4 個 VF
echo 4 > /sys/bus/pci/devices/$PF_BDF/sriov_numvfs

# 步驟 3: 等待 VF 設備出現
sleep 2

# 步驟 4: 配置每個 VF
for i in {0..3}; do
    # 設置 MAC
    ip link set $PF_DEV vf $i mac 00:11:22:33:44:$(printf '%02x' $((0x50 + i)))

    # 設置 VLAN
    ip link set $PF_DEV vf $i vlan $((100 + i))

    # 設置頻寬 (1 Gbps)
    ip link set $PF_DEV vf $i rate 1000

    # 啟用信任模式
    ip link set $PF_DEV vf $i trust on

    # 啟用防欺騙檢查
    ip link set $PF_DEV vf $i spoofchk on
done

# 步驟 5: 查看配置
ip link show $PF_DEV
```

### 6.2 VF Passthrough 到 VM

```bash
#!/bin/bash
VF_BDF="0000:01:10.0"

# 步驟 1: 解除 VF driver 綁定
echo $VF_BDF > /sys/bus/pci/devices/$VF_BDF/driver/unbind

# 步驟 2: 綁定到 vfio-pci driver
echo "8086 1889" > /sys/bus/pci/drivers/vfio-pci/new_id
echo $VF_BDF > /sys/bus/pci/drivers/vfio-pci/bind

# 步驟 3: 啟動 VM (QEMU 示例)
qemu-system-x86_64 \
    -device vfio-pci,host=$VF_BDF \
    ...其他參數...
```

### 6.3 禁用 VF

```bash
# 步驟 1: 檢查是否有 VF 被分配給 VM
if [ -n "$(lspci -vvv -s <VF_BDF> | grep 'Kernel driver in use: vfio-pci')" ]; then
    echo "Error: VF is assigned to VM, cannot disable"
    exit 1
fi

# 步驟 2: 禁用所有 VF
echo 0 > /sys/bus/pci/devices/<PF_BDF>/sriov_numvfs
```

---

## 7. 錯誤處理和限制

### 7.1 常見錯誤

| 錯誤 | 原因 | 解決方法 |
|------|------|----------|
| `Device or resource busy` | VF 被 VM 使用 | 先關閉 VM 或 unbind vfio-pci |
| `Invalid argument` | numvfs > totalvfs | 檢查 sriov_totalvfs |
| `No such device` | PF driver 未實現 `.sriov_configure` | 無法通過 sysfs 啟用 |
| `Resource temporarily unavailable` | 資源不足 (MSI-X, queue) | 減少 VF 數量 |

### 7.2 Kernel 限制

```c
// include/linux/pci.h
#define PCI_SRIOV_MAX_VFS  64  // 舊版 kernel 限制

// E810 driver 限制
#define ICE_MAX_SRIOV_VFS  256  // ice 最大支持
```

實際最大 VF 數受多個因素限制：
1. 硬件能力 (`sriov_totalvfs`)
2. Driver 實現 (`ICE_MAX_SRIOV_VFS`)
3. 系統資源 (MSI-X 向量數量、記憶體)

---

## 8. Kernel 內部機制

### 8.1 PCI SR-IOV Core 流程

當調用 `pci_enable_sriov()` 時：

```
pci_enable_sriov()                    [drivers/pci/iov.c]
  ├─ sriov_enable()
  │   ├─ pci_read_config_word()       # 讀取 TotalVFs
  │   ├─ pci_write_config_word()      # 寫入 NumVFs
  │   ├─ pci_cfg_access_lock()        # 鎖定 config space
  │   ├─ sriov_init()                 # 初始化 VF 設備
  │   │   └─ pci_iov_add_virtfn()     # 為每個 VF 創建 pci_dev
  │   └─ pci_cfg_access_unlock()
  └─ kobject_uevent()                 # 通知 udev
```

### 8.2 VF 設備創建

```c
// drivers/pci/iov.c: pci_iov_add_virtfn()
static int pci_iov_add_virtfn(struct pci_dev *dev, int id)
{
    struct pci_dev *virtfn;

    // 計算 VF 的 BDF
    virtfn->devfn = pci_iov_virtfn_devfn(dev, id);
    virtfn->bus = pci_iov_virtfn_bus(dev, id);

    // 設置 physfn 指針（VF -> PF 反向指針）
    virtfn->physfn = pci_dev_get(dev);

    // 掃描和初始化 VF 設備
    pci_device_add(virtfn, virtfn->bus);

    // 創建 sysfs 鏈接
    sysfs_create_link(...);  // physfn 和 virtfn<N>

    return 0;
}
```

---

## 9. 總結

### 9.1 推薦做法

1. **Driver 開發**: 實現 `.sriov_configure` callback，不要在 `probe()` 中調用 `pci_enable_sriov()`
2. **資源管理**: 在 `.sriov_configure` 中動態計算資源分配
3. **錯誤處理**: 檢查 `pci_vfs_assigned()` 再禁用 VF
4. **用戶介面**: 通過 sysfs 控制，結合 `ip link` 配置

### 9.2 Ice Driver 符合度

| 項目 | Ice Driver 實現 | 符合推薦 |
|------|----------------|---------|
| `.sriov_configure` | ✅ 實現 | ✅ |
| 動態資源計算 | ✅ `ice_set_per_vf_res()` | ✅ |
| Legacy NDO API | ✅ 完整實現 | ⚠️ (應遷移到 switchdev) |
| Switchdev | ❌ 未實現 | ❌ |
| Devlink support | ✅ 部分實現 | ✅ |

---

**文檔版本**: v1.0
**更新日期**: 2025-11-19
**參考**: Linux kernel Documentation/PCI/pci-iov-howto.rst
