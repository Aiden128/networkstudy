# Intel ICE SR-IOV 完整架構 (End-to-End)

> **基於 Linux Kernel ICE Driver 原始碼**
> 路徑: `drivers/net/ethernet/intel/ice/`
>
> **驗證日期**: 2025-11-14

---

## 目錄

1. [基本概念](#1-基本概念)
2. [初始化流程 (PF 端)](#2-初始化流程-pf-端)
3. [VF 端初始化](#3-vf-端初始化)
4. [Mailbox 機制 (VF ↔ PF 溝通)](#4-mailbox-機制-vf--pf-溝通)
5. [封包處理路徑 (Data Plane)](#5-封包處理路徑-data-plane)
6. [安全隔離機制](#6-安全隔離機制)
7. [進階功能：Switchdev Mode](#7-進階功能switchdev-mode-eswitch)
8. [MDD (Malicious Driver Detection)](#8-mdd-malicious-driver-detection)
9. [完整範例：端到端封包流程](#9-完整範例封包從-vm-到另一台機器)
10. [關鍵數據結構](#10-關鍵數據結構總結)
11. [Deep Dive: Scalability & Resource Management](#11-deep-dive-scalability--resource-management)

---

## 1. 基本概念

### SR-IOV 是什麼？

**Single Root I/O Virtualization** - 一張實體網卡虛擬成多張「虛擬網卡」

```
實體網卡 (PF - Physical Function)
    ├── VF 0 (Virtual Function) → 分配給 VM 1
    ├── VF 1 → 分配給 VM 2
    ├── VF 2 → 分配給 VM 3
    └── VF 3 → 分配給 Container 1
```

**關鍵術語**:

| 術語 | 說明 | 實際例子 |
|------|------|----------|
| **PF (Physical Function)** | 實體網卡，有完整管理權限 | `eth0` 在 host 上 |
| **VF (Virtual Function)** | 虛擬網卡，給 guest 使用 | `eth0` 在 VM 裡看到的 |
| **VSI (Virtual Station Interface)** | 虛擬 switch port，每個 VF 有一個 | 硬體內部概念 |
| **Mailbox** | VF 跟 PF 溝通的通道 | 類似 IPC |
| **Virtchnl** | VF-PF 溝通協定 | Intel 定義的 message format |

**為什麼需要 SR-IOV？**

傳統虛擬化：
```
VM → vSwitch (in hypervisor) → 實體 NIC
      ↑
   CPU intensive，高 latency
```

SR-IOV：
```
VM → VF (直接存取硬體) → 實體 NIC
      ↑
   接近原生效能，低 latency
```

**好處**:
- ✅ VM 直接存取硬體（bypass hypervisor）
- ✅ 接近原生網卡的效能（~95-98%）
- ✅ 減少 CPU overhead（不需要 software bridge）
- ✅ 硬體隔離（安全性）

**限制**:
- ❌ VF 數量有限（ICE 最多 256 個 VF）
- ❌ VM 遷移較複雜（需要 device detach/attach）
- ❌ VF 功能受限（某些操作需要透過 PF）

---

## 2. 初始化流程 (PF 端)

### 2.1 Enable SR-IOV (從 Host 管理員)

**方法 1: sysfs**
```bash
# 查看支援的最大 VF 數量
cat /sys/class/net/eth0/device/sriov_totalvfs
# 輸出: 256

# 啟用 4 個 VF
echo 4 > /sys/class/net/eth0/device/sriov_numvfs

# 驗證
lspci | grep -i ethernet
# 會看到 4 個新的 VF PCIe devices
```

**方法 2: ip link (with devlink)**
```bash
# 啟用 SR-IOV
ip link set eth0 vf 0 mac 02:00:00:00:00:01
ip link set eth0 vf 1 mac 02:00:00:00:00:02
ip link set eth0 vf 2 mac 02:00:00:00:00:03
ip link set eth0 vf 3 mac 02:00:00:00:00:04
```

### 2.2 Driver 處理流程

**Code**: `ice_sriov.c:30` - `ice_sriov_configure()`

```c
int ice_sriov_configure(struct pci_dev *pdev, int num_vfs)
{
    struct ice_pf *pf = pci_get_drvdata(pdev);
    struct device *dev = ice_pf_to_dev(pf);

    if (num_vfs) {
        // 啟用 num_vfs 個 VF
        dev_info(dev, "Allocating %d VFs\n", num_vfs);
        return ice_pci_sriov_ena(pf, num_vfs);
    } else {
        // num_vfs = 0，關閉所有 VF
        if (!pci_vfs_assigned(pf->pdev)) {
            ice_free_vfs(pf);
            return 0;
        } else {
            dev_err(dev, "Cannot disable SR-IOV: VFs are assigned to VMs\n");
            return -EBUSY;
        }
    }
}
```

**流程圖**:
```
echo 4 > sriov_numvfs
    ↓
Kernel 呼叫 ice_sriov_configure(pdev, 4)
    ↓
ice_pci_sriov_ena(pf, 4)
    ↓
ice_ena_vfs(pf, 4) → Sets ICE_FLAG_SRIOV_ENA
    ↓
為每個 VF 執行：
    ├─> ice_vf_alloc() - 分配 VF 結構
    ├─> ice_vf_vsi_setup() - 建立 VSI
    ├─> ice_alloc_vf_res() - 分配資源
    └─> ice_ena_vf_mappings() - 設定 HW register
    ↓
pci_enable_sriov(pdev, 4)
    ↓
Kernel 建立 4 個 VF PCIe devices
```

### 2.3 Resource Allocation for Each VF

**Code**: `ice_sriov.c:197` - `ice_vf_vsi_setup()`

Resources allocated for each VF:

```
VF #0 (vf_id = 0):
  ├── VSI (Virtual Station Interface)
  │     └── VSI index = 5 (in PF's VSI table)
  │
  ├── Queues
  │     ├── RX queues: 4 (default, configurable)
  │     ├── TX queues: 4
  │     └── Hardware queue base: 64 (in global queue space)
  │         → VF queue 0 = HW queue 64
  │         → VF queue 1 = HW queue 65
  │         → VF queue 2 = HW queue 66
  │         → VF queue 3 = HW queue 67
  │
  ├── MSI-X Vectors
  │     ├── Total: 5 (4 queue pairs + 1 mailbox)
  │     ├── Vector base: 100 (in global vector space)
  │     ├── Queue 0 → Vector 100
  │     ├── Queue 1 → Vector 101
  │     ├── Queue 2 → Vector 102
  │     ├── Queue 3 → Vector 103
  │     └── Mailbox → Vector 104
  │
  ├── MAC Address
  │     └── 02:00:00:00:00:01 (auto-generated or admin-configured)
  │
  ├── Mailbox
  │     ├── AdminQ for VF→PF communication
  │     └── Control VSI for PF→VF communication
  │
  └── Switch Rules
        ├── MAC filter: 02:00:00:00:00:01 → VSI 5
        ├── VLAN filter: (optional)
        └── Promiscuous mode: disabled (default)
```

### 2.4 Hardware Register Configuration

**Code**: `ice_sriov.c:89-125` - `ice_dis_vf_mappings()` (reverse-engineered)

```c
static void ice_ena_vf_mappings(struct ice_vf *vf)
{
    struct ice_pf *pf = vf->pf;
    struct ice_hw *hw = &pf->hw;
    int first, last, v;

    // ========== Interrupt Vector Allocation ==========
    // VPINT_ALLOC: Tell VF how many interrupt vectors it has
    wr32(hw, VPINT_ALLOC(vf->vf_id), vf->num_msix);

    // VPINT_ALLOC_PCI: PCIe interrupt allocation
    wr32(hw, VPINT_ALLOC_PCI(vf->vf_id), vf->num_msix);

    // ========== Vector Mapping ==========
    first = vf->first_vector_idx;
    last = first + vf->num_msix - 1;

    for (v = first; v <= last; v++) {
        u32 reg;

        // GLINT_VECT2FUNC: Set "which function owns vector v"
        reg = FIELD_PREP(GLINT_VECT2FUNC_VF_NUM_M, vf->vf_id) |
              FIELD_PREP(GLINT_VECT2FUNC_PF_NUM_M, hw->pf_id) |
              FIELD_PREP(GLINT_VECT2FUNC_IS_PF_M, 0);  // 0 = VF, 1 = PF
        wr32(hw, GLINT_VECT2FUNC(v), reg);
    }

    // ========== Queue Mapping ==========
    struct ice_vsi *vsi = ice_get_vf_vsi(vf);

    // Enable TX/RX queue mapping first
    wr32(hw, VPLAN_TXQ_MAPENA(vf->vf_id), VPLAN_TXQ_MAPENA_TX_ENA_M);
    wr32(hw, VPLAN_RXQ_MAPENA(vf->vf_id), VPLAN_RXQ_MAPENA_RX_ENA_M);

    if (vsi->tx_mapping_mode == ICE_VSI_MAP_CONTIG) {
        // Contiguous mode: VF queues are contiguous
        u32 reg;
        reg = FIELD_PREP(VPLAN_TX_QBASE_VFNUMQ_M, vsi->alloc_txq) |
              FIELD_PREP(VPLAN_TX_QBASE_VFQBASE_M, vsi->txq_map[0]);
        wr32(hw, VPLAN_TX_QBASE(vf->vf_id), reg);
    }

    if (vsi->rx_mapping_mode == ICE_VSI_MAP_CONTIG) {
        u32 reg;
        reg = FIELD_PREP(VPLAN_RX_QBASE_VFNUMQ_M, vsi->alloc_rxq) |
              FIELD_PREP(VPLAN_RX_QBASE_VFQBASE_M, vsi->rxq_map[0]);
        wr32(hw, VPLAN_RX_QBASE(vf->vf_id), reg);
    }
}
```

**Key Registers**:

| Register | Purpose | Example Value (VF #0) |
|----------|---------|----------------------|
| `VPINT_ALLOC(vf_id)` | VF's interrupt vector count | 5 |
| `VPINT_ALLOC_PCI(vf_id)` | PCIe interrupt configuration | 5 |
| `GLINT_VECT2FUNC(vec)` | Which VF/PF owns the vector | vf_num=0, is_pf=0 |
| `VPLAN_TX_QBASE(vf_id)` | TX queue base + count | base=64, num=4 |
| `VPLAN_RX_QBASE(vf_id)` | RX queue base + count | base=64, num=4 |

---

## 3. VF 端初始化

### 3.1 VF Driver (在 VM/Container 裡面)

VF driver 是獨立的 driver，通常叫 **iavf** (Intel Adaptive Virtual Function driver)。

**PCIe Device ID**:
- Vendor: 0x8086 (Intel)
- Device: 0x1889 (ICE VF)

### 3.2 VF Driver Initialization Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ VM/Container boot                                               │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Kernel PCI enumeration discovers VF device                      │
│   lspci shows: 00:05.0 Ethernet controller: Intel E810 VF      │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Load iavf.ko driver                                             │
│   modprobe iavf                                                 │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ iavf_probe() is called                                          │
│   • Allocate netdev structure                                   │
│   • Map BAR0 (mailbox registers)                                │
│   • Initialize AdminQ (mailbox)                                 │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Version Negotiation                                    │
│   VF → PF: VIRTCHNL_OP_VERSION                                 │
│             version = 1.1                                       │
│   PF → VF: SUCCESS, version = 1.1                              │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Request VF resources                                   │
│   VF → PF: VIRTCHNL_OP_GET_VF_RESOURCES                        │
│   PF → VF: Reply with resource list:                           │
│             {                                                   │
│               vsi_id: 5,                                        │
│               num_queue_pairs: 4,                               │
│               max_vectors: 5,                                   │
│               max_mtu: 9000,                                    │
│               vf_cap_flags: RSS | VLAN | ...,                  │
│               default_mac: 02:00:00:00:00:01,                   │
│               rss_key_size: 52,                                 │
│               rss_lut_size: 64                                  │
│             }                                                   │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Configure IRQ vectors                                  │
│   VF → PF: VIRTCHNL_OP_CONFIG_IRQ_MAP                          │
│             vector_id: 0 → queue 0,1                            │
│             vector_id: 1 → queue 2,3                            │
│             vector_id: 2 → mailbox                              │
│   PF → VF: SUCCESS                                              │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Configure RX/TX queues                                 │
│   VF → PF: VIRTCHNL_OP_CONFIG_VSI_QUEUES                       │
│             queue_pairs[0]: {                                   │
│               rxq: {vsi_id:5, queue_id:0, ring_len:512, ...}   │
│               txq: {vsi_id:5, queue_id:0, ring_len:512, ...}   │
│             }                                                   │
│             queue_pairs[1]: {...}                               │
│             ...                                                 │
│   PF → VF: SUCCESS                                              │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Enable queues                                          │
│   VF → PF: VIRTCHNL_OP_ENABLE_QUEUES                           │
│             rx_queues: bitmap 0b1111 (enable queue 0-3)         │
│             tx_queues: bitmap 0b1111                            │
│   PF → VF: SUCCESS                                              │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Link Status Notification (Passive - Event Driven)              │
│   PF → VF: VIRTCHNL_OP_EVENT (async, not requested by VF)     │
│            event = VIRTCHNL_EVENT_LINK_CHANGE                   │
│            link_event.link_speed = 25Gbps                       │
│            link_event.link_status = true                        │
│                                                                 │
│   Note: VF does NOT request this. PF sends it automatically    │
│         when link state changes or after VF initialization.    │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ NIC ready!                                                      │
│   • ifconfig eth0 shows the interface with link status         │
│   • ip link set eth0 up                                         │
│   • Can start sending/receiving packets                         │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 VF Limitations

Operations that VF driver **cannot perform directly** (must go through PF):

| Feature | VF Direct Access | Implementation Method |
|---------|-----------------|----------------------|
| Modify MAC address | ❌ | Via `VIRTCHNL_OP_ADD_ETH_ADDR` request to PF |
| Set promiscuous mode | ❌ | Via `VIRTCHNL_OP_CONFIG_PROMISCUOUS_MODE` |
| Set VLAN filter | ❌ | Via `VIRTCHNL_OP_ADD_VLAN` |
| Modify RSS key/LUT | ❌ | Via `VIRTCHNL_OP_CONFIG_RSS_KEY/LUT` |
| Set Flow Director rule | ❌ | Via `VIRTCHNL_OP_ADD_FDIR_FILTER` |
| Read statistics | ❌ | Via `VIRTCHNL_OP_GET_STATS` |
| Reset itself | ✅ | Can write to `VFGEN_RSTAT` register |
| Send/receive packets | ✅ | Directly write RX/TX descriptors |

---

## 4. Mailbox 機制 (VF ↔ PF 溝通)

### 4.1 為什麼需要 Mailbox？

**問題**: VF 不能直接存取所有硬體功能（安全考量）

**解法**: VF 透過「郵箱」向 PF 請求特權操作

**類比**:
- VF = 租客
- PF = 房東
- Mailbox = 信箱
- 租客想改門鎖（特權操作）→ 寫信給房東 → 房東批准後執行

### 4.2 Mailbox 硬體架構

```
┌─────────────────────────────────────────────────────────────────┐
│ VF (Guest)                                                      │
│                                                                 │
│   VF AdminQ (Mailbox TX side):                                 │
│     ┌────────────────────────────────────────┐                 │
│     │ AdminQ Send Ring (descriptor ring)    │                 │
│     │   - VF 寫入 descriptors                │                 │
│     │   - 每個 descriptor 指向一個 message  │                 │
│     └────────────────────────────────────────┘                 │
│              │                                                  │
│              │ (1) VF 寫 tail register                         │
│              ↓                                                  │
│     ┌────────────────────────────────────────┐                 │
│     │ VF_ATQT (Admin Transmit Queue Tail)   │                 │
│     └────────────────────────────────────────┘                 │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       │ (2) Hardware interrupt 到 PF
                       ↓
┌─────────────────────────────────────────────────────────────────┐
│ PF (Host)                                                       │
│                                                                 │
│              ┌──────────────────────────┐                       │
│              │ Mailbox Interrupt        │                       │
│              └──────────┬───────────────┘                       │
│                         │                                        │
│                         ↓                                        │
│     ┌────────────────────────────────────────┐                 │
│     │ ice_handle_vf_msg()                    │                 │
│     │   - 讀取 VF AdminQ                     │                 │
│     │   - 解析 message                       │                 │
│     │   - 呼叫對應的 handler                 │                 │
│     └────────────────────────────────────────┘                 │
│              │                                                  │
│              ↓                                                  │
│     ┌────────────────────────────────────────┐                 │
│     │ ice_vc_process_vf_msg()                │                 │
│     │ (in virt/virtchnl.c)                   │                 │
│     │   switch (v_opcode) {                  │                 │
│     │     case ADD_ETH_ADDR:                 │                 │
│     │       ice_vc_add_mac_addr_msg()        │                 │
│     │       break;                            │                 │
│     │     ...                                 │                 │
│     │   }                                     │                 │
│     └────────────────────────────────────────┘                 │
│              │                                                  │
│              │ (3) 執行操作（例如：新增 MAC filter）           │
│              ↓                                                  │
│     ┌────────────────────────────────────────┐                 │
│     │ ice_fltr_add_mac()                     │                 │
│     │   - 下載 switch rule 到 hardware      │                 │
│     └────────────────────────────────────────┘                 │
│              │                                                  │
│              │ (4) 準備 response                               │
│              ↓                                                  │
│     ┌────────────────────────────────────────┐                 │
│     │ ice_aq_send_msg_to_vf()                │                 │
│     │ (in ice_vf_mbx.c)                      │                 │
│     │   - 填寫 response message              │                 │
│     │   - 寫入 PF AdminQ                     │                 │
│     │   - Trigger VF interrupt               │                 │
│     └────────────────────────────────────────┘                 │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       │ (5) Interrupt 回 VF
                       ↓
┌─────────────────────────────────────────────────────────────────┐
│ VF (Guest)                                                      │
│                                                                 │
│     ┌────────────────────────────────────────┐                 │
│     │ Mailbox RX Interrupt                   │                 │
│     └──────────┬─────────────────────────────┘                 │
│                ↓                                                │
│     ┌────────────────────────────────────────┐                 │
│     │ iavf_handle_adminq_event()             │                 │
│     │   - 讀取 response                      │                 │
│     │   - 檢查 status: SUCCESS / ERROR       │                 │
│     │   - 完成請求                            │                 │
│     └────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Virtchnl Message 格式

**Code**: 定義在 `<linux/avf/virtchnl.h>` (kernel 共用 header)

```c
// Message header
struct virtchnl_msg {
    u32 opcode;          // 操作類型
    u32 status;          // 回覆狀態
    u32 datalen;         // payload 長度
    u8 data[];           // 實際資料 (flexible array)
};

// Opcode 範例
enum virtchnl_ops {
    VIRTCHNL_OP_UNKNOWN = 0,
    VIRTCHNL_OP_VERSION = 1,              // 版本協商
    VIRTCHNL_OP_RESET_VF = 2,             // Reset VF
    VIRTCHNL_OP_GET_VF_RESOURCES = 3,     // 取得 VF 資源
    VIRTCHNL_OP_CONFIG_TX_QUEUE = 4,      // 配置 TX queue
    VIRTCHNL_OP_CONFIG_RX_QUEUE = 5,      // 配置 RX queue
    VIRTCHNL_OP_CONFIG_VSI_QUEUES = 6,    // 配置 VSI queues
    VIRTCHNL_OP_CONFIG_IRQ_MAP = 7,       // 配置 IRQ mapping
    VIRTCHNL_OP_ENABLE_QUEUES = 8,        // 啟用 queues
    VIRTCHNL_OP_DISABLE_QUEUES = 9,       // 停用 queues
    VIRTCHNL_OP_ADD_ETH_ADDR = 10,        // 新增 MAC address
    VIRTCHNL_OP_DEL_ETH_ADDR = 11,        // 刪除 MAC address
    VIRTCHNL_OP_ADD_VLAN = 12,            // 新增 VLAN filter
    VIRTCHNL_OP_DEL_VLAN = 13,            // 刪除 VLAN filter
    VIRTCHNL_OP_CONFIG_PROMISCUOUS_MODE = 14,  // Promiscuous mode
    VIRTCHNL_OP_GET_STATS = 15,           // 取得統計數據
    VIRTCHNL_OP_EVENT = 17,               // PF 主動通知事件
    VIRTCHNL_OP_CONFIG_RSS_KEY = 23,      // 配置 RSS key
    VIRTCHNL_OP_CONFIG_RSS_LUT = 24,      // 配置 RSS LUT
    VIRTCHNL_OP_ADD_FDIR_FILTER = 47,     // 新增 FDIR rule
    VIRTCHNL_OP_DEL_FDIR_FILTER = 48,     // 刪除 FDIR rule
    // ... 更多
};

// Status codes
enum virtchnl_status_code {
    VIRTCHNL_STATUS_SUCCESS = 0,
    VIRTCHNL_STATUS_ERR_PARAM = -5,       // 參數錯誤
    VIRTCHNL_STATUS_ERR_NO_MEMORY = -18,  // 記憶體不足
    VIRTCHNL_STATUS_ERR_OPCODE_MISMATCH = -38,  // Opcode 不符
    VIRTCHNL_STATUS_ERR_NOT_SUPPORTED = -64,    // 不支援的功能
};
```

### 4.4 實際範例：VF 新增 MAC address

**VF 端 (iavf driver)**:

```c
// VF 想新增一個 unicast MAC address
int iavf_add_mac_filter(struct iavf_adapter *adapter, u8 *macaddr)
{
    struct virtchnl_ether_addr_list *veal;

    // 準備 message
    veal = kzalloc(sizeof(*veal) + sizeof(struct virtchnl_ether_addr));
    veal->vsi_id = adapter->vsi_res->vsi_id;
    veal->num_elements = 1;
    ether_addr_copy(veal->list[0].addr, macaddr);

    // 發送到 PF
    iavf_send_pf_msg(adapter,
                     VIRTCHNL_OP_ADD_ETH_ADDR,  // opcode
                     (u8 *)veal,                 // data
                     sizeof(*veal) + sizeof(struct virtchnl_ether_addr));

    // 等待 PF 回覆 (async)
    return 0;
}
```

**PF 端 (ice driver)** - `virt/virtchnl.c`:

```c
static int ice_vc_add_mac_addr_msg(struct ice_vf *vf, u8 *msg)
{
    struct virtchnl_ether_addr_list *al = (void *)msg;
    struct ice_vsi *vsi = ice_get_vf_vsi(vf);
    enum virtchnl_status_code v_ret = VIRTCHNL_STATUS_SUCCESS;
    int i;

    // 驗證 VF 權限
    if (!test_bit(ICE_VF_STATE_ACTIVE, vf->vf_states)) {
        v_ret = VIRTCHNL_STATUS_ERR_PARAM;
        goto handle_mac_exit;
    }

    // 檢查 MAC 數量限制
    if (vf->num_mac + al->num_elements > ICE_MAX_MACADDR_PER_VF) {
        v_ret = VIRTCHNL_STATUS_ERR_PARAM;
        goto handle_mac_exit;
    }

    // 為每個 MAC address 新增 filter
    for (i = 0; i < al->num_elements; i++) {
        u8 *mac_addr = al->list[i].addr;

        // Anti-spoofing check
        if (vf->pf_set_mac && !ether_addr_equal(mac_addr, vf->hw_lan_addr)) {
            v_ret = VIRTCHNL_STATUS_ERR_PARAM;
            goto handle_mac_exit;
        }

        // 下載 switch rule 到 hardware
        if (ice_fltr_add_mac(vsi, mac_addr, ICE_FWD_TO_VSI)) {
            v_ret = VIRTCHNL_STATUS_ERR_ADMIN_QUEUE_ERROR;
            goto handle_mac_exit;
        }

        vf->num_mac++;
    }

handle_mac_exit:
    // 回覆 VF
    return ice_vc_send_msg_to_vf(vf, VIRTCHNL_OP_ADD_ETH_ADDR, v_ret,
                                  NULL, 0);
}
```

### 4.5 PF 主動通知 VF (Events)

**範例：Link state change**

**Code**: `virt/virtchnl.c:79` - `ice_vc_notify_vf_link_state()`

```c
void ice_vc_notify_vf_link_state(struct ice_vf *vf)
{
    struct virtchnl_pf_event pfe = { 0 };
    struct ice_hw *hw = &vf->pf->hw;

    // 準備 event message
    pfe.event = VIRTCHNL_EVENT_LINK_CHANGE;
    pfe.severity = PF_EVENT_SEVERITY_INFO;

    if (ice_is_vf_link_up(vf)) {
        // Link UP
        ice_set_pfe_link(vf, &pfe,
                         hw->port_info->phy.link_info.link_speed,
                         true);
    } else {
        // Link DOWN
        ice_set_pfe_link(vf, &pfe, ICE_AQ_LINK_SPEED_UNKNOWN, false);
    }

    // 主動發送給 VF (不需要 VF 請求)
    ice_aq_send_msg_to_vf(hw, vf->vf_id,
                          VIRTCHNL_OP_EVENT,           // opcode = EVENT
                          VIRTCHNL_STATUS_SUCCESS,
                          (u8 *)&pfe, sizeof(pfe), NULL);
}
```

**VF 端接收**:

```c
// VF driver 收到 VIRTCHNL_OP_EVENT
void iavf_virtchnl_completion(struct iavf_adapter *adapter, ...)
{
    switch (v_opcode) {
    case VIRTCHNL_OP_EVENT:
        struct virtchnl_pf_event *pfe = (void *)msg;

        if (pfe->event == VIRTCHNL_EVENT_LINK_CHANGE) {
            // 更新 link state
            adapter->link_up = pfe->event_data.link_event.link_status;
            adapter->link_speed = pfe->event_data.link_event.link_speed;

            // 通知 kernel network stack
            if (adapter->link_up)
                netif_carrier_on(netdev);
            else
                netif_carrier_off(netdev);
        }
        break;
    }
}
```

---

## 5. Packet Processing Path (Data Plane)

### 5.1 RX Path: From Wire to VF

**Complete Flow**:

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Packet arrives from wire                                    │
│    Ethernet frame: dst_mac=02:00:00:00:00:01, VLAN=100, ...    │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. PHY/MAC Layer Reception                                     │
│    • FCS verification                                           │
│    • Preamble/SFD check                                         │
│    • Forward to Parser                                          │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. Parser (DDP-based)                                           │
│    • State machine identifies protocol layers                   │
│    • Output: ptype=0x0026 (IPv4/TCP)                            │
│    • Output: protocol offsets [(ETH,0),(IP,14),(TCP,34)]       │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. Field Vector Extraction                                      │
│    • Select FV based on ptype                                   │
│    • Extract: src_ip, dst_ip, src_port, dst_port, VLAN         │
│    • extracted[] = {192.168.1.100, 10.0.0.1, 12345, 80, 100}   │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. Classifier - **KEY: Determine packet destination**          │
│                                                                 │
│ Step 5.1: Switch Rule Lookup (MAC-based forwarding)            │
│   • Lookup switch rules for dst_mac = 02:00:00:00:00:01        │
│   • Switch rule table (maintained in ice_switch.c):            │
│       rule_id=100: dst_mac=00:50:56:aa:bb:cc                    │
│                    → FWD_TO_VSI(0) [PF]                         │
│       rule_id=101: dst_mac=02:00:00:00:00:01                    │
│                    → FWD_TO_VSI(5) [VF #0]  ← Match!           │
│       rule_id=102: dst_mac=02:00:00:00:00:02                    │
│                    → FWD_TO_VSI(6) [VF #1]                      │
│   • Result: Forward to VSI 5 (VF #0)                           │
│                                                                 │
│ Step 5.2: Port VLAN Processing (if configured - Access Mode)   │
│   • VF #0 configured with port_vlan = 100 (access port)        │
│                                                                 │
│   RX Direction (Wire → VF):                                     │
│     • Incoming packet from wire: NO VLAN tag (access port)     │
│     • Hardware automatically INSERTS VLAN tag = 100            │
│     • Packet now has: dst_mac=02:00:00:00:00:01, vlan_id=100   │
│     • If vlan_strip enabled (common):                           │
│       → Hardware STRIPS VLAN tag before forwarding to VF       │
│       → VF receives untagged packet                             │
│     • If vlan_strip disabled:                                   │
│       → VF receives packet with VLAN tag = 100                  │
│                                                                 │
│   Note: This is "Access Port" mode. In "Trunk Mode", packet    │
│         arrives WITH VLAN tag, and hardware checks against      │
│         allowed VLAN list instead of inserting.                 │
│                                                                 │
│ Step 5.3: Queue Selection within VSI                           │
│   • Check for FDIR rule match                                   │
│   • If no match → Use RSS                                       │
│                                                                 │
│   RSS Flow:                                                     │
│     hash = toeplitz(rss_key, extracted[])                       │
│          = toeplitz(key, [src_ip, dst_ip, src_port, dst_port])  │
│          = 0x12345678                                           │
│                                                                 │
│     lut_index = hash & (lut_size - 1)                           │
│               = 0x12345678 & 0x3F  (64 entries)                 │
│               = 56                                              │
│                                                                 │
│     queue_id = rss_lut[56]                                      │
│              = 2  (VF's local queue ID)                         │
│                                                                 │
│ Final decision: VSI=5, queue=2                                  │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. Queue Mapping (VSI queue → Hardware queue)                  │
│                                                                 │
│    Query VPLAN_RX_QBASE(vf_id=0):                               │
│      • vf_queue_base = 64                                       │
│      • vf_num_queues = 4                                        │
│                                                                 │
│    Calculate:                                                   │
│      hw_queue = vf_queue_base + local_queue_id                  │
│                = 64 + 2                                         │
│                = 66                                             │
│                                                                 │
│    Final: Use Hardware RX Queue #66                             │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 7. DMA Engine                                                   │
│                                                                 │
│    rx_ring = &all_rx_rings[66];  // Global queue 66            │
│                                                                 │
│    descriptor = rx_ring->desc[next_to_use];                     │
│                                                                 │
│    // DMA packet to descriptor's pkt_addr (VM's memory!)       │
│    dma_copy(packet_data,                                        │
│             descriptor->read.pkt_addr,  // DMA buffer from VM   │
│             packet_length);                                     │
│                                                                 │
│    // Fill descriptor writeback metadata                        │
│    descriptor->wb.pkt_len = packet_length;                      │
│    descriptor->wb.ptype = 0x0026;                               │
│    descriptor->wb.rss_hash = 0x12345678;                        │
│    descriptor->wb.status_error0 |= DD_BIT;  // Set last         │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 8. Trigger MSI-X Interrupt                                      │
│                                                                 │
│    Query vector mapping:                                        │
│      GLINT_VECT2FUNC(vector):                                   │
│        vector 100 → VF #0, queue 0                              │
│        vector 101 → VF #0, queue 1                              │
│        vector 102 → VF #0, queue 2  ← This one!                 │
│        vector 103 → VF #0, queue 3                              │
│                                                                 │
│    Send MSI-X interrupt #102 to **VM's vCPU**                   │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌═════════════════════════════════════════════════════════════════┐
║ Now entering VM                                                 ║
╠═════════════════════════════════════════════════════════════════╣
│ 9. VF Driver receives Interrupt (inside VM)                    │
│                                                                 │
│    Linux kernel (in VM):                                        │
│      IRQ handler → iavf_msix_clean_rings()                      │
│                 → napi_schedule()                               │
│                 → iavf_napi_poll()                              │
│                 → iavf_clean_rx_irq(rx_ring, budget)            │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 10. VF Driver processes RX Descriptor                           │
│                                                                 │
│     while (budget--) {                                          │
│         rx_desc = &rx_ring->desc[next_to_clean];                │
│                                                                 │
│         // Check DD bit                                         │
│         if (!(rx_desc->wb.status_error0 & DD_BIT))              │
│             break;  // No more packets                          │
│                                                                 │
│         dma_rmb();  // Memory barrier                           │
│                                                                 │
│         // Build SKB                                            │
│         skb = iavf_build_skb(rx_ring, rx_desc);                 │
│                                                                 │
│         // Attach metadata                                      │
│         skb->hash = rx_desc->wb.rss_hash;                       │
│         skb->ip_summed = CHECKSUM_UNNECESSARY;  // HW offload   │
│         skb->protocol = eth_type_trans(skb, netdev);            │
│                                                                 │
│         // Forward to network stack                             │
│         napi_gro_receive(napi, skb);                            │
│                                                                 │
│         next_to_clean++;                                        │
│     }                                                           │
│                                                                 │
│     // Update tail register, tell hardware descriptors reusable│
│     writel(next_to_use, rx_ring->tail);                         │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 11. VM Network Stack Processing                                │
│                                                                 │
│     netif_receive_skb(skb)                                      │
│       → ip_rcv()                                                │
│       → ip_local_deliver()                                      │
│       → tcp_v4_rcv()                                            │
│       → sock_queue_rcv_skb()                                    │
│       → application's socket buffer                             │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 12. Application receives packet                                 │
│                                                                 │
│     recv(sockfd, buffer, len, 0);                               │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 TX Path: From VF to Wire

```
┌═════════════════════════════════════════════════════════════════┐
║ VM (Guest OS)                                                   ║
╠═════════════════════════════════════════════════════════════════╣
│ 1. Application sends packet                                     │
│    send(sockfd, data, len, 0);                                  │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. VM Network Stack Processing                                 │
│    • TCP segmentation                                           │
│    • IP routing                                                 │
│    • Checksum calculation (or request HW offload)               │
│    • Build SKB                                                  │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. VF Driver (iavf) Transmit                                    │
│                                                                 │
│    iavf_xmit_frame(skb, netdev)                                 │
│    {                                                            │
│        tx_ring = &adapter->tx_rings[queue_id];                  │
│        tx_desc = &tx_ring->desc[next_to_use];                   │
│                                                                 │
│        // DMA mapping (SKB data → physical address)             │
│        dma_addr = dma_map_single(dev, skb->data, skb->len);     │
│                                                                 │
│        // Fill TX descriptor                                    │
│        tx_desc->buffer_addr = cpu_to_le64(dma_addr);            │
│        tx_desc->cmd_type_offset_bsz =                           │
│            FIELD_PREP(I40E_TXD_QW1_CMD_MASK,                    │
│                       I40E_TX_DESC_CMD_EOP |  // End of packet  │
│                       I40E_TX_DESC_CMD_RS);  // Report status   │
│                                                                 │
│        // Offload flags                                         │
│        if (skb->ip_summed == CHECKSUM_PARTIAL)                  │
│            tx_desc->cmd |= I40E_TX_DESC_CMD_IIPT_IPV4_CSUM;     │
│                                                                 │
│        // Update next_to_use                                    │
│        tx_ring->next_to_use++;                                  │
│                                                                 │
│        // Ring doorbell - notify hardware                       │
│        writel(tx_ring->next_to_use, tx_ring->tail);             │
│    }                                                            │
└────────────────┬────────────────────────────────────────────────┘
                 │ (MMIO write crosses VM boundary)
┌════════════════▼════════════════════════════════════════════════┐
║ Hardware (Intel E810)                                           ║
╠═════════════════════════════════════════════════════════════════╣
│ 4. TX Descriptor Fetch                                          │
│                                                                 │
│    // Hardware detects tail register update                     │
│    new_tail = read_tail_register(vf_queue_id);                  │
│                                                                 │
│    // Fetch descriptor from VM memory                           │
│    tx_desc = dma_read(vm_memory, descriptor_address);           │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. Security Checks (Anti-spoofing & VLAN)                       │
│                                                                 │
│    struct ice_vf *vf = get_vf_by_queue(hw_queue);               │
│                                                                 │
│    // Step 5.1: MAC Anti-Spoofing                               │
│    if (vf->spoofchk) {                                          │
│        packet_src_mac = read_mac_from_packet(packet_data);      │
│                                                                 │
│        if (!ether_addr_equal(packet_src_mac, vf->dev_lan_addr)) │
│            goto mdd_detected;  // Malicious Driver Detection    │
│    }                                                            │
│                                                                 │
│    // Step 5.2: Port VLAN Enforcement (Access Mode)             │
│    if (vf->port_vlan_info.vid != 0) {                           │
│        // In Access Port mode:                                  │
│        // - VF should send untagged packets                     │
│        // - Hardware will add port VLAN tag before egress       │
│                                                                 │
│        if (packet_has_vlan_tag(packet_data)) {                  │
│            u16 pkt_vlan = get_vlan_tag(packet_data);            │
│                                                                 │
│            // Some implementations allow VF to send tagged      │
│            // packets if tag matches port_vlan                  │
│            if (pkt_vlan != vf->port_vlan_info.vid)              │
│                goto mdd_detected;  // Wrong VLAN tag            │
│                                                                 │
│            // Strip VF's VLAN tag                               │
│            strip_vlan_tag(packet_data);                         │
│        }                                                        │
│                                                                 │
│        // Hardware automatically inserts port VLAN tag          │
│        insert_vlan_tag(packet_data, vf->port_vlan_info.vid,     │
│                        vf->port_vlan_info.qos);                 │
│    }                                                            │
│                                                                 │
│    // Step 5.3: Bandwidth Limiting                              │
│    if (vf->max_tx_rate) {                                       │
│        if (!token_bucket_allow(vf, packet_len))                 │
│            goto rate_limited;  // Drop or queue                 │
│    }                                                            │
│                                                                 │
│    goto checks_passed;                                          │
│                                                                 │
│ mdd_detected:                                                   │
│    // Record MDD event                                          │
│    vf->mdd_tx_events.count++;                                   │
│    // Trigger PF interrupt (notify PF driver)                   │
│    trigger_pf_interrupt(MDD_EVENT);                             │
│    // Drop packet                                               │
│    return;                                                      │
│                                                                 │
│ rate_limited:                                                   │
│    // Drop or queue for later                                   │
│    return;                                                      │
│                                                                 │
│ checks_passed:                                                  │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. DMA Packet Data (read from VM memory)                        │
│                                                                 │
│    packet_data = dma_read(vm_memory, tx_desc->buffer_addr,      │
│                           packet_length);                       │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 7. TX Offload Processing                                        │
│                                                                 │
│    // Checksum offload                                          │
│    if (tx_desc->cmd & I40E_TX_DESC_CMD_IIPT_IPV4_CSUM) {        │
│        compute_ipv4_checksum(packet_data);                      │
│        compute_tcp_checksum(packet_data);                       │
│    }                                                            │
│                                                                 │
│    // TSO (TCP Segmentation Offload)                            │
│    if (tx_desc->cmd & I40E_TX_DESC_CMD_TSO) {                   │
│        segment_large_packet(packet_data, mss);                  │
│    }                                                            │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 8. Send to MAC/PHY                                              │
│                                                                 │
│    // Through TX scheduler (QoS)                                │
│    // Send to physical layer                                    │
│    send_to_mac(packet_data);                                    │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 9. Packet on Wire                                               │
│    → Router → Internet                                          │
└─────────────────────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ 10. TX Completion (descriptor writeback)                        │
│                                                                 │
│     // Hardware sets descriptor done bit                        │
│     tx_desc->status |= TX_DESC_STATUS_DD;                       │
│                                                                 │
│     // If VF requested interrupt (RS bit)                       │
│     if (tx_desc->cmd & I40E_TX_DESC_CMD_RS) {                   │
│         trigger_vf_interrupt(vf, tx_queue_vector);              │
│     }                                                           │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌═════════════════════════════════════════════════════════════════┐
║ VM (Guest OS)                                                   ║
╠═════════════════════════════════════════════════════════════════╣
│ 11. VF Driver TX Completion Handler                             │
│                                                                 │
│     iavf_clean_tx_irq(tx_ring, budget)                          │
│     {                                                           │
│         while (tx_desc->status & TX_DESC_STATUS_DD) {           │
│             // Unmap DMA                                        │
│             dma_unmap_single(dev, tx_desc->buffer_addr, len);   │
│                                                                 │
│             // Free SKB                                         │
│             dev_kfree_skb_any(skb);                             │
│                                                                 │
│             // Move to next descriptor                          │
│             next_to_clean++;                                    │
│         }                                                       │
│     }                                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 安全隔離機制

### 6.1 MAC Address Anti-Spoofing

**目的**: 防止 VF 偽造 source MAC address

**Code**: `ice_sriov.h:51` - `ice_set_vf_spoofchk()`

```c
int ice_set_vf_spoofchk(struct net_device *netdev, int vf_id, bool ena)
{
    struct ice_pf *pf = ice_netdev_to_pf(netdev);
    struct ice_vf *vf;

    vf = ice_get_vf_by_id(pf, vf_id);
    if (!vf)
        return -EINVAL;

    vf->spoofchk = ena;

    // 下載 switch rule 到 hardware
    // 啟用後，hardware 會檢查每個 TX 封包的 src MAC
    ice_vsi_apply_spoofchk(vf->vsi, ena);

    return 0;
}
```

**使用方式**:
```bash
# 啟用 VF #0 的 MAC spoofing check
ip link set eth0 vf 0 spoofchk on

# 查看狀態
ip link show eth0
# vf 0 MAC 02:00:00:00:00:01, spoof checking on
```

**Hardware 實作**:

```
VF 送 TX packet (src_mac = XX:XX:XX:XX:XX:XX)
    ↓
Hardware 檢查:
    if (vf->spoofchk) {
        allowed_mac = vf->dev_lan_addr;  // 02:00:00:00:00:01

        if (packet.src_mac != allowed_mac) {
            // Spoofing detected!
            vf->mdd_tx_events.count++;
            trigger_pf_mdd_interrupt();
            drop_packet();
            return;
        }
    }
    ↓
允許送出
```

**MDD Event 處理**:

**Code**: `ice_sriov.h:62` - `ice_print_vf_tx_mdd_event()`

```c
void ice_print_vf_tx_mdd_event(struct ice_vf *vf)
{
    struct device *dev = ice_pf_to_dev(vf->pf);

    // Rate limiting: 每 30 秒最多印一次
    if (time_after_eq(jiffies, vf->mdd_tx_events.last_printed + 30 * HZ)) {
        dev_info(dev, "VF %d has caused %d Tx Malicious Driver Detection events\n",
                 vf->vf_id, vf->mdd_tx_events.count);
        vf->mdd_tx_events.last_printed = jiffies;
    }
}
```

### 6.2 VLAN Isolation (Port VLAN)

**目的**: 強制 VF 只能在特定 VLAN 通訊

**Code**: `ice_sriov.h:40` - `ice_set_vf_port_vlan()`

```c
int ice_set_vf_port_vlan(struct net_device *netdev, int vf_id,
                          u16 vlan_id, u8 qos, __be16 vlan_proto)
{
    struct ice_pf *pf = ice_netdev_to_pf(netdev);
    struct ice_vf *vf;
    int ret;

    vf = ice_get_vf_by_id(pf, vf_id);
    if (!vf)
        return -EINVAL;

    // 儲存 port VLAN 資訊
    vf->port_vlan_info.vid = vlan_id;
    vf->port_vlan_info.qos = qos;
    vf->port_vlan_info.tpid = vlan_proto;

    // 更新 VSI 的 VLAN filters
    ret = ice_vsi_manage_vlan_insertion(vf->vsi);
    if (ret)
        return ret;

    // 更新 VSI 的 VLAN stripping
    ret = ice_vsi_manage_vlan_stripping(vf->vsi);

    return ret;
}
```

**使用方式**:
```bash
# 設定 VF #0 只能在 VLAN 100, QoS priority 3
ip link set eth0 vf 0 vlan 100 qos 3

# 取消 port VLAN
ip link set eth0 vf 0 vlan 0
```

**效果**:

**RX 方向** (封包進入 VF):
```
Packet on wire (with VLAN tag 100)
    ↓
Hardware 檢查:
    if (packet.vlan_id != vf->port_vlan_info.vid) {
        drop_packet();  // 不是 VF 的 VLAN
        return;
    }
    ↓
Strip VLAN tag (VF 看不到 VLAN)
    ↓
DMA to VF (沒有 VLAN tag 的封包)
```

**TX 方向** (VF 送封包出去):
```
VF 送封包 (no VLAN tag)
    ↓
Hardware 檢查:
    if (packet has VLAN tag) {
        // VF 不能自己設 VLAN
        mdd_detected();
        drop_packet();
        return;
    }
    ↓
Insert port VLAN tag (100, QoS 3)
    ↓
Packet on wire (with VLAN tag 100)
```

**隔離效果**:
- VF #0 (VLAN 100) 只能跟 VLAN 100 的設備通訊
- VF #1 (VLAN 200) 只能跟 VLAN 200 的設備通訊
- 即使在同一台 host，不同 VLAN 的 VF 也無法直接通訊

### 6.3 Bandwidth Limiting (Rate Limiting)

**Code**: `ice_sriov.h:44` - `ice_set_vf_bw()`

```c
int ice_set_vf_bw(struct net_device *netdev, int vf_id,
                   int min_tx_rate, int max_tx_rate)
{
    struct ice_pf *pf = ice_netdev_to_pf(netdev);
    struct ice_vf *vf;
    struct ice_vsi *vsi;

    vf = ice_get_vf_by_id(pf, vf_id);
    if (!vf)
        return -EINVAL;

    vsi = ice_get_vf_vsi(vf);

    // min_tx_rate 通常不支援 (需要 ETS - Enhanced Transmission Selection)
    if (min_tx_rate) {
        dev_err(dev, "min_tx_rate is not supported\n");
        return -EOPNOTSUPP;
    }

    // 設定 max_tx_rate
    vf->max_tx_rate = max_tx_rate;

    // 配置 hardware TX scheduler
    // 使用 token bucket algorithm
    return ice_set_vsi_bw_limit(vsi, max_tx_rate);
}
```

**使用方式**:
```bash
# 限制 VF #0 的 TX 速率為 1 Gbps
ip link set eth0 vf 0 max_tx_rate 1000

# 限制為 100 Mbps
ip link set eth0 vf 0 max_tx_rate 100

# 取消限制
ip link set eth0 vf 0 max_tx_rate 0
```

**Hardware 實作 (Token Bucket)**:

```c
// 偽碼說明 token bucket algorithm
struct token_bucket {
    u64 tokens;          // 目前剩餘的 tokens
    u64 capacity;        // Bucket 容量 (burst size)
    u64 rate;            // Token 補充速率 (bits per second)
    u64 last_update;     // 上次更新時間
};

bool token_bucket_allow(struct ice_vf *vf, u32 packet_len)
{
    struct token_bucket *bucket = &vf->tx_rate_limiter;
    u64 now = get_time_ns();
    u64 elapsed = now - bucket->last_update;

    // 根據 elapsed time 補充 tokens
    u64 new_tokens = (elapsed * bucket->rate) / 1000000000;  // ns to sec
    bucket->tokens = min(bucket->tokens + new_tokens, bucket->capacity);
    bucket->last_update = now;

    // 檢查是否有足夠 tokens
    u64 tokens_needed = packet_len * 8;  // bytes to bits

    if (bucket->tokens >= tokens_needed) {
        bucket->tokens -= tokens_needed;
        return true;  // 允許送出
    } else {
        return false;  // Rate limited, drop or queue
    }
}
```

### 6.4 Trusted VF

**Code**: `ice_sriov.h:47` - `ice_set_vf_trust()`

```c
int ice_set_vf_trust(struct net_device *netdev, int vf_id, bool trusted)
{
    struct ice_pf *pf = ice_netdev_to_pf(netdev);
    struct ice_vf *vf;

    vf = ice_get_vf_by_id(pf, vf_id);
    if (!vf)
        return -EINVAL;

    vf->trusted = trusted;

    return 0;
}
```

**使用方式**:
```bash
# 設定 VF #0 為 trusted
ip link set eth0 vf 0 trust on
```

**Trusted VF 的特權**:

| 功能 | Untrusted VF | Trusted VF |
|------|--------------|------------|
| 修改 MAC address | ❌ | ✅ |
| Promiscuous mode | ❌ | ✅ (有條件) |
| Multicast promiscuous | ❌ | ✅ |
| 超過預設 MAC filter 數量 | ❌ | ✅ |
| 設定 VLAN filters | 受限 | 較寬鬆 |

**安全考量**:
- ⚠️ 只對「可信任」的 VM/container 啟用
- ⚠️ Trusted VF 可以執行某些可能影響網路的操作
- ⚠️ 雲端環境通常**不**給 tenant 啟用 trust

---

## 7. VSI 與硬體封包處理架構（基於實際 Code）

> **本章節基於 Linux Kernel ICE Driver 原始碼深入分析**
> **重點**：理解 VSI 在硬體中的作用、SR-IOV 完整生命週期、硬體封包處理路徑

---

### 7.1 VSI (Virtual Station Interface) 硬體架構

#### 7.1.1 VSI 是什麼？

**VSI 是 Intel E810 的核心抽象層**，類似 Mellanox MLX5 的 VPort 概念。每個 VSI 代表一個「虛擬網路介面」，連接到硬體內部的 Switch。

**Code**: `ice.h:264-356`

```c
struct ice_vsi {
	struct net_device *netdev;          // 對應的 Linux netdev（PF VSI 才有）
	struct ice_sw *vsw;                 // 連接到哪個 switch
	struct ice_pf *back;                // 指向 PF 結構
	struct ice_port_info *port_info;    // 實體 port 資訊

	struct ice_ring **rx_rings;         // RX ring 陣列
	struct ice_ring **tx_rings;         // TX ring 陣列
	struct ice_q_vector **q_vectors;    // Interrupt vector 陣列

	enum ice_vsi_type type;             // VSI 類型（PF/VF/CTRL）
	u16 vsi_num;                        // 硬體 VSI 編號（absolute index）
	u16 idx;                            // 軟體索引（pf->vsi[] 中的 index）

	s16 vf_id;                          // 如果是 VF VSI，對應的 VF ID

	// Queue 配置
	u16 alloc_txq;                      // 分配的 TX queue 數量
	u16 num_txq;                        // 使用中的 TX queue 數量
	u16 alloc_rxq;                      // 分配的 RX queue 數量
	u16 num_rxq;                        // 使用中的 RX queue 數量

	// RSS 配置
	u16 rss_table_size;                 // RSS hash table 大小
	u16 rss_size;                       // RSS queue 數量
	u8 *rss_hkey_user;                  // RSS hash key
	u8 *rss_lut_user;                   // RSS lookup table

	// VSI 屬性（硬體配置）
	struct ice_aqc_vsi_props info;      // 硬體 VSI 屬性
};
```

#### 7.1.2 VSI 類型

**Code**: `ice_type.h:137-142`

```c
enum ice_vsi_type {
	ICE_VSI_PF = 0,      // PF VSI（主 VSI，對應 eth0）
	ICE_VSI_VF = 1,      // VF VSI（每個 VF 一個）
	ICE_VSI_CTRL = 3,    // Control VSI（用於 FDIR，只有 1 個 queue pair）
	ICE_VSI_LB = 6,      // Loopback VSI
};
```

**VSI 用途**：

| VSI 類型 | 用途 | 數量 | 對應的 netdev |
|----------|------|------|---------------|
| `ICE_VSI_PF` | PF 的主要網路介面 | 1 per PF | `eth0` (host) |
| `ICE_VSI_VF` | 每個 VF 的網路介面 | 1 per VF | `eth0` (in VM) |
| `ICE_VSI_CTRL` | Flow Director control plane | 1 per VF (on-demand) | 無（內部使用） |
| `ICE_VSI_LB` | Loopback testing | 1 (optional) | 無 |

#### 7.1.3 VSI 與 Switch 的關係

**Code**: `ice.h:199-205`

```c
struct ice_sw {
	struct ice_pf *pf;              // 所屬的 PF
	u16 sw_id;                      // Switch ID（硬體分配）
	u16 bridge_mode;                // VEB/VEPA/Port Virtualizer
	struct ice_vsi *dflt_vsi;       // 預設 VSI（通常是 PF VSI）
	u8 dflt_vsi_ena:1;              // 是否啟用預設 VSI
};
```

**架構圖**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Ice NIC Hardware                            │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Hardware Switch (ice_sw)                                     │  │
│  │                                                               │  │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │  │
│  │   │ PF VSI   │  │ VF0 VSI  │  │ VF1 VSI  │  │ VF2 VSI  │   │  │
│  │   │ (type=0) │  │ (type=1) │  │ (type=1) │  │ (type=1) │   │  │
│  │   │ vsi_num  │  │ vsi_num  │  │ vsi_num  │  │ vsi_num  │   │  │
│  │   │   = 1    │  │   = 256  │  │   = 257  │  │   = 258  │   │  │
│  │   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │  │
│  │        │             │             │             │          │  │
│  │        │  RX/TX      │  RX/TX      │  RX/TX      │  RX/TX   │  │
│  │        │  Rings      │  Rings      │  Rings      │  Rings   │  │
│  └────────┼─────────────┼─────────────┼─────────────┼──────────┘  │
│           │             │             │             │             │
└───────────┼─────────────┼─────────────┼─────────────┼─────────────┘
            │             │             │             │
            ↓             ↓             ↓             ↓
     ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
     │   Host   │  │   VM 0   │  │   VM 1   │  │   VM 2   │
     │   eth0   │  │   eth0   │  │   eth0   │  │   eth0   │
     └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

**關鍵觀察**：
- 每個 VSI 有自己的 RX/TX queues
- VSI 透過 Hardware Switch 連接到實體 port
- PF VSI 的 `vsi_num` 通常是 1
- VF VSI 的 `vsi_num` 從 256 開始（`hw->func_caps.vf_base_id`）

#### 7.1.4 VF VSI 創建流程

**Code**: `ice_virtchnl_pf.c:825-843`

```c
static struct ice_vsi *ice_vf_vsi_setup(struct ice_vf *vf)
{
	struct ice_port_info *pi = ice_vf_get_port_info(vf);
	struct ice_pf *pf = vf->pf;
	struct ice_vsi *vsi;

	// 創建 VF VSI（類型 = ICE_VSI_VF）
	vsi = ice_vsi_setup(pf, pi, ICE_VSI_VF, vf->vf_id);

	if (!vsi) {
		dev_err(ice_pf_to_dev(pf), "Failed to create VF VSI\n");
		ice_vf_invalidate_vsi(vf);
		return NULL;
	}

	// 儲存 VSI 資訊到 VF 結構
	vf->lan_vsi_idx = vsi->idx;         // 軟體索引
	vf->lan_vsi_num = vsi->vsi_num;     // 硬體 VSI 編號

	return vsi;
}
```

**補充：Control VSI 創建** (`ice_virtchnl_pf.c:852-865`)

```c
struct ice_vsi *ice_vf_ctrl_vsi_setup(struct ice_vf *vf)
{
	struct ice_port_info *pi = ice_vf_get_port_info(vf);
	struct ice_pf *pf = vf->pf;
	struct ice_vsi *vsi;

	// 創建 Control VSI（用於 Flow Director）
	vsi = ice_vsi_setup(pf, pi, ICE_VSI_CTRL, vf->vf_id);
	if (!vsi) {
		dev_err(ice_pf_to_dev(pf), "Failed to create VF control VSI\n");
		ice_vf_ctrl_invalidate_vsi(vf);
	}

	return vsi;
}
```

---

### 7.2 SR-IOV 完整生命週期流程圖（基於實際 Code）

```
═══════════════════════════════════════════════════════════════════════════
                    STAGE 1: Driver 載入與初始化
═══════════════════════════════════════════════════════════════════════════

System Boot / Module Load
         │
         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ PCI Device Probe                                                        │
│ Code: ice_main.c - ice_probe()                                          │
│                                                                          │
│ 1. 偵測 ICE PCI device (PF)                                            │
│ 2. 分配 ice_pf 結構                                                    │
│ 3. 初始化 PCI resources (BAR, MSI-X)                                   │
│ 4. 初始化 Admin Queue (與 firmware 通訊)                               │
│ 5. 查詢 device capabilities                                            │
│     - max_vfs = pf->hw.func_caps.num_allocd_vfs (通常 256)             │
│     - vf_base_id = pf->hw.func_caps.vf_base_id (通常 256)              │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ PF VSI 創建                                                             │
│ Code: ice_lib.c - ice_vsi_setup()                                       │
│                                                                          │
│ 1. 分配 VSI 結構（type = ICE_VSI_PF）                                  │
│ 2. 分配 TX/RX queues                                                    │
│ 3. 配置 RSS (Receive Side Scaling)                                     │
│ 4. 創建對應的 netdev (eth0)                                            │
│ 5. 註冊到 Linux network stack                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ↓ Driver loaded, waiting for SR-IOV config

═══════════════════════════════════════════════════════════════════════════
                    STAGE 2: SR-IOV 啟用
═══════════════════════════════════════════════════════════════════════════

Administrator 執行:
  # echo 4 > /sys/class/net/eth0/device/sriov_numvfs
         │
         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ PCI SR-IOV Configure                                                    │
│ Code: ice_virtchnl_pf.c:2048 - ice_sriov_configure()                    │
│                                                                          │
│ int ice_sriov_configure(struct pci_dev *pdev, int num_vfs)              │
│ {                                                                        │
│     struct ice_pf *pf = pci_get_drvdata(pdev);                          │
│     int err;                                                             │
│                                                                          │
│     err = ice_check_sriov_allowed(pf);  // 檢查是否允許 SR-IOV          │
│     if (err) return err;                                                │
│                                                                          │
│     if (!num_vfs) {                                                      │
│         // === 停用 SR-IOV ===                                          │
│         ice_free_vfs(pf);                                               │
│         return 0;                                                        │
│     }                                                                    │
│                                                                          │
│     // === 啟用 SR-IOV ===                                              │
│     status = ice_mbx_init_snapshot(&pf->hw, num_vfs);  // Mailbox 初始化│
│     err = ice_pci_sriov_ena(pf, num_vfs);                               │
│     return err ? err : num_vfs;                                         │
│ }                                                                        │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │ num_vfs = 4
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ PCI SR-IOV Enable                                                       │
│ Code: ice_virtchnl_pf.c:1985 - ice_pci_sriov_ena()                      │
│                                                                          │
│ static int ice_pci_sriov_ena(struct ice_pf *pf, int num_vfs)            │
│ {                                                                        │
│     // 1. 檢查是否超過最大 VF 數量                                      │
│     if (num_vfs > pf->num_vfs_supported) {                              │
│         dev_err(dev, "Can't enable %d VFs, max VFs supported is %d\n",  │
│                 num_vfs, pf->num_vfs_supported);                        │
│         return -EOPNOTSUPP;                                             │
│     }                                                                    │
│                                                                          │
│     // 2. 執行實際的 SR-IOV 啟用                                        │
│     err = ice_ena_vfs(pf, num_vfs);                                     │
│     return err;                                                          │
│ }                                                                        │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ Enable VFs                                                              │
│ Code: ice_virtchnl_pf.c:1925 - ice_ena_vfs()                            │
│                                                                          │
│ static int ice_ena_vfs(struct ice_pf *pf, u16 num_vfs)                  │
│ {                                                                        │
│     // 1. 停用 OICR interrupt（避免處理 VFLR）                          │
│     wr32(hw, GLINT_DYN_CTL(pf->oicr_idx), ICE_ITR_NONE);                │
│     set_bit(ICE_OICR_INTR_DIS, pf->state);                              │
│                                                                          │
│     // 2. 啟用 PCI SR-IOV（硬體層面）                                   │
│     ret = pci_enable_sriov(pf->pdev, num_vfs);                          │
│     if (ret) goto err_unroll_intr;                                      │
│                                                                          │
│     // 3. 分配 VF 結構陣列                                              │
│     ret = ice_alloc_vfs(pf, num_vfs);                                   │
│     if (ret) goto err_pci_disable_sriov;                                │
│                                                                          │
│     // 4. 計算每個 VF 的資源（queues, MSI-X vectors）                   │
│     if (ice_set_per_vf_res(pf)) {                                       │
│         ret = -ENOSPC;                                                   │
│         goto err_unroll_sriov;                                          │
│     }                                                                    │
│                                                                          │
│     // 5. 設定 VF 預設值                                                │
│     ice_set_dflt_settings_vfs(pf);                                      │
│                                                                          │
│     // 6. 啟動所有 VF                                                   │
│     if (ice_start_vfs(pf)) {                                            │
│         ret = -EAGAIN;                                                   │
│         goto err_unroll_sriov;                                          │
│     }                                                                    │
│                                                                          │
│     clear_bit(ICE_VF_DIS, pf->state);                                   │
│     return 0;                                                            │
│ }                                                                        │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ↓

═══════════════════════════════════════════════════════════════════════════
                    STAGE 3: 資源分配與 VF 配置
═══════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────┐
│ 計算每個 VF 的資源                                                      │
│ Code: ice_virtchnl_pf.c:1224 - ice_set_per_vf_res()                     │
│                                                                          │
│ static int ice_set_per_vf_res(struct ice_pf *pf)                        │
│ {                                                                        │
│     // 1. 計算可用的 MSI-X vectors                                      │
│     msix_avail_for_sriov = pf->hw.func_caps.common_cap.num_msix_vectors │
│                            - pf->irq_tracker->num_entries;              │
│     msix_avail_per_vf = msix_avail_for_sriov / pf->num_alloc_vfs;       │
│                                                                          │
│     // 2. 決定每個 VF 的 MSI-X 數量（依照優先序）                       │
│     if (msix_avail_per_vf >= ICE_NUM_VF_MSIX_MED) {                     │
│         num_msix_per_vf = ICE_NUM_VF_MSIX_MED;        // 65             │
│     } else if (msix_avail_per_vf >= ICE_NUM_VF_MSIX_SMALL) {            │
│         num_msix_per_vf = ICE_NUM_VF_MSIX_SMALL;      // 17             │
│     } else if (msix_avail_per_vf >= ICE_NUM_VF_MSIX_MULTIQ_MIN) {       │
│         num_msix_per_vf = ICE_NUM_VF_MSIX_MULTIQ_MIN; // 3              │
│     } else if (msix_avail_per_vf >= ICE_MIN_INTR_PER_VF) {              │
│         num_msix_per_vf = ICE_MIN_INTR_PER_VF;        // 1              │
│     } else {                                                             │
│         return -EIO;  // 資源不足                                       │
│     }                                                                    │
│                                                                          │
│     // 3. 計算 TX/RX queue 數量                                         │
│     num_txq = ice_determine_res(pf, ice_get_avail_txq_count(pf),       │
│                                  min(num_msix_per_vf - 1,               │
│                                      ICE_MAX_RSS_QS_PER_VF));           │
│     num_rxq = ice_determine_res(pf, ice_get_avail_rxq_count(pf),       │
│                                  min(num_msix_per_vf - 1,               │
│                                      ICE_MAX_RSS_QS_PER_VF));           │
│                                                                          │
│     // 4. 儲存結果                                                      │
│     pf->num_qps_per_vf = min(num_txq, num_rxq);                         │
│     pf->num_msix_per_vf = num_msix_per_vf;                              │
│                                                                          │
│     dev_info(dev, "Enabling %d VFs with %d vectors and %d queues per VF\n",│
│              pf->num_alloc_vfs, pf->num_msix_per_vf, pf->num_qps_per_vf);│
│                                                                          │
│     return 0;                                                            │
│ }                                                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 設定 VF 預設值                                                          │
│ Code: ice_virtchnl_pf.c:1876 - ice_set_dflt_settings_vfs()              │
│                                                                          │
│ static void ice_set_dflt_settings_vfs(struct ice_pf *pf)                │
│ {                                                                        │
│     ice_for_each_vf(pf, i) {                                            │
│         struct ice_vf *vf = &pf->vf[i];                                 │
│                                                                          │
│         vf->pf = pf;                                                     │
│         vf->vf_id = i;                                  // VF ID        │
│         vf->vf_sw_id = pf->first_sw;                    // Switch ID    │
│         vf->spoofchk = true;                            // 啟用 spoofchk│
│         vf->num_vf_qs = pf->num_qps_per_vf;            // Queue pairs  │
│                                                                          │
│         // 設定 VF capabilities                                         │
│         set_bit(ICE_VIRTCHNL_VF_CAP_L2, &vf->vf_caps);                  │
│                                                                          │
│         // 設定 virtchnl opcode allowlist                               │
│         ice_vc_set_default_allowlist(vf);                               │
│                                                                          │
│         // Control VSI 初始值（尚未創建）                               │
│         ice_vf_ctrl_invalidate_vsi(vf);                                 │
│     }                                                                    │
│ }                                                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ↓

═══════════════════════════════════════════════════════════════════════════
                    STAGE 4: VF VSI 創建與硬體配置
═══════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────────┐
│ 啟動所有 VF                                                             │
│ Code: ice_virtchnl_pf.c:1836 - ice_start_vfs()                          │
│                                                                          │
│ static int ice_start_vfs(struct ice_pf *pf)                             │
│ {                                                                        │
│     ice_for_each_vf(pf, i) {                                            │
│         struct ice_vf *vf = &pf->vf[i];                                 │
│                                                                          │
│         // 1. 清除 VF reset trigger（允許 VF 啟動）                     │
│         ice_clear_vf_reset_trigger(vf);                                 │
│         //    Code: 清除 VPGEN_VFRTRIG register                         │
│                                                                          │
│         // 2. 初始化 VF VSI 資源                                        │
│         retval = ice_init_vf_vsi_res(vf);                               │
│         if (retval) goto teardown;                                      │
│                                                                          │
│         // 3. 啟用 VF MSI-X 和 Queue mapping                            │
│         ice_ena_vf_mappings(vf);                                        │
│                                                                          │
│         // 4. 設定 VF 狀態為 INIT                                       │
│         set_bit(ICE_VF_STATE_INIT, vf->vf_states);                      │
│                                                                          │
│         // 5. 通知 VF: "你現在可以初始化了"                             │
│         wr32(hw, VFGEN_RSTAT(vf->vf_id), VIRTCHNL_VFR_VFACTIVE);        │
│     }                                                                    │
│                                                                          │
│     ice_flush(hw);                                                       │
│     return 0;                                                            │
│ }                                                                        │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 初始化 VF VSI 資源                                                      │
│ Code: ice_virtchnl_pf.c:1791 - ice_init_vf_vsi_res()                    │
│                                                                          │
│ static int ice_init_vf_vsi_res(struct ice_vf *vf)                       │
│ {                                                                        │
│     // 1. 計算 VF 的第一個 MSI-X vector index                           │
│     vf->first_vector_idx = ice_calc_vf_first_vector_idx(pf, vf);        │
│     //    公式: pf->sriov_base_vector + pf->num_msix_per_vf * vf->vf_id │
│                                                                          │
│     // 2. 創建 VF VSI                                                   │
│     vsi = ice_vf_vsi_setup(vf);                                         │
│     if (!vsi) return -ENOMEM;                                           │
│                                                                          │
│     // 3. 新增 VLAN 0 filter（允許 untagged traffic）                   │
│     err = ice_vsi_add_vlan(vsi, 0, ICE_FWD_TO_VSI);                     │
│                                                                          │
│     // 4. 新增 broadcast MAC filter                                    │
│     eth_broadcast_addr(broadcast);  // ff:ff:ff:ff:ff:ff               │
│     status = ice_fltr_add_mac(vsi, broadcast, ICE_FWD_TO_VSI);          │
│                                                                          │
│     vf->num_mac = 1;  // 記錄 MAC filter 數量                           │
│     return 0;                                                            │
│ }                                                                        │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 啟用 VF MSI-X 和 Queue Mapping                                          │
│ Code: ice_virtchnl_pf.c:1080 - ice_ena_vf_mappings()                    │
│                                                                          │
│ static void ice_ena_vf_mappings(struct ice_vf *vf)                      │
│ {                                                                        │
│     struct ice_vsi *vsi = ice_get_vf_vsi(vf);                           │
│                                                                          │
│     // 1. 啟用 MSI-X mapping（見 Section 7.4）                          │
│     ice_ena_vf_msix_mappings(vf);                                       │
│                                                                          │
│     // 2. 啟用 Queue mapping（見 Section 7.4）                          │
│     ice_ena_vf_q_mappings(vf, vsi->alloc_txq, vsi->alloc_rxq);          │
│ }                                                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ↓ VF 硬體配置完成

═══════════════════════════════════════════════════════════════════════════
                    STAGE 5: VF Driver 初始化（在 VM 內）
═══════════════════════════════════════════════════════════════════════════

VF PCIe device 出現在 VM 的 PCI bus
         │
         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ VF Driver Probe (iavf driver)                                           │
│ Code: iavf_main.c - iavf_probe()                                        │
│                                                                          │
│ 1. 偵測 VF PCIe device                                                  │
│ 2. 讀取 VFGEN_RSTAT register → 狀態 = VIRTCHNL_VFR_VFACTIVE             │
│ 3. 初始化 Admin Queue (mailbox)                                         │
│ 4. 發送 VIRTCHNL_OP_VERSION 到 PF                                       │
│     ↓                                                                    │
│ 5. PF 回應 virtchnl version                                             │
│ 6. 發送 VIRTCHNL_OP_GET_VF_RESOURCES                                    │
│     ↓                                                                    │
│ 7. PF 回應 VF 的資源：                                                  │
│     - num_queue_pairs                                                    │
│     - num_vectors                                                        │
│     - vsi_id                                                             │
│     - max_mtu                                                            │
│ 8. 配置 RX/TX queues                                                    │
│ 9. 創建 netdev (eth0 in VM)                                             │
│10. 發送 VIRTCHNL_OP_ENABLE_QUEUES                                       │
│     ↓                                                                    │
│11. PF 啟用 VF 的 queues                                                 │
│12. VF 進入 running 狀態                                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                 │
                                 ↓ VF 初始化完成，可以收發封包
```

---

### 7.3 硬體封包處理路徑（RX/TX）

#### 7.3.1 E810 Packet Processing Pipeline

根據 DPDK 文檔和 code，E810 的封包處理包含以下階段：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      E810 Packet Processing Pipeline                    │
└─────────────────────────────────────────────────────────────────────────┘

封包從 Wire 進入 NIC
         │
         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ STAGE 1: Hardware Parser                                                │
│ Code: 硬體實作（可透過 DDP profile 程式化）                             │
│                                                                          │
│ 功能:                                                                    │
│ - 識別 L2/L3/L4 協議（Ethernet, VLAN, IPv4/IPv6, TCP/UDP, etc.)         │
│ - 提取關鍵欄位（MAC, IP, Port, etc.)                                    │
│ - 計算 packet hash (for RSS)                                            │
│                                                                          │
│ DDP (Dynamic Device Personalization):                                   │
│ - 允許定義新協議的 parser graph                                         │
│ - 支援 custom protocol offsets                                          │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │ Parsed metadata
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ STAGE 2: Switch Block                                                   │
│ Code: 硬體實作                                                           │
│                                                                          │
│ 功能: Exact match + Limited wildcard matching                           │
│ - 根據 VSI routing table 轉發封包                                       │
│ - 支援 VLAN/MAC filtering                                               │
│ - Large flow capacity (數千條 rules)                                    │
│                                                                          │
│ 決策:                                                                    │
│ - 封包應該送到哪個 VSI? (PF VSI / VF VSI)                               │
│ - 是否 drop? (based on filters)                                         │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │ VSI determined
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ STAGE 3: ACL (Access Control List)                                      │
│ Code: 僅在 DCF (Device Control Function) mode 啟用                      │
│                                                                          │
│ 功能: Wildcard matching with smaller flow capacity                      │
│ - 支援複雜的 match patterns                                             │
│ - 用於 Switchdev mode 的 TC offload                                     │
│ - Priority-based matching                                               │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │ (如果沒有 ACL rule match)
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ STAGE 4: FDIR (Flow Director)                                           │
│ Code: PF mode only                                                       │
│                                                                          │
│ 功能: Exact match with large flow capacity                              │
│ - Training packet based matching                                        │
│ - 精確匹配 5-tuple (src/dst IP, src/dst port, protocol)                 │
│ - Action: Queue redirect / Drop / Counter                               │
│ - Capacity: 最多 16384 filters                                          │
│                                                                          │
│ 使用案例:                                                                │
│ - 將特定 flow 導向特定 RX queue                                         │
│ - Application-specific QoS                                              │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │ Queue determined (or use default RSS)
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ STAGE 5: RSS (Receive Side Scaling)                                     │
│ Code: ice_lib.c - ice_vsi_cfg_rss_lut_key()                             │
│                                                                          │
│ 功能: 將封包分散到多個 RX queues                                        │
│ - Hash function: Toeplitz hash                                          │
│ - Hash input: 可配置（src/dst IP, src/dst port, etc.)                   │
│ - Lookup table: 將 hash value 映射到 queue ID                           │
│                                                                          │
│ 公式: queue_id = rss_lut[hash(packet) % rss_table_size]                 │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │ Final queue determined
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ STAGE 6: RX Queue & DMA                                                 │
│ Code: ice_txrx.c - ice_clean_rx_irq()                                   │
│                                                                          │
│ 1. 封包寫入 RX descriptor ring                                          │
│ 2. DMA 封包到 host memory (RX buffer)                                   │
│ 3. 產生 MSI-X interrupt                                                 │
│ 4. Driver 處理 interrupt (NAPI poll)                                    │
│ 5. 建立 sk_buff 並送到 network stack                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 7.3.2 RX Path: Wire → VF 的完整流程

```
Physical Wire
      │
      ↓ Packet arrives
┌──────────────────────────────────────────────────────────────────────────┐
│ NIC Hardware                                                             │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Parser: 解析協議 (L2: Ethernet, L3: IPv4, L4: TCP)            │     │
│  │ 提取: dst_mac=02:00:00:00:00:01, dst_ip=10.0.0.2, dst_port=80 │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
│                            │                                             │
│                            ↓                                             │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Switch: 查詢 MAC table                                         │     │
│  │ dst_mac=02:00:00:00:00:01 → VF #0 (VSI 256)                    │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
│                            │                                             │
│                            ↓                                             │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ VF #0 FDIR: 是否有 flow rule match?                           │     │
│  │ Match: dst_ip=10.0.0.2, dst_port=80 → Queue 2                  │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
│                            │ (如果沒有 FDIR rule)                        │
│                            ↓                                             │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ RSS: hash(src_ip, dst_ip, src_port, dst_port)                 │     │
│  │ hash = 0x12345678                                              │     │
│  │ queue_id = rss_lut[0x12345678 % 16] = 3                        │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
│                            │                                             │
│                            ↓                                             │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ RX Queue 3 (VF #0)                                             │     │
│  │ - 寫入 RX descriptor                                           │     │
│  │ - DMA 封包到 VM memory                                         │     │
│  │ - 產生 MSI-X interrupt                                         │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
└──────────────────────────────┼────────────────────────────────────────────┘
                               │ Interrupt
                               ↓
┌──────────────────────────────────────────────────────────────────────────┐
│ VM (VF #0)                                                               │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ iavf driver: ice_clean_rx_irq()                                │     │
│  │ - 處理 RX descriptor                                           │     │
│  │ - 建立 sk_buff                                                 │     │
│  │ - 送到 network stack (netif_receive_skb)                       │     │
│  └────────────────────────┬───────────────────────────────────────┘     │
│                           │                                              │
│                           ↓                                              │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Application 收到封包                                           │     │
│  └────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────┘
```

#### 7.3.3 TX Path: VF → Wire 的完整流程

```
┌──────────────────────────────────────────────────────────────────────────┐
│ VM (VF #0)                                                               │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Application 發送封包                                           │     │
│  │ socket.send(data)                                              │     │
│  └────────────────────────┬───────────────────────────────────────┘     │
│                           │                                              │
│                           ↓                                              │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Network Stack                                                  │     │
│  │ - 建立 sk_buff                                                 │     │
│  │ - Routing decision                                             │     │
│  │ - 呼叫 netdev->ndo_start_xmit                                  │     │
│  └────────────────────────┬───────────────────────────────────────┘     │
│                           │                                              │
│                           ↓                                              │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ iavf driver: iavf_xmit_frame()                                 │     │
│  │ - 選擇 TX queue (based on CPU)                                 │     │
│  │ - 填寫 TX descriptor                                           │     │
│  │ - Ring doorbell (通知 NIC)                                     │     │
│  └────────────────────────┬───────────────────────────────────────┘     │
└──────────────────────────────┼────────────────────────────────────────────┘
                               │ TX descriptor + doorbell
                               ↓
┌──────────────────────────────────────────────────────────────────────────┐
│ NIC Hardware                                                             │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ TX Queue (VF #0)                                               │     │
│  │ - 讀取 TX descriptor                                           │     │
│  │ - DMA 封包從 VM memory                                         │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
│                            │                                             │
│                            ↓                                             │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ TX Pipeline                                                    │     │
│  │ - Checksum offload (如果啟用)                                  │     │
│  │ - TSO (TCP Segmentation Offload)                               │     │
│  │ - VLAN insertion (如果設定 Port VLAN)                          │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
│                            │                                             │
│                            ↓                                             │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Switch: VF #0 (VSI 256) → Physical Port                        │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
│                            │                                             │
│                            ↓                                             │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │ Physical MAC                                                   │     │
│  │ - Add preamble                                                 │     │
│  │ - FCS (Frame Check Sequence)                                   │     │
│  └─────────────────────────┬──────────────────────────────────────┘     │
└──────────────────────────────┼────────────────────────────────────────────┘
                               │
                               ↓
                         Physical Wire
```

---

### 7.4 MSI-X 和 Queue 的硬體映射

#### 7.4.1 MSI-X Vector 映射

**Code**: `ice_virtchnl_pf.c:982-1024`

```c
static void ice_ena_vf_msix_mappings(struct ice_vf *vf)
{
	int device_based_first_msix, device_based_last_msix;
	int pf_based_first_msix, pf_based_last_msix, v;
	struct ice_pf *pf = vf->pf;
	int device_based_vf_id;
	struct ice_hw *hw;
	u32 reg;

	hw = &pf->hw;

	// 1. 計算 VF 的 MSI-X vector 範圍（PF-based index）
	pf_based_first_msix = vf->first_vector_idx;
	pf_based_last_msix = (pf_based_first_msix + pf->num_msix_per_vf) - 1;

	// 2. 轉換為 device-based index
	//    device_based = pf_based + global_offset
	device_based_first_msix = pf_based_first_msix +
		pf->hw.func_caps.common_cap.msix_vector_first_id;
	device_based_last_msix =
		(device_based_first_msix + pf->num_msix_per_vf) - 1;
	device_based_vf_id = vf->vf_id + hw->func_caps.vf_base_id;

	// 3. 配置 VPINT_ALLOC register
	//    告訴硬體：VF X 可以使用哪些 MSI-X vectors
	reg = (((device_based_first_msix << VPINT_ALLOC_FIRST_S) &
		VPINT_ALLOC_FIRST_M) |
	       ((device_based_last_msix << VPINT_ALLOC_LAST_S) &
		VPINT_ALLOC_LAST_M) | VPINT_ALLOC_VALID_M);
	wr32(hw, VPINT_ALLOC(vf->vf_id), reg);

	// 4. 配置 VPINT_ALLOC_PCI register
	//    PCI MSI-X capability 的配置
	reg = (((device_based_first_msix << VPINT_ALLOC_PCI_FIRST_S)
		 & VPINT_ALLOC_PCI_FIRST_M) |
	       ((device_based_last_msix << VPINT_ALLOC_PCI_LAST_S) &
		VPINT_ALLOC_PCI_LAST_M) | VPINT_ALLOC_PCI_VALID_M);
	wr32(hw, VPINT_ALLOC_PCI(vf->vf_id), reg);

	// 5. 配置 GLINT_VECT2FUNC register（每個 vector 一個）
	//    將 vector 映射到特定的 PF/VF
	for (v = pf_based_first_msix; v <= pf_based_last_msix; v++) {
		reg = (((device_based_vf_id << GLINT_VECT2FUNC_VF_NUM_S) &
			GLINT_VECT2FUNC_VF_NUM_M) |
		       ((hw->pf_id << GLINT_VECT2FUNC_PF_NUM_S) &
			GLINT_VECT2FUNC_PF_NUM_M));
		wr32(hw, GLINT_VECT2FUNC(v), reg);
	}

	// 6. 配置 Mailbox interrupt (vector 0)
	wr32(hw, VPINT_MBX_CTL(device_based_vf_id), VPINT_MBX_CTL_CAUSE_ENA_M);
}
```

**關鍵 Registers**：

| Register | 作用 | 範例值 (VF #0, 假設 4 vectors) |
|----------|------|-------------------------------|
| `VPINT_ALLOC` | VF 可用的 MSI-X 範圍 | first=256, last=259, valid=1 |
| `VPINT_ALLOC_PCI` | PCI MSI-X 配置 | first=256, last=259, valid=1 |
| `GLINT_VECT2FUNC(256)` | Vector 256 → VF #0 | vf_num=256, pf_num=0 |
| `GLINT_VECT2FUNC(257)` | Vector 257 → VF #0 | vf_num=256, pf_num=0 |
| `GLINT_VECT2FUNC(258)` | Vector 258 → VF #0 | vf_num=256, pf_num=0 |
| `GLINT_VECT2FUNC(259)` | Vector 259 → VF #0 | vf_num=256, pf_num=0 |
| `VPINT_MBX_CTL` | Mailbox interrupt enable | cause_ena=1 |

#### 7.4.2 TX/RX Queue 映射

**Code**: `ice_virtchnl_pf.c:1032-1075`

```c
static void ice_ena_vf_q_mappings(struct ice_vf *vf, u16 max_txq, u16 max_rxq)
{
	struct ice_vsi *vsi = ice_get_vf_vsi(vf);
	struct ice_hw *hw = &vf->pf->hw;
	u32 reg;

	// ==================== TX Queue Mapping ====================

	// 1. 啟用 TX queue mapping
	wr32(hw, VPLAN_TXQ_MAPENA(vf->vf_id), VPLAN_TXQ_MAPENA_TX_ENA_M);

	// 2. 配置 TX queue base 和數量
	if (vsi->tx_mapping_mode == ICE_VSI_MAP_CONTIG) {
		// Contiguous mode: VF 的 queues 是連續的
		// VFNUMQ: queue 數量 - 1 (0 表示 1 個 queue, 255 表示 256 個)
		reg = (((vsi->txq_map[0] << VPLAN_TX_QBASE_VFFIRSTQ_S) &
			VPLAN_TX_QBASE_VFFIRSTQ_M) |
		       (((max_txq - 1) << VPLAN_TX_QBASE_VFNUMQ_S) &
			VPLAN_TX_QBASE_VFNUMQ_M));
		wr32(hw, VPLAN_TX_QBASE(vf->vf_id), reg);
	} else {
		// Scattered mode: 不支援（需要逐一配置每個 queue）
		dev_err(dev, "Scattered mode for VF Tx queues is not yet implemented\n");
	}

	// ==================== RX Queue Mapping ====================

	// 3. 啟用 RX queue mapping
	wr32(hw, VPLAN_RXQ_MAPENA(vf->vf_id), VPLAN_RXQ_MAPENA_RX_ENA_M);

	// 4. 配置 RX queue base 和數量
	if (vsi->rx_mapping_mode == ICE_VSI_MAP_CONTIG) {
		reg = (((vsi->rxq_map[0] << VPLAN_RX_QBASE_VFFIRSTQ_S) &
			VPLAN_RX_QBASE_VFFIRSTQ_M) |
		       (((max_rxq - 1) << VPLAN_RX_QBASE_VFNUMQ_S) &
			VPLAN_RX_QBASE_VFNUMQ_M));
		wr32(hw, VPLAN_RX_QBASE(vf->vf_id), reg);
	} else {
		dev_err(dev, "Scattered mode for VF Rx queues is not yet implemented\n");
	}
}
```

**關鍵 Registers**：

| Register | 作用 | 範例值 (VF #0, 4 queue pairs) |
|----------|------|-------------------------------|
| `VPLAN_TXQ_MAPENA` | TX queue mapping enable | tx_ena=1 |
| `VPLAN_TX_QBASE` | TX queue 起始 + 數量 | first_q=256, num_q=3 (表示 4 個) |
| `VPLAN_RXQ_MAPENA` | RX queue mapping enable | rx_ena=1 |
| `VPLAN_RX_QBASE` | RX queue 起始 + 數量 | first_q=256, num_q=3 (表示 4 個) |

**Queue 編號規則**：
- PF queues: 0-255
- VF #0 queues: 256-259 (假設 4 個 queue pairs)
- VF #1 queues: 260-263
- VF #2 queues: 264-267
- ...

---

### 7.5 與 MLX5 架構對比

| Feature | Intel ICE (E810) | Mellanox MLX5 (ConnectX-5/6) |
|---------|------------------|------------------------------|
| **Virtual Port 抽象** | VSI (Virtual Station Interface)<br>Code: `ice.h:264` | VPort<br>Code: `mlx5/eswitch.h:179` |
| **每個 VF 的虛擬 port** | 1 VSI per VF<br>`ice_vf_vsi_setup()` | 1 VPort per VF<br>`mlx5_esw_vport_enable()` |
| **Switch 機制** | Hardware Switch (`ice_sw`)<br>- VEB/VEPA mode<br>- MAC/VLAN filtering | E-Switch (Embedded Switch)<br>- FDB (Forwarding Database)<br>- STE hash table lookup |
| **Switchdev 支援** | ✅ 有限支援<br>- 需要啟用 eSwitch mode<br>- 透過 Control VSI<br>- Code: `ice_repr.c` | ✅ 完整支援<br>- OFFLOADS mode<br>- 每個 VPort 一個 representor<br>- Code: `mlx5/eswitch_offloads.c` |
| **Representor 實作** | Control VSI (type=CTRL)<br>- 1 個 Control VSI per VF (on-demand)<br>- 用於 TC offload | VF Representor netdev<br>- 每個 VPort 一個獨立 netdev<br>- 直接下 TC rules |
| **Parser 可程式化** | ✅ DDP (Dynamic Device Personalization)<br>- 可定義新協議<br>- 可修改 parser graph<br>- 需要 DDP package file | ⚠️ 有限<br>- 大部分協議固定在硬體<br>- FLEX_PARSER 支援有限自定義<br>- Code: `mlx5/steering/dr_ste_v1.c:50-51` |
| **Flow Steering 機制** | 多層 Pipeline:<br>1. Switch (exact + limited wildcard)<br>2. ACL (wildcard, DCF mode)<br>3. FDIR (exact match, 16K rules)<br>4. RSS | STE Hash Table:<br>- CRC32 hash<br>- Collision handling<br>- lu_type based lookup<br>- Code: `mlx5/steering/dr_ste.c` |
| **封包轉發決策** | 基於 VSI routing:<br>- MAC table lookup<br>- VLAN filtering<br>- Switch block決定目標 VSI | 基於 FDB lookup:<br>- TC Chains (fast path)<br>- Slow Path FDB<br>- Source SQN + metadata |
| **硬體資源管理** | 明確的 register 配置:<br>- `VPINT_ALLOC` (MSI-X)<br>- `VPLAN_TXQ_MAPENA` (Queues)<br>- `GLINT_VECT2FUNC` (Mapping) | Firmware 管理:<br>- `mlx5_cmd_exec()` 命令<br>- Firmware 分配資源<br>- Driver 查詢 capabilities |
| **VF-to-VF 通訊** | 透過 Hardware Switch<br>- 需要 switch 支援<br>- 可能經過 wire (VEPA mode) | Hairpin (zero-copy):<br>- VF TX → 硬體 → VF RX<br>- 不經過 host memory<br>- Code: `mlx5/transobj.c:391-425` |
| **最大 VF 數量** | 256 VFs<br>Code: `hw->func_caps.num_allocd_vfs` | 127 VFs (per PF)<br>Code: `MLX5_MAX_VF_PORTS` |
| **VSI/VPort 總數** | 768 VSIs<br>(根據 E810 datasheet) | 取決於 firmware<br>可達 1024+ vports |
| **Flow Table 架構** | Flat pipeline:<br>Switch → ACL → FDIR → RSS | Hierarchical:<br>Namespace → Priority → Table → Group → Entry |
| **Mailbox 通訊** | Admin Queue (virtchnl protocol)<br>Code: `ice_virtchnl_pf.c` | Control Queue (HCA commands)<br>Code: `mlx5/vport.c` |

**架構哲學對比**：

**Intel ICE**:
- **明確的硬體抽象**：VSI 是清晰的硬體概念，直接對應 register 配置
- **Pipeline-based**：封包依序經過 Parser → Switch → ACL → FDIR → RSS
- **可程式化 Parser**：透過 DDP 允許深度客製化協議解析
- **Register-level 控制**：Driver 直接配置硬體 registers（`VPINT_*`, `VPLAN_*`）

**Mellanox MLX5**:
- **Firmware 抽象**：大部分硬體細節由 firmware 管理
- **FDB-based**：使用 Forwarding Database 和 hash table lookup
- **固定 Parser**：大部分協議內建於硬體，僅有限度可程式化
- **Command-based 控制**：透過 firmware commands 配置硬體

**何時選擇 ICE？**
- 需要深度客製化協議解析（DDP）
- 需要精細的 register-level 控制
- 預算考量（Intel 通常較便宜）

**何時選擇 MLX5？**
- 需要完整的 Switchdev/OVS offload
- 需要 Hairpin（VF-to-VF zero-copy）
- 需要更高的靈活性和 scalability

---

**下一章節**: [Section 8: Switchdev Mode (eSwitch) 進階功能](#8-switchdev-mode-eswitch-進階功能)
## 8. 進階功能：Switchdev Mode (eSwitch)

### 11.1 傳統 Mode vs Switchdev Mode

**傳統 SR-IOV Mode (Legacy)**:

```
┌──────────────────────────────────────────────┐
│ Host (PF)                                    │
│   ice driver 管理 VF                         │
│   VF ↔ PF 透過 mailbox 溝通                  │
│   封包路徑: VF ↔ Hardware ↔ Wire            │
└──────────────────────────────────────────────┘
                  ↕
┌──────────────────────────────────────────────┐
│ VM 1 (VF #0)                                 │
└──────────────────────────────────────────────┘
```

**限制**:
- ❌ Host 看不到 VF 的流量
- ❌ 無法做 VF-to-VF forwarding (不經過 wire)
- ❌ 難以整合 OVS/eBPF

**Switchdev Mode**:

```
┌──────────────────────────────────────────────────────────────────┐
│ Host (PF)                                                        │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Linux Kernel eSwitch (虛擬 switch)                         │ │
│  │                                                             │ │
│  │   eth0 (PF uplink)                                         │ │
│  │     │                                                       │ │
│  │     ├─── eth0_vf0 (VF #0 representor)                      │ │
│  │     ├─── eth0_vf1 (VF #1 representor)                      │ │
│  │     └─── eth0_vf2 (VF #2 representor)                      │ │
│  │                                                             │ │
│  │   可以用 tc/OVS 設定 flow rules                             │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
                  ↕                    ↕                   ↕
┌─────────────────────┐  ┌─────────────────┐  ┌──────────────────┐
│ VM 1 (VF #0)        │  │ VM 2 (VF #1)    │  │ VM 3 (VF #2)     │
└─────────────────────┘  └─────────────────┘  └──────────────────┘
```

**好處**:
- ✅ Host 可以看到 VF 流量（透過 representor）
- ✅ 可以用 tc/OVS 設定 offload rules
- ✅ 支援 VF-to-VF 內部轉發
- ✅ 整合 SDN 控制

### 11.2 啟用 Switchdev Mode

**Code**: `ice_eswitch.c`

```bash
# 檢查當前 mode
devlink dev eswitch show pci/0000:18:00.0
# mode legacy

# 切換到 switchdev mode
devlink dev eswitch set pci/0000:18:00.0 mode switchdev

# 啟用 VF
echo 4 > /sys/class/net/eth0/device/sriov_numvfs

# 檢查 representor netdevs
ip link show | grep eth0_vf
# eth0_vf0: <BROADCAST,MULTICAST> ...
# eth0_vf1: <BROADCAST,MULTICAST> ...
# eth0_vf2: <BROADCAST,MULTICAST> ...
# eth0_vf3: <BROADCAST,MULTICAST> ...
```

### 11.3 Representor 的概念

**Representor** = VF 在 host 上的「代表」

```
VM 裡的 VF netdev (eth0)  ←→  Host 上的 representor (eth0_vf0)
       │                                │
       │                                │
       └────────── 同一個 VSI ──────────┘
```

**從 VM 送出的封包**:
```
VM eth0 (VF) 送封包
    ↓
Hardware VSI
    ↓
    ├─> 送到 wire (normal path)
    └─> 複製一份到 representor (if tc rules 設定)
```

**從 representor 送的封包**:
```
Host eth0_vf0 (representor) 送封包
    ↓
Hardware VSI
    ↓
送入 VF (VM 收到)
```

### 11.4 TC Offload 範例

**範例 1: 鏡像 VF 流量到 host**

```bash
# 把 VF #0 的所有流量 mirror 到 host interface
tc qdisc add dev eth0_vf0 ingress
tc filter add dev eth0_vf0 ingress \
    protocol all \
    flower skip_sw \
    action mirred egress mirror dev eth0
```

**範例 2: VF-to-VF 轉發 (不經過 wire)**

```bash
# VF #0 → VF #1 的流量直接在 hardware 轉發
tc filter add dev eth0_vf0 ingress \
    protocol ip \
    flower skip_sw \
    dst_ip 10.0.0.2 \
    action mirred egress redirect dev eth0_vf1
```

**範例 3: 基於 5-tuple 的 drop**

```bash
# Drop 特定 src IP 的流量
tc filter add dev eth0_vf0 ingress \
    protocol ip \
    flower skip_sw \
    src_ip 192.168.1.100 \
    ip_proto tcp \
    dst_port 22 \
    action drop
```

**`skip_sw` 的意義**:
- `skip_sw`: 只用 hardware offload (不走 software datapath)
- 如果 hardware 不支援 → rule 安裝失敗
- 確保 line-rate forwarding

### 11.5 OVS Integration

**搭配 Open vSwitch**:

```bash
# 建立 OVS bridge
ovs-vsctl add-br br0

# 加入 representors
ovs-vsctl add-port br0 eth0_vf0
ovs-vsctl add-port br0 eth0_vf1
ovs-vsctl add-port br0 eth0_vf2

# 加入 uplink (PF)
ovs-vsctl add-port br0 eth0

# 啟用 hardware offload
ovs-vsctl set Open_vSwitch . other_config:hw-offload=true
systemctl restart openvswitch

# 設定 flow rules (會自動 offload 到 hardware)
ovs-ofctl add-flow br0 "in_port=eth0_vf0,actions=output:eth0_vf1"
```

**OVS offload 的流程**:

```
ovs-ofctl add-flow ...
    ↓
OVS userspace daemon
    ↓
呼叫 TC API
    ↓
Kernel TC subsystem
    ↓
ice driver 的 tc_setup_clsflower callback
    ↓
ice_add_tc_flower_repr()
    ↓
下載 switch rule 到 hardware
    ↓
Hardware 直接轉發 (bypass kernel)
```

---

## 9. MDD (Malicious Driver Detection)

### 11.1 什麼是 MDD？

**Malicious Driver Detection** - 偵測 VF driver 的「惡意」或「錯誤」行為

**觸發 MDD 的常見原因**:

| 事件類型 | 說明 | 範例 |
|---------|------|------|
| TX MAC spoofing | VF 送出的封包 src MAC 不符 | VF MAC=02:00:00:00:00:01, 但送 src=AA:BB:CC:DD:EE:FF |
| TX VLAN violation | VF 嘗試送到不允許的 VLAN | Port VLAN=100, 但 VF 送 VLAN=200 的封包 |
| Invalid descriptor | VF 寫入非法的 descriptor 格式 | 錯誤的 descriptor type/command |
| Queue overflow | VF 寫入超過允許範圍的 queue | VF 有 4 個 queues，卻寫 queue #5 |
| RX buffer corruption | VF 破壞 RX descriptor ring | 寫入錯誤的 buffer address |

### 11.2 MDD Event 處理流程

**Code**: `ice_sriov.h:61-62`

```c
void ice_print_vf_rx_mdd_event(struct ice_vf *vf)
{
    struct ice_pf *pf = vf->pf;
    struct device *dev = ice_pf_to_dev(pf);

    // Rate limiting: 避免 log flood
    if (vf->mdd_rx_events.count > vf->mdd_rx_events.last_printed) {
        dev_info(dev, "VF %d has caused %d Rx Malicious Driver Detection events\n",
                 vf->vf_id,
                 vf->mdd_rx_events.count);
        vf->mdd_rx_events.last_printed = vf->mdd_rx_events.count;
    }
}

void ice_print_vf_tx_mdd_event(struct ice_vf *vf)
{
    // 類似 RX
}
```

**完整流程**:

```
┌─────────────────────────────────────────────────────────────────┐
│ VF 執行錯誤操作                                                  │
│   例如：送出 src_mac 不符的封包                                 │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Hardware 偵測到異常                                              │
│   • 設定 MDD status register                                    │
│   • 觸發 PF interrupt                                           │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ PF Driver Interrupt Handler                                     │
│                                                                 │
│   ice_misc_intr() - 讀取 interrupt cause                        │
│       │                                                         │
│       ├─> 如果是 MDD event                                      │
│       │                                                         │
│       └─> ice_handle_mdd_event()                                │
│             │                                                   │
│             ├─> 讀取 GL_MDET_TX_PQM register                    │
│             │     • 找出是哪個 VF 觸發                          │
│             │     • VF ID 從 register 的 bit field 取得         │
│             │                                                   │
│             ├─> vf->mdd_tx_events.count++                       │
│             │                                                   │
│             ├─> ice_print_vf_tx_mdd_event(vf)                   │
│             │     → dmesg: "VF 2 has caused 5 Tx MDD events"   │
│             │                                                   │
│             └─> 可選動作：                                       │
│                   if (pf->vf_reset_on_mdd)                      │
│                       ice_reset_vf(vf);  // 強制 reset VF      │
└─────────────────────────────────────────────────────────────────┘
```

### 11.3 PF 管理員的反應選項

**選項 1: 只記錄 (預設)**

```bash
# 查看 kernel log
dmesg | grep MDD
# ice 0000:18:00.0: VF 2 has caused 5 Tx Malicious Driver Detection events
```

**選項 2: 自動 reset VF**

```bash
# 透過 module parameter 設定
modprobe ice vf_reset_on_mdd=1

# 或在 /etc/modprobe.d/ice.conf:
# options ice vf_reset_on_mdd=1
```

**選項 3: 手動 reset VF**

```bash
# 找出 VF 的 PCIe address
lspci | grep "Virtual Function"
# 18:00.2 Ethernet controller: Intel E810 Virtual Function

# Reset VF
echo 1 > /sys/bus/pci/devices/0000:18:00.2/reset
```

**選項 4: 關閉 VF**

```bash
# 如果 VF 持續造成問題，可以移除
echo 0 > /sys/class/net/eth0/device/sriov_numvfs
# 重新啟用（跳過問題 VF）
echo 3 > /sys/class/net/eth0/device/sriov_numvfs
```

---

## 10. 完整範例：封包從 VM 到另一台機器

### 11.1 環境設定

```
┌──────────────────────────────────────────────────────────────┐
│ Host Server                                                  │
│   NIC: Intel E810 (eth0)                                     │
│   IP: 192.168.1.10/24                                        │
│   Gateway: 192.168.1.1                                       │
│                                                              │
│   VF #0:                                                     │
│     MAC: 02:00:00:00:00:01                                   │
│     Port VLAN: 100                                           │
│     Anti-spoofing: ON                                        │
│     Max TX rate: 1 Gbps                                      │
│                                                              │
│   ┌────────────────────────────────────────────────────┐    │
│   │ VM (Ubuntu 22.04)                                  │    │
│   │   NIC: eth0 (VF #0)                                │    │
│   │   IP: 192.168.1.100/24                             │    │
│   │   Gateway: 192.168.1.1                             │    │
│   │                                                    │    │
│   │   Application: curl http://8.8.8.8                 │    │
│   └────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### 11.2 完整封包流程

#### Step 1: Application 發起 HTTP Request (在 VM 裡)

```
VM$ curl http://8.8.8.8
    ↓
glibc getaddrinfo() → DNS lookup → 8.8.8.8
    ↓
socket(AF_INET, SOCK_STREAM, 0)
    ↓
connect(sockfd, {8.8.8.8:80}, ...)
    ↓
send(sockfd, "GET / HTTP/1.1\r\n...", len, 0)
```

#### Step 2: VM Network Stack 處理

```
Application layer
    ↓
TCP layer:
    • 建立 TCP segment
    • src_port = 54321 (ephemeral)
    • dst_port = 80
    • seq_num, ack_num, flags=SYN, ...
    • 計算 TCP checksum (或請求 HW offload)
    ↓
IP layer:
    • 建立 IP header
    • src_ip = 192.168.1.100 (VM's IP)
    • dst_ip = 8.8.8.8
    • protocol = 6 (TCP)
    • 計算 IP checksum (或請求 HW offload)
    ↓
Ethernet layer:
    • 查 ARP table: 8.8.8.8 → gateway 192.168.1.1
    • dst_mac = gateway's MAC (00:11:22:33:44:55)
    • src_mac = VF's MAC (02:00:00:00:00:01)
    • ethertype = 0x0800 (IPv4)
    ↓
建立 SKB (struct sk_buff)
    • skb->data = Ethernet frame
    • skb->len = 74 bytes (14 ETH + 20 IP + 20 TCP + 20 HTTP)
    • skb->ip_summed = CHECKSUM_PARTIAL (請求 HW offload)
```

#### Step 3: VF Driver (iavf) 發送

```
netdev_start_xmit(skb, netdev)
    ↓
iavf_xmit_frame(skb, netdev)
{
    // 選擇 TX queue (通常用 skb_get_queue_mapping)
    tx_ring = &adapter->tx_rings[0];  // 假設用 queue 0

    // DMA mapping
    dma_addr = dma_map_single(dev, skb->data, skb->len, DMA_TO_DEVICE);
    // dma_addr = 0x1234567890ABCDEF (VM 的 physical address)

    // 填寫 TX descriptor
    tx_desc = &tx_ring->desc[next_to_use];
    tx_desc->buffer_addr = cpu_to_le64(dma_addr);
    tx_desc->cmd_type_offset_bsz =
        BUILD_CTOB(I40E_TX_DESC_CMD_ICRC |    // Compute checksum
                   I40E_TX_DESC_CMD_EOP |     // End of packet
                   I40E_TX_DESC_CMD_RS,       // Report status
                   0, skb->len, 0);

    // 更新 tail register
    tx_ring->next_to_use++;
    writel(tx_ring->next_to_use, tx_ring->tail);
    // MMIO write → 離開 VM，進入 hardware
}
```

#### Step 4: Hardware 處理 (離開 VM)

```
┌═════════════════════════════════════════════════════════════════┐
║ Intel E810 Hardware                                             ║
╠═════════════════════════════════════════════════════════════════╣
│                                                                 │
│ (A) Detect tail register write                                 │
│     • VF queue 0's tail 從 5 → 6                                │
│     • 知道有新的 descriptor                                     │
│                                                                 │
│ (B) Fetch TX descriptor from VM memory                          │
│     • Read descriptor at ring_base + (6 * sizeof(desc))         │
│     • 取得: buffer_addr = 0x1234567890ABCDEF, len = 74          │
│                                                                 │
│ (C) Security Checks                                             │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ Check 1: MAC Anti-Spoofing                          │    │
│     │   vf = get_vf_by_queue(queue_0);  // VF #0          │    │
│     │   if (vf->spoofchk) {                               │    │
│     │       allowed_mac = 02:00:00:00:00:01;              │    │
│     │       packet_src_mac = read_from_packet(dma_addr);  │    │
│     │                      = 02:00:00:00:00:01;           │    │
│     │       if (packet_src_mac == allowed_mac)            │    │
│     │           ✓ PASS                                    │    │
│     │   }                                                 │    │
│     └─────────────────────────────────────────────────────┘    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ Check 2: Port VLAN Enforcement                      │    │
│     │   if (vf->port_vlan_info.vid == 100) {              │    │
│     │       // VF 送的封包不能有 VLAN tag                 │    │
│     │       if (packet_has_vlan_tag())                    │    │
│     │           ✗ MDD - Drop                              │    │
│     │                                                     │    │
│     │       // Hardware 加上 VLAN 100                     │    │
│     │       insert_vlan_tag(packet, vid=100, qos=0);      │    │
│     │       ✓ PASS                                        │    │
│     │   }                                                 │    │
│     └─────────────────────────────────────────────────────┘    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ Check 3: Bandwidth Limiting                         │    │
│     │   if (vf->max_tx_rate == 1000 Mbps) {               │    │
│     │       if (token_bucket_allow(vf, 74 bytes))         │    │
│     │           ✓ PASS (目前未超過限制)                   │    │
│     │       else                                          │    │
│     │           ✗ Rate Limited - Drop or Queue            │    │
│     │   }                                                 │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│ (D) DMA Read Packet Data                                        │
│     • Read 74 bytes from VM memory @ 0x1234567890ABCDEF         │
│     • Packet content:                                           │
│         [00:11:22:33:44:55] [02:00:00:00:00:01] [81:00]        │
│         [00:64] [08:00] [45:00...] [IP payload] [TCP...]       │
│          ↑dst_mac        ↑src_mac    ↑VLAN   ↑IPv4            │
│                                       (HW 加的)                 │
│                                                                 │
│ (E) TX Offload Processing                                       │
│     • Compute IP checksum                                       │
│     • Compute TCP checksum                                      │
│     • 最終封包 ready                                            │
│                                                                 │
│ (F) Send to MAC/PHY                                             │
│     • 經過 TX scheduler (QoS)                                   │
│     • Add Preamble + SFD                                        │
│     • Compute FCS                                               │
│     • 送出到實體層                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ Packet on Wire (VLAN tagged)                                    │
│   Destination: Gateway (00:11:22:33:44:55), VLAN 100           │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Switch/Router                                                    │
│   • Check VLAN 100 membership                                   │
│   • Route to next hop based on dst_ip = 8.8.8.8                │
│   • Remove VLAN tag (if going to non-VLAN port)                │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Internet → Google's server (8.8.8.8)                            │
└─────────────────────────────────────────────────────────────────┘
```

#### Step 5: 回程 (8.8.8.8 → VM)

```
┌─────────────────────────────────────────────────────────────────┐
│ 回覆封包從 8.8.8.8 回來                                          │
│   src_ip = 8.8.8.8, dst_ip = 192.168.1.100                     │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│ Router/Switch                                                    │
│   • Add VLAN 100 tag (因為 192.168.1.100 在 VLAN 100)           │
│   • dst_mac = 02:00:00:00:00:01 (VF's MAC)                      │
└────────────────┬────────────────────────────────────────────────┘
                 ↓
┌═════════════════════════════════════════════════════════════════┐
║ Intel E810 RX Path                                              ║
╠═════════════════════════════════════════════════════════════════╣
│                                                                 │
│ (1) PHY/MAC 接收封包                                            │
│     • FCS check ✓                                               │
│                                                                 │
│ (2) Parser                                                       │
│     • ptype = IPv4/TCP                                          │
│     • offsets = [(ETH,0), (VLAN,14), (IP,18), (TCP,38)]        │
│                                                                 │
│ (3) Field Vector Extraction                                     │
│     • Extract: dst_mac = 02:00:00:00:00:01                      │
│     • Extract: VLAN = 100                                       │
│     • Extract: 5-tuple for RSS                                  │
│                                                                 │
│ (4) Classifier                                                   │
│     MAC Filter Table:                                           │
│       02:00:00:00:00:01 → VSI 5 (VF #0)  ✓                     │
│                                                                 │
│     VLAN Check:                                                 │
│       packet.vlan = 100                                         │
│       vf->port_vlan = 100  ✓ Match                              │
│       → Strip VLAN tag (VF 看不到)                              │
│                                                                 │
│     RSS Hash:                                                   │
│       hash = toeplitz(...) = 0xABCDEF12                         │
│       lut_index = 0xABCDEF12 & 0x3F = 18                        │
│       queue = rss_lut[18] = 1  (VF queue 1)                    │
│                                                                 │
│ (5) Queue Mapping                                                │
│     hw_queue = vf_queue_base + 1 = 64 + 1 = 65                  │
│                                                                 │
│ (6) DMA to VF RX Ring                                            │
│     rx_ring = &all_rx_rings[65];                                │
│     descriptor = rx_ring->desc[next_to_use];                    │
│                                                                 │
│     // DMA 封包到 VM memory                                     │
│     dma_write(packet_data,                                      │
│               descriptor->read.pkt_addr,  // VM 的 buffer       │
│               packet_len);                                      │
│                                                                 │
│     // 寫入 metadata                                            │
│     descriptor->wb.pkt_len = packet_len;                        │
│     descriptor->wb.ptype = ptype;                               │
│     descriptor->wb.rss_hash = 0xABCDEF12;                       │
│     descriptor->wb.status_error0 |= DD_BIT;                     │
│                                                                 │
│ (7) Trigger Interrupt                                            │
│     vector = vf_vector_base + queue_1 = 100 + 1 = 101           │
│     send_msi_x_interrupt(vector=101, target=VM_vCPU);           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                             ↓
┌═════════════════════════════════════════════════════════════════┐
║ VM (Guest OS)                                                   ║
╠═════════════════════════════════════════════════════════════════╣
│                                                                 │
│ (8) Interrupt 到達 VM                                           │
│     IRQ #101 → iavf_msix_clean_rings()                          │
│               → napi_schedule()                                 │
│               → iavf_napi_poll()                                │
│                                                                 │
│ (9) iavf_clean_rx_irq()                                         │
│     rx_desc = &rx_ring->desc[next_to_clean];                    │
│                                                                 │
│     // Check DD bit                                             │
│     if (rx_desc->wb.status_error0 & DD_BIT) {                   │
│         dma_rmb();                                              │
│                                                                 │
│         // 建立 SKB                                              │
│         skb = build_skb(rx_desc->pkt_addr);                     │
│         skb->len = rx_desc->wb.pkt_len;                         │
│         skb->hash = rx_desc->wb.rss_hash;                       │
│         skb->ip_summed = CHECKSUM_UNNECESSARY;  // HW verified  │
│                                                                 │
│         // 送入 network stack                                   │
│         napi_gro_receive(napi, skb);                            │
│     }                                                           │
│                                                                 │
│ (10) Network Stack 處理                                         │
│      netif_receive_skb()                                        │
│        → ip_rcv()                                               │
│        → tcp_v4_rcv()                                           │
│        → tcp_data_queue()                                       │
│        → sock_queue_rcv_skb()                                   │
│        → wake_up_interruptible(socket->wait)                    │
│                                                                 │
│ (11) Application 收到資料                                       │
│      recv(sockfd, buffer, len, 0)  // 返回 HTTP response        │
│                                                                 │
│      curl 顯示: "<!doctype html>..."                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. 關鍵數據結構總結

### 11.1 PF 端 (ice driver)

**Code**: `ice_vf_lib.h:83-150`

```c
// VF 管理結構
struct ice_vfs {
    DECLARE_HASHTABLE(table, 8);    // VF hash table (256 buckets)
    struct mutex table_lock;         // 保護 hash table
    u16 num_supported;               // 最大支援 VF 數量 (256)
    u16 num_qps_per;                 // 每個 VF 的 queue pairs
    u16 num_msix_per;                // 每個 VF 的 MSI-X vectors
};

// 每個 VF 的資訊
struct ice_vf {
    struct hlist_node entry;         // Hash table entry
    struct ice_pf *pf;               // 指向 PF
    struct pci_dev *vfdev;           // VF 的 PCIe device

    u16 vf_id;                       // VF ID (0-255)
    u16 lan_vsi_idx;                 // VSI index in PF's VSI table
    u16 ctrl_vsi_idx;                // Control VSI (for mailbox)

    // Resources
    u16 num_vf_qs;                   // Queue 數量
    u16 num_msix;                    // MSI-X vector 數量
    int first_vector_idx;            // Vector base index

    // MAC & VLAN
    u8 dev_lan_addr[ETH_ALEN];       // VF 的 MAC address
    struct ice_vlan port_vlan_info;  // Port VLAN 設定

    // Security & QoS
    bool spoofchk;                   // Anti-spoofing enable
    bool trusted;                    // Trusted VF flag
    unsigned int max_tx_rate;        // TX bandwidth limit (Mbps)

    // State
    DECLARE_BITMAP(vf_states, ICE_VF_STATES_NBITS);
    // States: INIT, ACTIVE, QS_ENA, DIS, MC_PROMISC, UC_PROMISC

    // MDD tracking
    struct ice_mdd_vf_events mdd_tx_events;
    struct ice_mdd_vf_events mdd_rx_events;

    // Virtchnl
    struct virtchnl_version_info vf_ver;  // VF driver version
    u32 driver_caps;                      // VF capabilities
    DECLARE_BITMAP(opcodes_allowlist, VIRTCHNL_OP_MAX);
};
```

### 11.2 VF 端 (iavf driver)

```c
// VF adapter 結構 (簡化)
struct iavf_adapter {
    struct net_device *netdev;
    struct pci_dev *pdev;

    // Hardware info
    struct iavf_hw hw;
    u16 vsi_id;                      // VSI ID (from PF)

    // Resources (from PF)
    u16 num_active_queues;           // Queue 數量
    u16 num_msix_vectors;            // MSI-X vector 數量

    // Rings
    struct iavf_ring *tx_rings[MAX_QUEUES];
    struct iavf_ring *rx_rings[MAX_QUEUES];

    // RSS
    u8 rss_key[IAVF_RSS_KEY_SIZE];
    u8 rss_lut[IAVF_RSS_LUT_SIZE];

    // Link state
    bool link_up;
    u32 link_speed;

    // Virtchnl state
    enum iavf_state_t state;
    // STARTUP, RUNNING, RESETTING, etc.

    // Pending virtchnl requests
    struct list_head pending_cmds;
};
```

### 11.3 Hardware Registers

**重要的 VF-related registers**:

| Register | 用途 | 範例值 (VF #0) |
|----------|------|---------------|
| `VPINT_ALLOC(vf_id)` | VF 的 interrupt vector 數量 | 5 |
| `VPINT_ALLOC_PCI(vf_id)` | PCIe interrupt 配置 | 5 |
| `VPLAN_TX_QBASE(vf_id)` | TX queue base + 數量 | base=64, num=4 |
| `VPLAN_RX_QBASE(vf_id)` | RX queue base + 數量 | base=64, num=4 |
| `GLINT_VECT2FUNC(vec)` | Vector 屬於哪個 VF/PF | vf_num=0, is_pf=0 |
| `GL_MDET_TX_PQM` | TX MDD event status | bit[2]=1 (VF #2 MDD) |
| `GL_MDET_RX` | RX MDD event status | - |
| `VFGEN_RSTAT` | VF reset status | ACTIVE/IN_RESET |

---


---

## 11. Deep Dive: Scalability & Resource Management

### 11.1 VF "Birth" Process: Kernel vs. Firmware

VF 的誕生是一個 Kernel 和 Firmware 協作的過程。

**1. Kernel Layer (PCIe Configuration)**
當你執行 `echo 4 > sriov_numvfs` 時，Kernel 的 `pci_enable_sriov()` 會：
- 寫入 PCIe SR-IOV Capability 結構中的 `NumVFs` register。
- 透過 PCIe 總線枚舉出新的 VF device (e.g., `0000:05:00.1`)。
- 此時 VF 雖然在 lspci 看得到，但內部資源 (VSI, Queues) 還沒準備好。

**2. Driver Layer (Resource Allocation)**
Driver 的 `ice_ena_vfs()` 負責實際的資源分配：
- **Software**: `ice_create_vf_entries()` 分配 `struct ice_vf` 結構。
- **Hardware**: `ice_start_vfs()` 呼叫 `ice_init_vf_vsi_res()` 建立 VSI。

**3. Firmware Layer (Switch Configuration)**
最後透過 AdminQ 告訴 Firmware：
- "建立一個 VSI (Type = VF)"
- "將這個 VSI 關聯到 VF ID x"
- "設定 Default MAC filter"

### 11.2 Resource Starvation & Pooling (資源爭奪戰)

當 VF 數量很大時 (e.g., 256 VFs)，硬體資源 (Interrupt Vectors, Queues) 可能不夠分。Driver 有一套 **"Dynamic Downgrade"** 機制。

**Code**: `ice_sriov.c: ice_set_per_vf_res()`

**邏輯**:
1. 計算平均可用 MSI-X: `msix_avail_per_vf = global_avail / num_vfs`
2. 根據可用數量決定 VF 等級：

| Level | MSI-X Count | Queue Pairs | Condition |
|-------|-------------|-------------|-----------|
| **Medium** | 17 | 16 | `msix >= 17` |
| **Small** | 5 | 4 | `msix >= 5` |
| **MultiQ Min** | 3 | 2 | `msix >= 3` |
| **Min** | 2 | 1 | `msix >= 2` |

**Fail Condition**:
如果 `msix_avail_per_vf < 2`，則回傳 `-ENOSPC`，拒絕啟用 SR-IOV。這保護了系統不會因為資源過度分割而崩潰。

### 11.3 Bandwidth Management (QoS)

如何防止一個 VF 吃掉所有頻寬？

**1. Oversubscription Check**
在設定 `min_tx_rate` 時，Driver 會檢查總和是否超過 Link Speed。

**Code**: `ice_sriov.c: ice_min_tx_rate_oversubscribed()`
```c
if (all_vfs_min_tx_rate + new_rate > link_speed) {
    return -EINVAL; // Reject configuration
}
```

**2. Hardware Enforcement (Scheduler)**
Driver 使用 `ice_set_max_bw_limit()` 設定硬體 Scheduler。
- 這會對應到 AdminQ command `Add/Cfg Scheduler Node`。
- Firmware 會在 Transmit Scheduler Tree 中設定 Rate Limit (RL) Profile。
- 超過速率的封包會被硬體 buffer 或 drop，完全不消耗 PCIe 頻寬。

### 11.4 PCIe Limits (Scalability)

**1. BDF (Bus:Device.Function) Limit**
- 標準 PCIe 只支援 8 個 Function (3 bits)。
- SR-IOV 使用 **ARI (Alternative Routing-ID Interpretation)** 來支援更多 VF。
- ARI 將 Device (5 bits) 和 Function (3 bits) 合併為 8 bits Function ID，允許單一 Bus 下有 256 個 Function。
- ICE 硬體支援 ARI，因此可以單 Port 支援 256 VFs。

**2. BAR (Base Address Register) Space**
- 每個 VF 需要獨立的 MMIO 空間 (BAR0 for Mailbox/Registers, BAR3 for MSI-X)。
- 如果 System BIOS 分配的 64-bit MMIO 空間不足，啟用大量 VF 會失敗 (Error: `Not enough MMIO resources`)。
- **解法**: 在 BIOS 開啟 "Above 4G Decoding" 和 "SR-IOV Support"。

---

## 總結

ICE driver 的 SR-IOV 實作展現了完整的硬體虛擬化架構：

**關鍵設計原則**:

1. **資源隔離**: 每個 VF 有獨立的 VSI、queues、interrupts
2. **安全隔離**: Anti-spoofing、VLAN enforcement、bandwidth limiting
3. **彈性管理**: Mailbox 讓 VF 請求特權操作，PF 有完整控制權
4. **效能**: 封包直接 DMA 到 VM memory，接近原生效能
5. **可程式化**: Switchdev mode 整合 TC/OVS，支援 offload

**End-to-End 的封包路徑**清楚展示了 SR-IOV 如何在保持隔離的同時，提供高效能的網路 I/O。

---

**驗證日期**: 2025-11-14
**基於**: Linux Kernel ICE Driver (`drivers/net/ethernet/intel/ice/`)
