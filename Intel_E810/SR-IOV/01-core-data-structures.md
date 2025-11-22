# Intel E810 (ice) Driver SR-IOV 核心數據結構分析

## 概述

本文檔分析 Linux kernel 中 Intel E810 網卡驅動 (ice driver) 的 SR-IOV 實現機制，專注於核心數據結構設計。

**代碼位置**: `drivers/net/ethernet/intel/ice/`

---

## 1. 核心數據結構層次

### 1.1 PF (Physical Function) 主結構

**文件**: `ice.h:554`

```c
struct ice_pf {
    struct pci_dev *pdev;           /* PCIe 設備指針 */
    struct ice_hw hw;               /* 硬件抽象層 */

    /* VSI (Virtual Station Interface) 管理 */
    struct ice_vsi **vsi;           /* VSI 陣列 */
    u16 num_alloc_vsi;              /* 已分配的 VSI 數量 */
    u16 next_vsi;                   /* 下一個可用 VSI 索引 */
    u16 ctrl_vsi_idx;               /* 控制 VSI 索引 */

    /* VF 管理結構 */
    struct ice_vfs vfs;             /* SR-IOV VF 管理結構 */

    /* Queue 資源管理 */
    unsigned long *avail_txqs;      /* 可用 Tx queue bitmap */
    unsigned long *avail_rxqs;      /* 可用 Rx queue bitmap */
    u16 max_pf_txqs;                /* PF 最大 Tx queues */
    u16 max_pf_rxqs;                /* PF 最大 Rx queues */
    u16 num_lan_tx;                 /* LAN Tx queues 數量 */
    u16 num_lan_rx;                 /* LAN Rx queues 數量 */

    /* 中斷資源 */
    struct ice_irq_tracker irq_tracker;           /* 物理 IRQ 追蹤器 */
    struct ice_virt_irq_tracker virt_irq_tracker; /* 虛擬 IRQ 追蹤器（VF用）*/
    struct ice_pf_msix msix;                      /* MSI-X 向量管理 */

    /* Switch 結構 */
    struct ice_sw *first_sw;        /* 第一個 switch（firmware 創建）*/

    /* 鎖機制 */
    struct mutex avail_q_mutex;     /* 保護 queue 分配 */
    struct mutex sw_mutex;          /* 保護 VSI 分配流程 */

    /* 聚合節點（QoS/頻寬管理）*/
    struct ice_agg_node pf_agg_node[ICE_MAX_PF_AGG_NODES]; /* PF 聚合節點 */
    struct ice_agg_node vf_agg_node[ICE_MAX_VF_AGG_NODES]; /* VF 聚合節點 */
};
```

**關鍵洞察**:
- PF 使用 `ice_vfs` 結構集中管理所有 VF
- Queue 資源通過 bitmap 追蹤，使用 mutex 保護並發訪問
- 中斷資源分為物理和虛擬兩套追蹤系統（VF 使用虛擬 IRQ）

---

### 1.2 VF 集合管理結構

**文件**: `ice_vf_lib.h:83`

```c
struct ice_vfs {
    DECLARE_HASHTABLE(table, 8);    /* VF hash table（256 entries）*/
    struct mutex table_lock;         /* 保護 hash table 的鎖 */
    u16 num_supported;               /* 此 PF 支持的最大 VF 數 */
    u16 num_qps_per;                 /* 每個 VF 的 queue pairs 數量 */
    u16 num_msix_per;                /* 每個 VF 的默認 MSI-X 向量數 */
    unsigned long last_printed_mdd_jiffies; /* MDD 訊息速率限制 */
};
```

**設計特點**:
- 使用 hash table 而非陣列存儲 VF，支持高效查找
- 通過 RCU 和 mutex 雙重保護機制支持並發訪問
- MDD (Malicious Driver Detection) 集成在 VF 管理中

**常數定義**:
```c
#define ICE_MAX_SRIOV_VFS  256    /* 最大 VF 數量 */
```

---

### 1.3 單個 VF 描述符

**文件**: `ice_vf_lib.h:93`

```c
struct ice_vf {
    /* 基本屬性 */
    struct hlist_node entry;        /* hash table 鏈表節點 */
    struct rcu_head rcu;            /* RCU 保護 */
    struct kref refcnt;             /* 引用計數 */
    struct ice_pf *pf;              /* 反向指針到 PF */
    struct pci_dev *vfdev;          /* VF 的 PCIe 設備 */

    u16 vf_id;                      /* VF ID（在 PF 空間內）*/
    u16 lan_vsi_idx;                /* LAN VSI 索引 */
    u16 ctrl_vsi_idx;               /* 控制 VSI 索引 */

    /* 資源分配 */
    u16 num_vf_qs;                  /* 配置的 queue pairs 數量 */
    u16 num_msix;                   /* 配置的 MSI-X 數量 */
    int first_vector_idx;           /* 第一個向量索引（在 PF 空間內）*/

    DECLARE_BITMAP(txq_ena, ICE_MAX_RSS_QS_PER_VF); /* 啟用的 Tx queues */
    DECLARE_BITMAP(rxq_ena, ICE_MAX_RSS_QS_PER_VF); /* 啟用的 Rx queues */

    /* Mailbox 通信 */
    struct ice_mbx_vf_info mbx_info;         /* Mailbox 信息 */
    struct virtchnl_version_info vf_ver;     /* VF 驅動版本 */
    u32 driver_caps;                         /* VF 驅動能力 */

    /* MAC 地址 */
    u8 dev_lan_addr[ETH_ALEN];      /* 設備 LAN MAC */
    u8 hw_lan_addr[ETH_ALEN];       /* 硬件 LAN MAC */

    /* VLAN 配置 */
    struct ice_vlan port_vlan_info; /* Port VLAN ID, QoS, TPID */
    struct virtchnl_vlan_caps vlan_v2_caps; /* VLAN v2 能力 */

    /* 狀態和配置 */
    DECLARE_BITMAP(vf_states, ICE_VF_STATES_NBITS);
    unsigned long vf_caps;          /* VF 高級能力 */
    u8 num_req_qs;                  /* VF 請求的 queue pairs */

    /* QoS/頻寬管理 */
    unsigned int min_tx_rate;       /* 最小 Tx 頻寬（Mbps）*/
    unsigned int max_tx_rate;       /* 最大 Tx 頻寬（Mbps）*/
    struct ice_vf_qs_bw qs_bw[ICE_MAX_RSS_QS_PER_VF]; /* 每 queue 頻寬 */

    /* 安全和信任 */
    u8 pf_set_mac:1;                /* VMM 管理員設定的 MAC */
    u8 trusted:1;                   /* 信任狀態 */
    u8 spoofchk:1;                  /* 防欺騙檢查 */
    u8 link_forced:1;               /* 強制鏈路狀態 */
    u8 link_up:1;                   /* 鏈路狀態（當強制時有效）*/

    /* MDD 事件追蹤 */
    struct ice_mdd_vf_events mdd_rx_events;
    struct ice_mdd_vf_events mdd_tx_events;

    /* 操作函數表 */
    const struct ice_virtchnl_ops *virtchnl_ops; /* Virtual channel 操作 */
    const struct ice_vf_ops *vf_ops;              /* VF 特定操作 */

    /* Flow Director */
    struct ice_vf_fdir fdir;
    struct ice_fdir_prof_info fdir_prof_info[ICE_MAX_PTGS];

    /* RSS 配置 */
    u64 rss_hashcfg;

    /* Devlink */
    struct devlink_port devlink_port;

    /* 同步鎖 */
    struct mutex cfg_lock;          /* VirtChnl 訊息和 NDO ops 鎖 */
};
```

**設計亮點**:

1. **生命週期管理**:
   - `kref refcnt` + RCU 確保在並發環境下安全釋放
   - Hash table 節點設計支持快速查找

2. **資源隔離**:
   - 每個 VF 有獨立的 VSI（lan_vsi_idx, ctrl_vsi_idx）
   - Queue 和中斷資源通過 bitmap 追蹤
   - 頻寬限制通過 QoS 結構實現

3. **安全機制**:
   - MAC 防欺騙檢查（spoofchk）
   - 信任模型（trusted）
   - MDD (Malicious Driver Detection) 事件追蹤

---

### 1.4 VF 狀態機

**文件**: `ice_vf_lib.h:34`

```c
enum ice_vf_states {
    ICE_VF_STATE_INIT = 0,    /* PF 正在初始化 VF */
    ICE_VF_STATE_ACTIVE,      /* VF 資源已分配可用 */
    ICE_VF_STATE_QS_ENA,      /* VF queue(s) 已啟用 */
    ICE_VF_STATE_DIS,         /* VF 已禁用 */
    ICE_VF_STATE_MC_PROMISC,  /* VF 處於多播混雜模式 */
    ICE_VF_STATE_UC_PROMISC,  /* VF 處於單播混雜模式 */
    ICE_VF_STATES_NBITS
};
```

---

## 2. 資源管理約束

### 2.1 VF 資源限制

**文件**: `ice_sriov.h:18` 和 `ice_vf_lib.h:19`

```c
/* VF resource constraints */
#define ICE_MIN_QS_PER_VF          1     /* 最小 queue pairs */
#define ICE_MAX_RSS_QS_PER_VF      16    /* 最大 RSS queues */

/* MSI-X 配置 */
#define ICE_NONQ_VECS_VF           1     /* 非 queue 向量數 */
#define ICE_NUM_VF_MSIX_MED        17    /* 中型配置 MSI-X */
#define ICE_NUM_VF_MSIX_SMALL      5     /* 小型配置 MSI-X */
#define ICE_NUM_VF_MSIX_MULTIQ_MIN 3     /* 多 queue 最小 MSI-X */
#define ICE_MIN_INTR_PER_VF        (ICE_MIN_QS_PER_VF + 1)

/* Reset 相關 */
#define ICE_MAX_VF_RESET_TRIES     40    /* 最大重試次數 */
#define ICE_MAX_VF_RESET_SLEEP_MS  20    /* 重試間隔（毫秒）*/
```

### 2.2 PF 聚合節點配置

**文件**: `ice.h:658`

```c
#define ICE_INVALID_AGG_NODE_ID      0
#define ICE_PF_AGG_NODE_ID_START     1
#define ICE_MAX_PF_AGG_NODES         32
#define ICE_VF_AGG_NODE_ID_START     65
#define ICE_MAX_VF_AGG_NODES         32
```

**用途**: 用於 QoS 和頻寬管理的樹狀調度結構

---

## 3. VF 操作抽象層

### 3.1 VF 操作函數表

**文件**: `ice_vf_lib.h:70`

```c
struct ice_vf_ops {
    enum ice_disq_rst_src reset_type;
    void (*free)(struct ice_vf *vf);
    void (*clear_reset_state)(struct ice_vf *vf);
    void (*clear_mbx_register)(struct ice_vf *vf);
    void (*trigger_reset_register)(struct ice_vf *vf, bool is_vflr);
    bool (*poll_reset_status)(struct ice_vf *vf);
    void (*clear_reset_trigger)(struct ice_vf *vf);
    void (*irq_close)(struct ice_vf *vf);
    void (*post_vsi_rebuild)(struct ice_vf *vf);
};
```

**設計模式**: 使用 ops 結構實現多態，支持不同類型 VF（SR-IOV, Scalable IOV 等）

---

## 4. 關鍵 API 接口

### 4.1 SR-IOV 配置

**文件**: `ice_sriov.h:30`

```c
int ice_sriov_configure(struct pci_dev *pdev, int num_vfs);
void ice_free_vfs(struct ice_pf *pf);
void ice_process_vflr_event(struct ice_pf *pf);  /* VF Level Reset */
```

### 4.2 VF 查找和引用管理

**文件**: `ice_vf_lib.h:241`

```c
struct ice_vf *ice_get_vf_by_id(struct ice_pf *pf, u16 vf_id);
void ice_put_vf(struct ice_vf *vf);
bool ice_has_vfs(struct ice_pf *pf);
u16 ice_get_num_vfs(struct ice_pf *pf);
```

### 4.3 VF 資源控制（NDO 接口）

**文件**: `ice_sriov.h:31`

```c
int ice_set_vf_mac(struct net_device *netdev, int vf_id, u8 *mac);
int ice_set_vf_trust(struct net_device *netdev, int vf_id, bool trusted);
int ice_set_vf_port_vlan(struct net_device *netdev, int vf_id,
                         u16 vlan_id, u8 qos, __be16 vlan_proto);
int ice_set_vf_bw(struct net_device *netdev, int vf_id,
                  int min_tx_rate, int max_tx_rate);
int ice_set_vf_link_state(struct net_device *netdev, int vf_id, int link_state);
int ice_set_vf_spoofchk(struct net_device *netdev, int vf_id, bool ena);
int ice_get_vf_cfg(struct net_device *netdev, int vf_id,
                   struct ifla_vf_info *ivi);
int ice_get_vf_stats(struct net_device *netdev, int vf_id,
                     struct ifla_vf_stats *vf_stats);
```

---

## 5. 設計重點

### 5.1 數據結構選擇

*   **Hash Table + RCU**:
    *   **設計**: 使用 `DECLARE_HASHTABLE(table, 8)` 管理 VF。
    *   **重點**: 256 個 VF 使用 Hash Table (256 buckets) 相比固定陣列更靈活，且配合 RCU 機制 (`hash_for_each_rcu`) 實現無鎖快速查找，適合頻繁的 Mailbox 通信場景。

*   **引用計數 (Reference Counting)**:
    *   **設計**: `struct kref refcnt` 配合 `struct rcu_head`。
    *   **重點**: 確保在並發環境下（如 VF 銷毀時仍有 Mailbox 消息處理）物件生命週期的安全性，避免 Use-After-Free。

*   **Bitmap 資源追蹤**:
    *   **設計**: `DECLARE_BITMAP(txq_ena, ...)`。
    *   **重點**: 使用位圖高效管理 Queue 的啟用狀態，操作原子性高且節省空間。

### 5.2 可優化空間

*   **Boolean Flags**:
    *   **現狀**: `struct ice_vf` 中包含多個 `u8 flag:1` 欄位。
    *   **建議**: 可合併為 `unsigned long flags` 並使用原子位操作 (atomic bitops) 統一管理，提升代碼整潔度與並發安全性。

*   **結構體內聚性**:
    *   **現狀**: `struct ice_vf` 包含 30+ 個欄位，混合了配置、狀態和運行時數據。
    *   **建議**: 將運行時易變狀態 (Runtime State) 與靜態配置 (Config) 分離，例如拆分出 `ice_vf_runtime` 結構，提高 Cache Locality。

---

## 下一步分析

1. ✅ 核心數據結構（本文檔）
2. ⏭️ SR-IOV 初始化流程
3. ⏭️ BAR mapping 和 memory management
4. ⏭️ Mailbox (virtchnl) 通信機制
5. ⏭️ Queue 和封包派發
6. ⏭️ PCIe 配置空間操作
7. ⏭️ IOMMU/DMA 整合

---

**文檔版本**: v1.0
**更新時間**: 2025-11-19
**分析工具**: Linux kernel mainline (latest)
