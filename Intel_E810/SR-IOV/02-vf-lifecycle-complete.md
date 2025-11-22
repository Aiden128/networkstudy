# Intel E810 VF 完整生命週期 End-to-End 代碼追蹤

## 概述

本文檔從**實際代碼**角度，完整追蹤一個 VF 從創建到銷毀的全部生命週期。每個階段都包含關鍵函數的實際代碼片段和寄存器操作。

**核心文件**: `ice_sriov.c`, `ice_vf_lib.c`, `virt/virtchnl.c`, `virt/queues.c`, `ice_txrx.c`

---

## VF 生命週期總覽

```
┌─────────────────────────────────────────────────────────────────┐
│  階段 1: VF 創建 (Hardware Initialization)                      │
│  用戶: echo 4 > sriov_numvfs                                    │
│  → ice_sriov_configure()                                        │
│  → pci_enable_sriov()  【PCIe 層創建 VF 設備】                  │
│  → ice_ena_vf_mappings() 【硬件寄存器配置】                     │
│  → VFGEN_RSTAT = ACTIVE 【VF 可見】                            │
│  狀態: VF 硬件就緒，等待 driver                                 │
└─────────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│  階段 2: VF Driver Probe & Virtchnl 協商                        │
│  VF driver (iavf) 偵測到設備                                    │
│  → VF: VIRTCHNL_OP_VERSION                                      │
│  → PF: ice_vc_get_ver_msg() 【版本協商】                        │
│  → VF: VIRTCHNL_OP_GET_VF_RESOURCES                             │
│  → PF: ice_vc_get_vf_res_msg() 【告知 VF 資源】                 │
│  狀態: VF STATE_ACTIVE, 知道自己有多少 queue/MSI-X              │
└─────────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│  階段 3: Queue 配置與啟用                                        │
│  → VF: VIRTCHNL_OP_CONFIG_QUEUES                                │
│  → PF: ice_vc_cfg_qs_msg() 【配置 descriptor rings】            │
│  → VF: VIRTCHNL_OP_ENABLE_QUEUES                                │
│  → PF: ice_vc_ena_qs_msg() 【啟用 Rx queues】                   │
│  狀態: Queues ready, 可以收發封包                               │
└─────────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│  階段 4: Runtime 封包處理                                        │
│  Rx: 網路封包 → DMA → descriptor → MSI-X → ice_clean_rx_irq()  │
│  Tx: ice_start_xmit() → DMA map → descriptor → doorbell → 網路 │
│  持續的 virtchnl 通信: 統計、配置變更                           │
└─────────────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│  階段 5: VF 銷毀                                                 │
│  用戶: echo 0 > sriov_numvfs                                    │
│  → ice_free_vfs()                                               │
│  → ice_vf_vsi_release() 【清理 VSI】                            │
│  → pci_disable_sriov() 【移除 PCIe VF 設備】                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 階段 1: VF 創建 (Hardware Initialization)

### 1.1 初始化流程總覽

```
用戶空間
    |
    | echo N > /sys/class/net/ethX/device/sriov_numvfs
    |
    v
ice_sriov_configure()         [ice_sriov.c:1039]
    |
    +-- ice_check_sriov_allowed()
    |
    +-- ice_pci_sriov_ena()       [ice_sriov.c:810]
        |
        +-- ice_ena_vfs()          [ice_sriov.c:742]
            |
            +-- pci_enable_sriov()          【PCIe 層：啟用 SR-IOV】
            |
            +-- ice_set_per_vf_res()        [ice_sriov.c:371]
            |   |
            |   +-- 計算每個 VF 的資源分配
            |       - MSI-X vectors
            |       - Queue pairs (Tx/Rx)
            |
            +-- ice_create_vf_entries()     [ice_sriov.c:683]
            |   |
            |   +-- 讀取 PCIe SR-IOV capability
            |   +-- 為每個 VF 分配 struct ice_vf
            |   +-- 將 VF 加入 hash table
            |
            +-- ice_start_vfs()             [ice_sriov.c:467]
                |
                +-- FOR EACH VF:
                    |
                    +-- ice_init_vf_vsi_res()   [ice_sriov.c:438]
                    |   |
                    |   +-- ice_virt_get_irqs()      【分配虛擬 IRQ】
                    |   +-- ice_vf_vsi_setup()       【創建 VSI】
                    |   +-- ice_vf_init_host_cfg()   【初始化 host 配置】
                    |
                    +-- ice_ena_vf_mappings()       [ice_sriov.c:323]
                    |   |
                    |   +-- ice_ena_vf_msix_mappings()  【MSI-X 映射】
                    |   +-- ice_ena_vf_q_mappings()     【Queue 映射】
                    |
                    +-- VFGEN_RSTAT = VFACTIVE      【標記 VF 為 active】
```

---

## 2. 關鍵函數詳解

### 2.1 ice_sriov_configure() - 主入口

**文件**: `ice_sriov.c:1039`

```c
int ice_sriov_configure(struct pci_dev *pdev, int num_vfs)
{
    struct ice_pf *pf = pci_get_drvdata(pdev);

    // 1. 檢查是否允許 SR-IOV
    err = ice_check_sriov_allowed(pf);

    // 2. 如果 num_vfs == 0，釋放所有 VF
    if (!num_vfs) {
        if (!pci_vfs_assigned(pdev)) {  // 確保沒有 VF 被分配給 VM
            ice_free_vfs(pf);
            return 0;
        }
        return -EBUSY;  // VF 正在使用中
    }

    // 3. 啟用 SR-IOV
    err = ice_pci_sriov_ena(pf, num_vfs);
    return num_vfs;
}

**設計重點**:
*   **資源分配策略**: 採用 "Best Effort" 策略。如果無法滿足請求的資源量，會嘗試降級配置 (Medium -> Small)，而不是直接失敗，這提高了系統的可用性。
- ✅ `pci_vfs_assigned()` 檢查消除了 "VF 正在使用時被刪除" 的邊界情況

---

### 2.2 ice_set_per_vf_res() - 資源計算核心

**文件**: `ice_sriov.c:371`

這個函數是整個 SR-IOV 資源管理的靈魂，決定了每個 VF 能獲得多少資源。

```c
static int ice_set_per_vf_res(struct ice_pf *pf, u16 num_vfs)
{
    u16 num_msix_per_vf, num_txq, num_rxq, avail_qs;
    int msix_avail_per_vf, msix_avail_for_sriov;

    // === 第一步：MSI-X 向量分配 ===
    msix_avail_for_sriov = pf->virt_irq_tracker.num_entries;
    msix_avail_per_vf = msix_avail_for_sriov / num_vfs;

    // 分級策略：根據可用資源決定配置規模
    if (msix_avail_per_vf >= ICE_NUM_VF_MSIX_MED) {        // >= 17
        num_msix_per_vf = ICE_NUM_VF_MSIX_MED;
    } else if (msix_avail_per_vf >= ICE_NUM_VF_MSIX_SMALL) {  // >= 5
        num_msix_per_vf = ICE_NUM_VF_MSIX_SMALL;
    } else if (msix_avail_per_vf >= ICE_NUM_VF_MSIX_MULTIQ_MIN) { // >= 3
        num_msix_per_vf = ICE_NUM_VF_MSIX_MULTIQ_MIN;
    } else if (msix_avail_per_vf >= ICE_MIN_INTR_PER_VF) {  // >= 2
        num_msix_per_vf = ICE_MIN_INTR_PER_VF;
    } else {
        return -ENOSPC;  // 資源不足
    }

    // === 第二步：Tx Queue 分配 ===
    // MSI-X - 1 (mailbox) = 可用於 queue 的向量數
    num_txq = min_t(u16, num_msix_per_vf - ICE_NONQ_VECS_VF,
                    ICE_MAX_RSS_QS_PER_VF);  // 最多 16 queues

    // 檢查 PF 是否有足夠的 Tx queue
    avail_qs = ice_get_avail_txq_count(pf) / num_vfs;
    if (!avail_qs)
        num_txq = 0;
    else if (num_txq > avail_qs)
        num_txq = rounddown_pow_of_two(avail_qs);  // 保證是2的冪次

    // === 第三步：Rx Queue 分配（邏輯同 Tx）===
    num_rxq = min_t(u16, num_msix_per_vf - ICE_NONQ_VECS_VF,
                    ICE_MAX_RSS_QS_PER_VF);
    avail_qs = ice_get_avail_rxq_count(pf) / num_vfs;
    // ... 同 Tx 邏輯

    // === 第四步：最終驗證 ===
    if (num_txq < ICE_MIN_QS_PER_VF || num_rxq < ICE_MIN_QS_PER_VF) {
        return -ENOSPC;  // 必須至少有 1 個 queue pair
    }

    // === 第五步：強制 Tx/Rx 對稱 ===
    pf->vfs.num_qps_per = min_t(int, num_txq, num_rxq);
    pf->vfs.num_msix_per = num_msix_per_vf;

    return 0;
}
```

**資源分配策略**:

| MSI-X 可用數 | 配置策略      | Queues   | 用途                      |
| ------------ | ------------- | -------- | ------------------------- |
| ≥ 17         | MEDIUM        | 最多 16  | 高性能多 queue RSS        |
| 5-16         | SMALL         | 4        | 標準性能                  |
| 3-4          | MULTIQ_MIN    | 2        | 基本多 queue              |
| 2            | MIN_INTR      | 1        | 最小可用配置              |
| < 2          | 失敗          | -        | 資源不足                  |

**潛在問題**:
*   **Magic Number**: 代碼中直接使用了 `17` (ICE_NUM_VF_MSIX_MED) 等硬編碼數值。
*   **建議**: 應將這些數值定義為具名的常數或通過 device tree/firmware 配置，以提高可讀性和維護性。
   - 可以改用 table-driven approach

---

### 2.3 ice_create_vf_entries() - VF 條目創建

**文件**: `ice_sriov.c:683`

```c
static int ice_create_vf_entries(struct ice_pf *pf, u16 num_vfs)
{
    struct pci_dev *pdev = pf->pdev;
    struct ice_vfs *vfs = &pf->vfs;

    // === 步驟 1: 讀取 PCIe SR-IOV Capability ===
    pos = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_SRIOV);
    pci_read_config_word(pdev, pos + PCI_SRIOV_VF_DID, &vf_pdev_id);

    // === 步驟 2: 為每個 VF 創建管理結構 ===
    for (u16 vf_id = 0; vf_id < num_vfs; vf_id++) {
        // 分配 VF 結構
        vf = kzalloc(sizeof(*vf), GFP_KERNEL);

        // 初始化引用計數（用於生命週期管理）
        kref_init(&vf->refcnt);

        // 設置基本屬性
        vf->pf = pf;
        vf->vf_id = vf_id;
        vf->vf_ops = &ice_sriov_vf_ops;  // 操作函數表

        // 初始化 VF entry（mailbox, 狀態等）
        ice_initialize_vf_entry(vf);

        // === 步驟 3: 關聯物理 PCIe VF 設備 ===
        // 通過 vendor ID 和 device ID 找到對應的 VF PCI 設備
        do {
            vfdev = pci_get_device(pdev->vendor, vf_pdev_id, vfdev);
        } while (vfdev && vfdev->physfn != pdev);  // 確保是此 PF 的 VF

        vf->vfdev = vfdev;
        vf->vf_sw_id = pf->first_sw;  // 關聯 switch

        pci_dev_get(vfdev);  // 增加引用計數

        // === 步驟 4: 加入 Hash Table ===
        hash_add_rcu(vfs->table, &vf->entry, vf_id);
    }

    pci_dev_put(vfdev);  // 平衡最後一次引用
    return 0;
}
```

**關鍵點**:
1.  **PCIe 綁定**: 通過 `pci_get_device()` 將軟件 VF 結構與硬件 PCIe VF 設備關聯
2.  **RCU 保護**: 使用 `hash_add_rcu()` 確保並發訪問安全
3.  **引用計數**: `kref` + `pci_dev_get/put` 雙重引用計數管理

---

### 2.4 ice_init_vf_vsi_res() - VSI 資源初始化

**文件**: `ice_sriov.c:438`

```c
static int ice_init_vf_vsi_res(struct ice_vf *vf)
{
    struct ice_pf *pf = vf->pf;
    struct ice_vsi *vsi;

    // === 步驟 1: 分配虛擬 IRQ ===
    vf->first_vector_idx = ice_virt_get_irqs(pf, vf->num_msix);
    if (vf->first_vector_idx < 0)
        return -ENOMEM;

    // === 步驟 2: 創建 VSI（Virtual Station Interface）===
    vsi = ice_vf_vsi_setup(vf);
    if (!vsi)
        return -ENOMEM;

    // === 步驟 3: 初始化 Host 配置 ===
    // 設置 MAC 過濾器、VLAN、廣播等
    err = ice_vf_init_host_cfg(vf, vsi);

    return 0;
}
```

**VSI 的作用**:
- VSI (Virtual Station Interface) 是 Intel 網卡的核心概念
- **每個 VF 有一個獨立的 VSI**，實現流量隔離
- VSI 包含：
  - Queue 集合（Tx/Rx rings）
  - 過濾器規則（MAC, VLAN）
  - 統計計數器

---

### 2.5 ice_ena_vf_mappings() - 硬件映射配置

**文件**: `ice_sriov.c:323`

這是最關鍵的一步：**將軟件資源映射到硬件寄存器**。

#### 2.5.1 MSI-X 映射

**函數**: `ice_ena_vf_msix_mappings()` [ice_sriov.c:230]

```c
static void ice_ena_vf_msix_mappings(struct ice_vf *vf)
{
    struct ice_pf *pf = vf->pf;
    struct ice_hw *hw = &pf->hw;

    // === 計算 MSI-X 向量範圍 ===
    // PF 空間內的索引
    pf_based_first_msix = vf->first_vector_idx;
    pf_based_last_msix = (pf_based_first_msix + vf->num_msix) - 1;

    // 設備全局空間內的索引
    device_based_first_msix = pf_based_first_msix +
        pf->hw.func_caps.common_cap.msix_vector_first_id;
    device_based_last_msix = (device_based_first_msix + vf->num_msix) - 1;

    // 設備全局 VF ID
    device_based_vf_id = vf->vf_id + hw->func_caps.vf_base_id;

    // === 硬件寄存器配置 ===

    // 1. VPINT_ALLOC: 分配中斷範圍給 VF
    reg = FIELD_PREP(VPINT_ALLOC_FIRST_M, device_based_first_msix) |
          FIELD_PREP(VPINT_ALLOC_LAST_M, device_based_last_msix) |
          VPINT_ALLOC_VALID_M;
    wr32(hw, VPINT_ALLOC(vf->vf_id), reg);

    // 2. VPINT_ALLOC_PCI: PCI 層面的中斷分配
    reg = FIELD_PREP(VPINT_ALLOC_PCI_FIRST_M, device_based_first_msix) |
          FIELD_PREP(VPINT_ALLOC_PCI_LAST_M, device_based_last_msix) |
          VPINT_ALLOC_PCI_VALID_M;
    wr32(hw, VPINT_ALLOC_PCI(vf->vf_id), reg);

    // 3. GLINT_VECT2FUNC: 將每個中斷向量映射到 VF function
    for (v = pf_based_first_msix; v <= pf_based_last_msix; v++) {
        reg = FIELD_PREP(GLINT_VECT2FUNC_VF_NUM_M, device_based_vf_id) |
              FIELD_PREP(GLINT_VECT2FUNC_PF_NUM_M, hw->pf_id);
        wr32(hw, GLINT_VECT2FUNC(v), reg);
    }

    // 4. VPINT_MBX_CTL: Mailbox 中斷配置
    // 將 mailbox 中斷映射到 VF 的第 0 個 MSI-X 向量
    wr32(hw, VPINT_MBX_CTL(device_based_vf_id), VPINT_MBX_CTL_CAUSE_ENA_M);
}
```

**寄存器說明**:

| 寄存器名稱          | 作用                                           |
| ------------------- | ---------------------------------------------- |
| `VPINT_ALLOC`       | 告訴 VF 它擁有哪些中斷向量（第一個到最後一個）|
| `VPINT_ALLOC_PCI`   | PCI 層面的中斷分配（用於 PCIe MSI-X table）    |
| `GLINT_VECT2FUNC`   | 全局向量到 function 的映射（指定向量屬於哪個 VF）|
| `VPINT_MBX_CTL`     | Mailbox 中斷控制（VF-PF 通信）                 |

**地址空間轉換**:
```
PF space:      [0 ... first_vector_idx ... last]
                           |
                           | + msix_vector_first_id
                           v
Device space:  [0 ... device_first ... device_last ... MAX]
```

#### 2.5.2 Queue 映射

**函數**: `ice_ena_vf_q_mappings()` [ice_sriov.c:276]

```c
static void ice_ena_vf_q_mappings(struct ice_vf *vf, u16 max_txq, u16 max_rxq)
{
    struct ice_vsi *vsi = ice_get_vf_vsi(vf);
    struct ice_hw *hw = &vf->pf->hw;

    // === Tx Queue 映射 ===

    // 1. 啟用 Tx queue mapping
    wr32(hw, VPLAN_TXQ_MAPENA(vf->vf_id), VPLAN_TXQ_MAPENA_TX_ENA_M);

    // 2. 配置 Tx queue 基址和數量
    if (vsi->tx_mapping_mode == ICE_VSI_MAP_CONTIG) {
        // 連續模式：告訴 VF 它的 queue 從哪裡開始，有多少個
        reg = FIELD_PREP(VPLAN_TX_QBASE_VFFIRSTQ_M, vsi->txq_map[0]) |
              FIELD_PREP(VPLAN_TX_QBASE_VFNUMQ_M, max_txq - 1);
        wr32(hw, VPLAN_TX_QBASE(vf->vf_id), reg);
    } else {
        // 分散模式：尚未實現
        dev_err(dev, "Scattered mode not implemented\n");
    }

    // === Rx Queue 映射（邏輯同 Tx）===
    wr32(hw, VPLAN_RXQ_MAPENA(vf->vf_id), VPLAN_RXQ_MAPENA_RX_ENA_M);

    if (vsi->rx_mapping_mode == ICE_VSI_MAP_CONTIG) {
        reg = FIELD_PREP(VPLAN_RX_QBASE_VFFIRSTQ_M, vsi->rxq_map[0]) |
              FIELD_PREP(VPLAN_RX_QBASE_VFNUMQ_M, max_rxq - 1);
        wr32(hw, VPLAN_RX_QBASE(vf->vf_id), reg);
    }
}
```

**Queue 映射模式**:

1.  **ICE_VSI_MAP_CONTIG (連續模式)** - 當前實現:
    ```
    PF Queue Space:  [0 1 2 3 4 5 6 7 8 9 10 11 ...]
                            ↑       ↑
    VF0 Queues:            [3 4 5 6]    <- VFFIRSTQ=3, VFNUMQ=3 (4-1)
    VF1 Queues:                    [7 8 9 10]
    ```

2.  **ICE_VSI_MAP_SCATTER (分散模式)** - 未實現:
    ```
    PF Queue Space:  [0 1 2 3 4 5 6 7 8 9 10 11 ...]
    VF0 Queues:         [1   3   5   7]    <- 需要額外的映射表
    ```

---

### 2.6 VF 狀態轉換

**文件**: `ice_sriov.c:467`

```c
static int ice_start_vfs(struct ice_pf *pf)
{
    ice_for_each_vf(pf, bkt, vf) {
        // 1. 清除 reset trigger
        vf->vf_ops->clear_reset_trigger(vf);

        // 2. 初始化 VSI 資源
        ice_init_vf_vsi_res(vf);

        // 3. 啟用硬件映射
        ice_ena_vf_mappings(vf);

        // 4. 設置狀態為 INIT
        set_bit(ICE_VF_STATE_INIT, vf->vf_states);

        // 5. 通知硬件 VF 已就緒
        wr32(hw, VFGEN_RSTAT(vf->vf_id), VIRTCHNL_VFR_VFACTIVE);
    }

    ice_flush(hw);  // 確保寄存器寫入完成
    return 0;
}
```

**VFGEN_RSTAT** 寄存器值:
- `VIRTCHNL_VFR_VFACTIVE` (1): VF 已就緒，可以開始通信
- VF driver 會輪詢這個狀態，知道自己可以開始初始化

---

## 3. PCIe 層互動

### 3.1 SR-IOV Capability 讀取

**代碼**: `ice_sriov.c:694`

```c
pos = pci_find_ext_capability(pdev, PCI_EXT_CAP_ID_SRIOV);
pci_read_config_word(pdev, pos + PCI_SRIOV_VF_DID, &vf_pdev_id);
```

**PCIe Config Space 結構** (SR-IOV Capability):
```
Offset  | Field                | Description
--------|----------------------|------------------------------------
0x00    | Capability ID        | 0x10 (SR-IOV Extended Capability)
0x04    | SR-IOV Capabilities  | 支持的功能
0x08    | SR-IOV Control       | 啟用/禁用 VF
0x0A    | SR-IOV Status        | VF 狀態
0x0C    | InitialVFs           | 初始 VF 數量
0x0E    | TotalVFs             | 最大 VF 數量
0x10    | NumVFs               | 當前啟用的 VF 數量
0x14    | First VF Offset      | 第一個 VF 的 BDF offset
0x16    | VF Stride            | VF 之間的 BDF 間隔
0x18    | VF Device ID         | VF 的 Device ID
0x1A    | Supported Page Sizes | 支持的 page 大小
0x24    | VF BAR0              | VF BAR0 基址
...
```

### 3.2 pci_enable_sriov() - Kernel 層 SR-IOV 啟用

**調用位置**: `ice_sriov.c:754`

```c
ret = pci_enable_sriov(pf->pdev, num_vfs);
```

**這個函數做什麼**:
1.  寫入 PCIe Config Space 的 `NumVFs` 字段
2.  啟用 SR-IOV Control 寄存器中的 VF Enable bit
3.  為每個 VF 創建 PCI 設備 (`struct pci_dev`)
4.  分配 BDF (Bus:Device.Function)

**結果**: 系統中出現 N 個新的 PCIe 設備（VF），可以被 IOMMU 管理

---

## 4. 資源分配流程圖

```
┌─────────────────────────────────────────────────────────┐
│  ice_set_per_vf_res()                                   │
│  ├─ 計算可用資源                                         │
│  │  ├─ Total MSI-X: virt_irq_tracker.num_entries       │
│  │  ├─ Total Tx queues: avail_txqs bitmap              │
│  │  └─ Total Rx queues: avail_rxqs bitmap              │
│  │                                                       │
│  ├─ 分配策略（按優先級）:                                │
│  │  1. MSI-X vectors (每 VF)                            │
│  │  2. Tx queue pairs (受 MSI-X 和可用 queue 限制)      │
│  │  3. Rx queue pairs (必須與 Tx 對稱)                  │
│  │                                                       │
│  └─ 結果:                                                │
│     pf->vfs.num_msix_per = X                            │
│     pf->vfs.num_qps_per = Y                             │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  ice_init_vf_vsi_res()                                  │
│  ├─ ice_virt_get_irqs(pf, vf->num_msix)                │
│  │  └─ 從 virt_irq_tracker 分配 MSI-X                   │
│  │     Result: vf->first_vector_idx                     │
│  │                                                       │
│  └─ ice_vf_vsi_setup(vf)                                │
│     └─ 創建 VSI 並分配 Tx/Rx queues                     │
│        Result: vsi->txq_map[], vsi->rxq_map[]           │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│  ice_ena_vf_mappings()                                  │
│  ├─ ice_ena_vf_msix_mappings()                          │
│  │  ├─ VPINT_ALLOC(vf_id) = [first_msix, last_msix]    │
│  │  ├─ VPINT_ALLOC_PCI(vf_id) = [...]                  │
│  │  ├─ FOR each vector:                                 │
│  │  │  └─ GLINT_VECT2FUNC(v) = VF_NUM | PF_NUM         │
│  │  └─ VPINT_MBX_CTL(vf_id) = ENABLE                   │
│  │                                                       │
│  └─ ice_ena_vf_q_mappings()                             │
│     ├─ VPLAN_TXQ_MAPENA(vf_id) = ENABLE                │
│     ├─ VPLAN_TX_QBASE(vf_id) = [first_q, num_q]        │
│     ├─ VPLAN_RXQ_MAPENA(vf_id) = ENABLE                │
│     └─ VPLAN_RX_QBASE(vf_id) = [first_q, num_q]        │
└─────────────────────────────────────────────────────────┘
```

---

## 5. 設計重點與代碼審查

### 5.1 資源分配邏輯

*   **分級分配**:
    *   **設計**: `ice_set_per_vf_res` 實現了資源的動態調整。
    *   **重點**: 優先保證 VF 能跑起來 (Small config)，資源充裕時再給更多 (Medium config)。

### 5.2 代碼異味 (Code Smell)

*   **Magic Numbers**:
    *   **問題**: `ICE_NUM_VF_MSIX_MED` 等常數缺乏上下文解釋。
    *   **改進**: 應在定義處增加註釋說明數值來源 (e.g., 16 Queues + 1 Control Vector)。

*   **冗餘檢查**:
    *   **問題**: `ice_ena_vf_q_mappings` 中檢查了 `tx_mapping_mode`，但目前只支持 `CONTIG` 模式。
    *   **改進**: 如果不支持 Scatter 模式，應在初始化階段就拒絕，而不是在熱路徑 (Hot Path) 中反覆檢查。

---

## 階段 2: VF Driver Probe & Virtchnl 協商

當 PF 完成 VF 硬件初始化後，VF driver (iavf) 會偵測到新的 PCIe 設備並開始初始化。VF 和 PF 之間通過 **mailbox (virtchnl)** 通信。

### 2.1 Mailbox 消息分發器

**文件**: `virt/virtchnl.c:2736`

```c
void ice_vc_process_vf_msg(struct ice_pf *pf,
                           struct ice_rq_event_info *event)
{
    u32 v_opcode = le32_to_cpu(event->desc.cookie_high);
    s16 vf_id = le16_to_cpu(event->desc.retval);
    const struct ice_virtchnl_ops *ops;
    struct ice_vf *vf = NULL;
    u16 msglen = event->msg_len;
    u8 *msg = event->msg_buf;

    // 1. 從 hash table 查找 VF (RCU 保護)
    vf = ice_get_vf_by_id(pf, vf_id);
    if (!vf) {
        dev_err(dev, "Unable to locate VF for message from VF ID %d\n", vf_id);
        return;
    }

    // 2. 獲取操作函數表
    device_lock(&vf->dev);
    ops = ice_vc_get_ops(vf);

    // 3. 消息分發
    switch (v_opcode) {
    case VIRTCHNL_OP_VERSION:
        err = ops->get_ver_msg(vf, msg);
        break;
    case VIRTCHNL_OP_GET_VF_RESOURCES:
        err = ops->get_vf_res_msg(vf, msg);
        break;
    case VIRTCHNL_OP_CONFIG_VSI_QUEUES:
        err = ops->cfg_qs_msg(vf, msg, msglen);
        break;
    case VIRTCHNL_OP_ENABLE_QUEUES:
        err = ops->ena_qs_msg(vf, msg);
        break;
    case VIRTCHNL_OP_DISABLE_QUEUES:
        err = ops->dis_qs_msg(vf, msg);
        break;
    case VIRTCHNL_OP_CONFIG_IRQ_MAP:
        err = ops->cfg_irq_map_msg(vf, msg);
        break;
    // ... 30+ 種消息類型
    default:
        dev_err(dev, "Unsupported opcode %d from VF %d\n", v_opcode, vf_id);
        err = ice_vc_send_msg_to_vf(vf, v_opcode, VIRTCHNL_STATUS_ERR_NOT_SUPPORTED,
                                     NULL, 0);
    }

    device_unlock(&vf->dev);
    ice_put_vf(vf);
}
```

**關鍵設計**:
- **RCU 查找**: `ice_get_vf_by_id()` 使用 RCU 無鎖查找，適合高頻 mailbox 消息
- **操作函數表**: `ops` 指針允許不同類型的 VF 有不同的處理邏輯
- **狀態驗證**: 每個 handler 都會檢查 VF 是否處於正確狀態

---

### 2.2 版本協商

**文件**: `virt/virtchnl.c:99`

VF 啟動後發送的第一個消息是 `VIRTCHNL_OP_VERSION`。

```c
int ice_vc_get_ver_msg(struct ice_vf *vf, u8 *msg)
{
    struct virtchnl_version_info info = {
        VIRTCHNL_VERSION_MAJOR, VIRTCHNL_VERSION_MINOR
    };

    vf->vf_ver = *(struct virtchnl_version_info *)msg;

    // PF 回復自己支持的版本
    return ice_vc_send_msg_to_vf(vf, VIRTCHNL_OP_VERSION,
                                  VIRTCHNL_STATUS_SUCCESS,
                                  (u8 *)&info, sizeof(info));
}
```

**協商結果**: PF 和 VF 確定使用的 virtchnl 協議版本

---

### 2.3 資源協商 - 關鍵步驟

**文件**: `virt/virtchnl.c:245`

VF 發送 `VIRTCHNL_OP_GET_VF_RESOURCES` 請求，PF 告知 VF 可用的資源。

```c
static int ice_vc_get_vf_res_msg(struct ice_vf *vf, u8 *msg)
{
    struct virtchnl_vf_resource *vfres = NULL;
    struct ice_vsi *vsi;
    int len = 0;
    int ret;

    // 檢查 VF 狀態
    if (ice_check_vf_init(vf)) {
        v_ret = VIRTCHNL_STATUS_ERR_PARAM;
        goto err;
    }

    // 計算回復消息大小
    len = virtchnl_get_vf_resource_len(1);  // 1 VSI
    vfres = kzalloc(len, GFP_KERNEL);
    if (!vfres) {
        v_ret = VIRTCHNL_STATUS_ERR_NO_MEMORY;
        len = 0;
        goto err;
    }

    // === 告知 VF 支持的 capabilities ===
    vfres->vf_cap_flags = VIRTCHNL_VF_OFFLOAD_L2;
    if (vf->driver_caps & VIRTCHNL_VF_OFFLOAD_RSS_PF)
        vfres->vf_cap_flags |= VIRTCHNL_VF_OFFLOAD_RSS_PF;
    if (vf->driver_caps & VIRTCHNL_VF_OFFLOAD_VLAN)
        vfres->vf_cap_flags |= VIRTCHNL_VF_OFFLOAD_VLAN;
    // ... 更多 capability 協商

    vsi = ice_get_vf_vsi(vf);
    if (!vsi) {
        v_ret = VIRTCHNL_STATUS_ERR_PARAM;
        goto err;
    }

    // === 告知 VF 分配的資源 ===
    vfres->num_vsis = 1;
    vfres->num_queue_pairs = vsi->num_txq;  // Queue pairs
    vfres->max_vectors = vf->num_msix;       // MSI-X vectors
    vfres->rss_key_size = ICE_VSIQF_HKEY_ARRAY_SIZE;
    vfres->rss_lut_size = ICE_VSIQF_HLUT_ARRAY_SIZE;

    // === 填充 VSI 資訊 ===
    vfres->vsi_res[0].vsi_id = ICE_VF_VSI_ID;
    vfres->vsi_res[0].vsi_type = VIRTCHNL_VSI_SRIOV;
    vfres->vsi_res[0].num_queue_pairs = vsi->num_txq;

    // === 重要: 告知 VF 的 MAC 地址 ===
    ether_addr_copy(vfres->vsi_res[0].default_mac_addr,
                    vf->hw_lan_addr);

    // === VF 狀態轉換: INIT → ACTIVE ===
    set_bit(ICE_VF_STATE_ACTIVE, vf->vf_states);

err:
    ret = ice_vc_send_msg_to_vf(vf, VIRTCHNL_OP_GET_VF_RESOURCES,
                                 v_ret, (u8 *)vfres, len);
    kfree(vfres);
    return ret;
}
```

**關鍵資訊交換**:
1.  **Capability flags**: VF 知道自己支持哪些 offload (RSS, VLAN, checksum 等)
2.  **Queue pairs**: VF 知道可以使用幾個 Tx/Rx queue
3.  **MSI-X vectors**: VF 知道有幾個中斷向量
4.  **MAC 地址**: PF 分配的 MAC address
5.  **狀態轉換**: `ICE_VF_STATE_ACTIVE` - VF 現在可以開始配置 queues

---

## 階段 3: Queue 配置與啟用

### 3.1 Queue 配置 - VF 提供 DMA 地址

**文件**: `virt/queues.c:749`

VF driver 分配好 descriptor ring 的內存後，通過 `VIRTCHNL_OP_CONFIG_QUEUES` 告訴 PF 這些 rings 的 DMA 地址。

```c
int ice_vc_cfg_qs_msg(struct ice_vf *vf, u8 *msg, u16 msglen)
{
    struct virtchnl_vsi_queue_config_info *qci =
        (struct virtchnl_vsi_queue_config_info *)msg;
    struct virtchnl_queue_pair_info *qpi;
    struct ice_vsi *vsi;
    int i, q_idx;

    // 狀態檢查
    if (!test_bit(ICE_VF_STATE_ACTIVE, vf->vf_states)) {
        v_ret = VIRTCHNL_STATUS_ERR_PARAM;
        goto error_param;
    }

    vsi = ice_get_vf_vsi(vf);
    if (!vsi) {
        v_ret = VIRTCHNL_STATUS_ERR_PARAM;
        goto error_param;
    }

    // === 逐個配置 queue pair ===
    for (i = 0; i < qci->num_queue_pairs; i++) {
        qpi = &qci->qpair[i];

        // VF 提供的 queue ID (VF 視角: 0-based)
        if (qpi->txq.queue_id >= vsi->alloc_txq ||
            qpi->rxq.queue_id >= vsi->alloc_rxq) {
            v_ret = VIRTCHNL_STATUS_ERR_PARAM;
            goto error_param;
        }

        q_idx = qpi->txq.queue_id;

        // === 配置 Tx Queue ===
        // VF 提供 descriptor ring 的 DMA 地址和大小
        vsi->tx_rings[q_idx]->dma = qpi->txq.dma_ring_addr;
        vsi->tx_rings[q_idx]->count = qpi->txq.ring_len;

        // PF 配置硬件 Tx queue
        if (ice_vsi_cfg_single_txq(vsi, vsi->tx_rings, q_idx)) {
            v_ret = VIRTCHNL_STATUS_ERR_PARAM;
            goto error_param;
        }

        // === 配置 Rx Queue ===
        // VF 同樣提供 descriptor ring 的 DMA 地址和大小
        vsi->rx_rings[q_idx]->dma = qpi->rxq.dma_ring_addr;
        vsi->rx_rings[q_idx]->count = qpi->rxq.ring_len;

        // 設置 Rx buffer 大小
        vsi->rx_rings[q_idx]->rx_buf_len = qpi->rxq.databuffer_size;

        // 配置 Rx 的 packet split 模式
        if (qpi->rxq.splithdr_enabled) {
            vsi->rx_rings[q_idx]->hsplit_mode = ICE_HSPLIT_ENABLED;
        }

        // PF 配置硬件 Rx queue
        if (ice_vsi_cfg_single_rxq(vsi, q_idx)) {
            v_ret = VIRTCHNL_STATUS_ERR_PARAM;
            goto error_param;
        }
    }

error_param:
    return ice_vc_send_msg_to_vf(vf, VIRTCHNL_OP_CONFIG_QUEUES,
                                  v_ret, NULL, 0);
}
```

**重要概念**:
1.  **VF 分配內存**: VF driver 在自己的地址空間分配 descriptor ring
2.  **DMA mapping**: VF driver 呼叫 `dma_map_*()` 得到 IOVA (IO Virtual Address)
3.  **告知 PF**: VF 將 IOVA 通過 mailbox 告訴 PF
4.  **PF 配置硬件**: PF 將這些 IOVA 寫入硬件寄存器

**數據流**:
```
VF 地址空間:
  kzalloc() → Virtual Address (VA)
       ↓
  dma_map_single() → IOVA (通過 IOMMU 轉換)
       ↓
  Mailbox → 告訴 PF
       ↓
PF:
  ice_vsi_cfg_single_txq() → 寫入硬件寄存器
       ↓
硬件:
  使用 IOVA 進行 DMA 操作
```

---

### 3.2 Queue 啟用 - 開始接收封包

**文件**: `virt/queues.c:234`

配置完成後，VF 發送 `VIRTCHNL_OP_ENABLE_QUEUES` 啟用 queues。

```c
int ice_vc_ena_qs_msg(struct ice_vf *vf, u8 *msg)
{
    struct virtchnl_queue_select *vqs =
        (struct virtchnl_queue_select *)msg;
    struct ice_vsi *vsi;
    unsigned long q_map;
    u16 vf_q_id;

    if (!test_bit(ICE_VF_STATE_ACTIVE, vf->vf_states)) {
        v_ret = VIRTCHNL_STATUS_ERR_PARAM;
        goto error_param;
    }

    vsi = ice_get_vf_vsi(vf);
    if (!vsi) {
        v_ret = VIRTCHNL_STATUS_ERR_PARAM;
        goto error_param;
    }

    // === 啟用請求的 Rx queues ===
    q_map = vqs->rx_queues;
    for_each_set_bit(vf_q_id, &q_map, ICE_MAX_RSS_QS_PER_VF) {
        // 驗證 queue ID
        if (!ice_vc_isvalid_q_id(vf, vqs->vsi_id, vf_q_id)) {
            v_ret = VIRTCHNL_STATUS_ERR_PARAM;
            goto error_param;
        }

        // === Rx queue 需要 driver 明確啟用 ===
        // 這裡呼叫底層函數啟用 Rx queue
        if (ice_vsi_ctrl_one_rx_ring(vsi, true, vf_q_id, true)) {
            dev_err(dev, "Failed to enable Rx ring %d on VSI %d\n",
                    vf_q_id, vsi->vsi_num);
            v_ret = VIRTCHNL_STATUS_ERR_PARAM;
            goto error_param;
        }

        // 標記此 queue 已啟用
        set_bit(vf_q_id, vf->rxq_ena);
    }

    // === 啟用請求的 Tx queues ===
    q_map = vqs->tx_queues;
    for_each_set_bit(vf_q_id, &q_map, ICE_MAX_RSS_QS_PER_VF) {
        if (!ice_vc_isvalid_q_id(vf, vqs->vsi_id, vf_q_id)) {
            v_ret = VIRTCHNL_STATUS_ERR_PARAM;
            goto error_param;
        }

        // === 注意: Tx rings 由 firmware 自動啟用 ===
        // 代碼註釋: "Tx rings were enabled by the FW when the config queues msg was received"
        // 所以這裡不需要額外操作，只需標記狀態

        set_bit(vf_q_id, vf->txq_ena);
    }

    // === 設置 VF 狀態: Queues Enabled ===
    set_bit(ICE_VF_STATE_QS_ENA, vf->vf_states);

error_param:
    return ice_vc_send_msg_to_vf(vf, VIRTCHNL_OP_ENABLE_QUEUES,
                                  v_ret, NULL, 0);
}
```

**Tx 和 Rx 的不同處理**:
- **Rx Queue**: 需要 driver 明確呼叫 `ice_vsi_ctrl_one_rx_ring()` 啟用
- **Tx Queue**: Firmware 在收到 `CONFIG_QUEUES` 時已經啟用，這裡只是記錄狀態

**為什麼 Tx 和 Rx 不同？**:
- **Tx**: 需要立即準備好發送，所以提前啟用
- **Rx**: 需要等 VF driver 先分配好 buffer (ice_alloc_rx_bufs)，否則封包來了沒地方放

---

## 階段 4: Runtime 封包處理

### 4.1 Rx 封包接收路徑完整代碼追蹤

**流程**: 網路封包到達 → 硬件 DMA → Descriptor ring → MSI-X 中斷 → NAPI poll → ice_clean_rx_irq()

#### 4.1.1 NAPI Poll - 中斷處理入口

**文件**: `ice_txrx.c:1702`

```c
int ice_napi_poll(struct napi_struct *napi, int budget)
{
    struct ice_q_vector *q_vector =
                container_of(napi, struct ice_q_vector, napi);
    struct ice_tx_ring *tx_ring;
    struct ice_rx_ring *rx_ring;
    bool clean_complete = true;
    int work_done = 0;

    // === 先處理 Tx completion ===
    ice_for_each_tx_ring(tx_ring, q_vector->tx) {
        bool wd = ice_clean_tx_irq(tx_ring, budget);
        if (!wd)
            clean_complete = false;
    }

    // === 處理 Rx packets ===
    ice_for_each_rx_ring(rx_ring, q_vector->rx) {
        int cleaned;

        // 呼叫 Rx 處理函數
        cleaned = ice_clean_rx_irq(rx_ring, budget_per_ring);
        work_done += cleaned;

        // 如果處理了 budget 數量的封包，表示還有更多
        if (cleaned >= budget_per_ring)
            clean_complete = false;
    }

    // 如果全部處理完畢，重新啟用中斷
    if (clean_complete) {
        napi_complete_done(napi, work_done);
        ice_enable_interrupt(q_vector);
    }

    return work_done;
}
```

---

#### 4.1.2 Rx Descriptor 處理

**文件**: `ice_txrx.c:1381`

```c
static int ice_clean_rx_irq(struct ice_rx_ring *rx_ring, int budget)
{
    unsigned int total_rx_bytes = 0, total_rx_pkts = 0;
    u32 ntc = rx_ring->next_to_clean;  // Next To Clean index
    u32 cnt = rx_ring->count;

    // === 循環處理 Rx descriptors (受 budget 限制) ===
    while (likely(total_rx_pkts < (unsigned int)budget)) {
        union ice_32b_rx_flex_desc *rx_desc;
        struct ice_rx_buf *rx_buf;
        struct sk_buff *skb;
        unsigned int size;
        u16 stat_err_bits;

        // === 1. 從 descriptor ring 取得下一個 descriptor ===
        rx_desc = ICE_RX_DESC(rx_ring, ntc);

        // === 2. 檢查 DD (Descriptor Done) bit ===
        // 硬件寫入封包後會設置此 bit
        stat_err_bits = BIT(ICE_RX_FLEX_DESC_STATUS0_DD_S);
        if (!ice_test_staterr(rx_desc->wb.status_error0, stat_err_bits))
            break;  // 沒有新封包，退出

        // === 3. Memory barrier - 確保先讀取 DD bit ===
        dma_rmb();

        // === 4. 從 descriptor 讀取封包長度 ===
        size = le16_to_cpu(rx_desc->wb.pkt_len) & ICE_RX_FLX_DESC_PKT_LEN_M;

        // === 5. 取得對應的 buffer ===
        rx_buf = ice_get_rx_buf(rx_ring, size, ntc);

        // === 6. 推進 next_to_clean index ===
        if (++ntc == cnt)
            ntc = 0;

        // === 7. 構建 skb (socket buffer) ===
        // 從 DMA buffer 轉換為 kernel 的 skb
        if (likely(ice_ring_uses_build_skb(rx_ring)))
            skb = ice_build_skb(rx_ring, xdp);
        else
            skb = ice_construct_skb(rx_ring, xdp);

        if (!skb) {
            rx_ring->ring_stats->rx_stats.alloc_buf_failed++;
            break;
        }

        // === 8. 檢查錯誤 ===
        stat_err_bits = BIT(ICE_RX_FLEX_DESC_STATUS0_RXE_S);
        if (unlikely(ice_test_staterr(rx_desc->wb.status_error0, stat_err_bits))) {
            dev_kfree_skb_any(skb);  // 有錯誤，丟棄封包
            continue;
        }

        // === 9. 處理 VLAN tag ===
        vlan_tci = ice_get_vlan_tci(rx_desc);

        // === 10. 填充 skb metadata (checksum, protocol, VLAN) ===
        ice_process_skb_fields(rx_ring, rx_desc, skb);

        // === 11. 將 skb 送往網路協議棧 ===
        ice_receive_skb(rx_ring, skb, vlan_tci);

        // 統計
        total_rx_bytes += skb->len;
        total_rx_pkts++;
    }

    // === 12. 更新 next_to_clean ===
    rx_ring->next_to_clean = ntc;

    // === 13. 補充新的 Rx buffers ===
    // 處理掉一些 buffer 後，需要分配新的 buffer 並告訴硬件
    failure = ice_alloc_rx_bufs(rx_ring, ICE_RX_DESC_UNUSED(rx_ring));

    // 更新統計
    ice_update_rx_ring_stats(rx_ring, total_rx_pkts, total_rx_bytes);

    return failure ? budget : (int)total_rx_pkts;
}
```

---

#### 4.1.3 Rx Buffer 補充 - DMA Mapping

**文件**: `ice_txrx.c:883`

```c
bool ice_alloc_rx_bufs(struct ice_rx_ring *rx_ring, unsigned int cleaned_count)
{
    union ice_32b_rx_flex_desc *rx_desc;
    u16 ntu = rx_ring->next_to_use;
    struct ice_rx_buf *bi;

    rx_desc = ICE_RX_DESC(rx_ring, ntu);
    bi = &rx_ring->rx_buf[ntu];

    do {
        // === 1. 分配新的 page ===
        if (!ice_alloc_mapped_page(rx_ring, bi))
            break;

        // === 2. DMA sync - 確保 device 可以訪問 ===
        dma_sync_single_range_for_device(rx_ring->dev, bi->dma,
                                         bi->page_offset,
                                         rx_ring->rx_buf_len,
                                         DMA_FROM_DEVICE);

        // === 3. 將 DMA 地址寫入 descriptor ===
        // 硬件會使用這個地址進行 DMA 寫入
        rx_desc->read.pkt_addr = cpu_to_le64(bi->dma + bi->page_offset);

        // 推進 index
        rx_desc++;
        bi++;
        ntu++;
        if (unlikely(ntu == rx_ring->count)) {
            rx_desc = ICE_RX_DESC(rx_ring, 0);
            bi = rx_ring->rx_buf;
            ntu = 0;
        }

        // === 4. 清除 status bits ===
        // 準備讓硬件寫入新封包
        rx_desc->wb.status_error0 = 0;

        cleaned_count--;
    } while (cleaned_count);

    // === 5. 更新 next_to_use 並 ring doorbell ===
    if (rx_ring->next_to_use != ntu)
        ice_release_rx_desc(rx_ring, ntu);

    return !!cleaned_count;
}
```

**Rx 完整流程總結**:
```
1. 網路封包到達網卡
2. 硬件執行 DMA: 封包數據 → rx_desc->read.pkt_addr (VF 提供的 IOVA)
3. 硬件設置 DD bit in descriptor
4. 硬件觸發 MSI-X 中斷 → VF 的中斷向量
5. VF driver 的 NAPI poll 被喚醒
6. ice_clean_rx_irq():
   - 檢查 DD bit
   - 讀取封包長度
   - 構建 skb
   - 傳給網路協議棧
7. ice_alloc_rx_bufs():
   - 分配新 buffer
   - DMA mapping
   - 寫入 descriptor
   - Ring doorbell (更新 tail pointer)
```

---

### 4.2 Tx 封包發送路徑完整代碼追蹤

**流程**: 網路協議棧 → ice_start_xmit() → ice_xmit_frame_ring() → ice_tx_map() → 硬件 DMA → 網路

#### 4.2.1 Tx 入口

**文件**: `ice_txrx.c:2701`

```c
netdev_tx_t ice_start_xmit(struct sk_buff *skb, struct net_device *netdev)
{
    struct ice_netdev_priv *np = netdev_priv(netdev);
    struct ice_vsi *vsi = np->vsi;
    struct ice_tx_ring *tx_ring;

    // === 選擇 Tx queue (基於 skb->queue_mapping) ===
    tx_ring = vsi->tx_rings[skb->queue_mapping];

    // === 確保封包長度符合硬件最小要求 ===
    if (skb_put_padto(skb, ICE_MIN_TX_LEN))
        return NETDEV_TX_OK;

    // 呼叫實際發送函數
    return ice_xmit_frame_ring(skb, tx_ring);
}
```

---

#### 4.2.2 Tx Descriptor 構建

**文件**: `ice_txrx.c:2586`

```c
static netdev_tx_t
ice_xmit_frame_ring(struct sk_buff *skb, struct ice_tx_ring *tx_ring)
{
    struct ice_tx_offload_params offload = { 0 };
    struct ice_vsi *vsi = tx_ring->vsi;
    struct ice_tx_buf *first;
    unsigned int count;

    // === 1. 計算需要多少個 descriptors ===
    count = ice_xmit_desc_count(skb);

    // === 2. 檢查 descriptor ring 是否有足夠空間 ===
    if (ice_maybe_stop_tx(tx_ring, count + ICE_DESCS_PER_CACHE_LINE +
                          ICE_DESCS_FOR_CTX_DESC)) {
        tx_ring->ring_stats->tx_stats.tx_busy++;
        return NETDEV_TX_BUSY;
    }

    // === 3. 記錄第一個 descriptor 位置 ===
    first = &tx_ring->tx_buf[tx_ring->next_to_use];
    first->skb = skb;
    first->type = ICE_TX_BUF_SKB;
    first->bytecount = max_t(unsigned int, skb->len, ETH_ZLEN);

    // === 4. 準備 VLAN tagging ===
    ice_tx_prepare_vlan_flags(tx_ring, first);

    // === 5. 設置 TSO offload (如果需要) ===
    tso = ice_tso(first, &offload);
    if (tso < 0)
        goto out_drop;

    // === 6. 設置 checksum offload ===
    csum = ice_tx_csum(first, &offload);
    if (csum < 0)
        goto out_drop;

    // === 7. 如果需要 context descriptor，先寫入 ===
    if (offload.cd_qw1 & ICE_TX_DESC_DTYPE_CTX) {
        struct ice_tx_ctx_desc *cdesc;
        u16 i = tx_ring->next_to_use;

        cdesc = ICE_TX_CTX_DESC(tx_ring, i);
        i++;
        tx_ring->next_to_use = (i < tx_ring->count) ? i : 0;

        cdesc->tunneling_params = cpu_to_le32(offload.cd_tunnel_params);
        cdesc->l2tag2 = cpu_to_le16(offload.cd_l2tag2);
        cdesc->qw1 = cpu_to_le64(offload.cd_qw1);
    }

    // === 8. 執行 DMA mapping 並寫入 data descriptors ===
    ice_tx_map(tx_ring, first, &offload);

    return NETDEV_TX_OK;

out_drop:
    dev_kfree_skb_any(skb);
    return NETDEV_TX_OK;
}
```

**關鍵概念**:
1.  **Descriptor ring 空間檢查**: 如果 ring 滿了，返回 `NETDEV_TX_BUSY`，協議棧會稍後重試
2.  **Context descriptor**: 用於 TSO、VLAN、tunnel 等特殊功能
3.  **Data descriptor**: 包含實際封包數據的 DMA 地址

---

#### 4.2.3 DMA Mapping 和 Doorbell

**文件**: `ice_txrx_lib.c` (實際代碼略，核心邏輯如下)

```c
void ice_tx_map(struct ice_tx_ring *tx_ring,
                struct ice_tx_buf *first,
                struct ice_tx_offload_params *off)
{
    struct ice_tx_desc *tx_desc;
    struct ice_tx_buf *tx_buf;
    struct sk_buff *skb = first->skb;
    skb_frag_t *frag;
    dma_addr_t dma;
    u16 i = tx_ring->next_to_use;

    // === 1. Map skb head (linear data) ===
    dma = dma_map_single(tx_ring->dev, skb->data,
                         skb_headlen(skb), DMA_TO_DEVICE);
    if (dma_mapping_error(tx_ring->dev, dma))
        goto dma_error;

    // === 2. 寫入第一個 data descriptor ===
    tx_desc = ICE_TX_DESC(tx_ring, i);
    tx_buf = first;

    tx_desc->buf_addr = cpu_to_le64(dma);
    tx_desc->cmd_type_offset_bsz = ice_build_ctob(td_cmd, td_offset,
                                                   size, td_tag);

    i++;
    tx_buf++;

    // === 3. Map skb frags (scattered data) ===
    for (frag = &skb_shinfo(skb)->frags[0]; ; frag++) {
        unsigned int frag_size = skb_frag_size(frag);

        dma = skb_frag_dma_map(tx_ring->dev, frag, 0,
                               frag_size, DMA_TO_DEVICE);
        if (dma_mapping_error(tx_ring->dev, dma))
            goto dma_error;

        tx_desc = ICE_TX_DESC(tx_ring, i);
        tx_desc->buf_addr = cpu_to_le64(dma);
        tx_desc->cmd_type_offset_bsz = ice_build_ctob(...);

        i++;
        // ... handle ring wrap
    }

    // === 4. 設置最後一個 descriptor 的 EOP (End of Packet) bit ===
    td_cmd |= ICE_TXD_LAST_DESC_CMD;
    tx_desc->cmd_type_offset_bsz = ice_build_ctob(td_cmd, ...);

    // === 5. Memory barrier - 確保 descriptors 寫入完成 ===
    wmb();

    // === 6. 更新 next_to_use ===
    tx_ring->next_to_use = i;

    // === 7. Ring doorbell - 通知硬件有新的 descriptors ===
    // 寫入 tail register
    writel(i, tx_ring->tail);

    return;

dma_error:
    // 錯誤處理: unmap 已經 map 的 buffers
    ice_unmap_and_free_tx_resource(tx_ring, first);
}
```

**Tx 完整流程總結**:
```
1. 網路協議棧呼叫 ice_start_xmit()
2. ice_xmit_frame_ring():
   - 檢查 ring 空間
   - 構建 context descriptor (如果需要)
3. ice_tx_map():
   - DMA map skb 數據 (得到 IOVA)
   - 寫入 descriptors (DMA 地址、長度、flags)
   - Memory barrier
   - 寫入 tail register (doorbell)
4. 硬件:
   - 讀取 descriptors
   - 從 DMA 地址讀取數據
   - 組裝封包並發送到網路
   - 設置 descriptor 的 DD bit (done)
5. Tx completion (ice_clean_tx_irq):
   - 檢查 DD bit
   - Unmap DMA buffers
   - 釋放 skb
```

---

## 階段 5: VF 銷毀

**文件**: `ice_sriov.c:131`

當用戶執行 `echo 0 > sriov_numvfs` 時，觸發 VF 銷毀流程。

```c
void ice_free_vfs(struct ice_pf *pf)
{
    struct ice_vfs *vfs = &pf->vfs;
    struct ice_vf *vf;
    unsigned int bkt;

    // 檢查是否有 VF 被分配給 VM
    if (pci_vfs_assigned(pf->pdev)) {
        dev_err(dev, "VFs are assigned to VMs - cannot free\n");
        return;
    }

    // === 1. 停止所有 VF 的 mailbox 工作 ===
    ice_eswitch_stop_all_tx_queues(pf);

    // === 2. 逐個釋放 VF ===
    mutex_lock(&vfs->table_lock);

    ice_for_each_vf(pf, bkt, vf) {
        // 觸發 VF reset
        ice_reset_vf(vf, ICE_VF_RESET_NOTIFY);

        // === 3. 釋放 VSI (Virtual Station Interface) ===
        if (vf->lan_vsi_idx != ICE_NO_VSI)
            ice_vf_vsi_release(vf);

        // === 4. 釋放虛擬 IRQ ===
        ice_virt_free_irqs(pf, vf);

        // === 5. 清除硬件寄存器配置 ===
        ice_dis_vf_mappings(vf);

        // === 6. 從 hash table 移除 ===
        hash_del_rcu(&vf->entry);

        // === 7. 釋放 VF 結構 ===
        ice_put_vf(vf);  // 減少引用計數，最終呼叫 kfree()
    }

    mutex_unlock(&vfs->table_lock);

    // === 8. 清空 hash table ===
    hash_init(vfs->table);

    // === 9. 重置 VF 統計 ===
    pf->vfs.num_qps_per = 0;
    pf->vfs.num_msix_per = 0;

    // === 10. 禁用 PCIe SR-IOV ===
    pci_disable_sriov(pf->pdev);
}
```

**銷毀流程關鍵步驟**:
1.  **檢查 VF 使用狀態**: 如果 VF 被分配給 VM (passthrough)，拒絕銷毀
2.  **停止流量**: 停止所有 Tx queues
3.  **Reset VF**: 通知 VF driver reset (如果還在運行)
4.  **釋放軟件資源**: VSI, IRQ, 內存結構
5.  **清除硬件配置**: 寄存器恢復到初始狀態
6.  **禁用 SR-IOV**: PCIe 層移除 VF 設備

---

## 完整生命週期代碼路徑總結

### 從 echo 到封包的完整路徑

```c
// ========== 階段 1: 創建 ==========
用戶: echo 4 > sriov_numvfs
  → ice_sriov_configure()                  [ice_sriov.c:1039]
    → ice_pci_sriov_ena()                  [ice_sriov.c:810]
      → pci_enable_sriov()                 【Kernel PCIe layer】
      → ice_set_per_vf_res()               [ice_sriov.c:371]
      → ice_create_vf_entries()            [ice_sriov.c:683]
      → ice_start_vfs()                    [ice_sriov.c:467]
        → ice_init_vf_vsi_res()            [ice_sriov.c:438]
        → ice_ena_vf_mappings()            [ice_sriov.c:323]
          → ice_ena_vf_msix_mappings()     [ice_sriov.c:230]
            → wr32(VPINT_ALLOC, ...)       【硬件寄存器】
            → wr32(GLINT_VECT2FUNC, ...)   【硬件寄存器】
          → ice_ena_vf_q_mappings()        [ice_sriov.c:276]
            → wr32(VPLAN_TX_QBASE, ...)    【硬件寄存器】
        → wr32(VFGEN_RSTAT, ACTIVE)        【VF 可見】

// ========== 階段 2: 協商 ==========
VF driver (iavf) probe
  → VF 發送: VIRTCHNL_OP_VERSION
    → PF: ice_vc_process_vf_msg()          [virt/virtchnl.c:2736]
      → ice_vc_get_ver_msg()               [virt/virtchnl.c:99]

  → VF 發送: VIRTCHNL_OP_GET_VF_RESOURCES
    → PF: ice_vc_get_vf_res_msg()          [virt/virtchnl.c:245]
      → vfres->num_queue_pairs = ...
      → vfres->max_vectors = ...
      → set_bit(ICE_VF_STATE_ACTIVE, ...)  【狀態轉換】

// ========== 階段 3: 配置 ==========
VF driver 分配 descriptor rings
  → VF 呼叫: dma_alloc_coherent()          【VF 分配內存】
  → VF 呼叫: dma_map_*()                   【得到 IOVA】

  → VF 發送: VIRTCHNL_OP_CONFIG_QUEUES
    → PF: ice_vc_cfg_qs_msg()              [virt/queues.c:749]
      → vsi->tx_rings[i]->dma = qpi->txq.dma_ring_addr  【記錄 IOVA】
      → ice_vsi_cfg_single_txq()           【配置硬件】
      → ice_vsi_cfg_single_rxq()           【配置硬件】

  → VF 發送: VIRTCHNL_OP_ENABLE_QUEUES
    → PF: ice_vc_ena_qs_msg()              [virt/queues.c:234]
      → ice_vsi_ctrl_one_rx_ring(..., true)  【啟用 Rx】
      → set_bit(ICE_VF_STATE_QS_ENA, ...)  【狀態轉換】

// ========== 階段 4: Runtime - Rx ==========
網路封包到達
  → 硬件 DMA: 封包 → rx_desc->pkt_addr   【使用 VF 的 IOVA】
  → 硬件設置: DD bit in descriptor
  → 硬件觸發: MSI-X 中斷 → VF CPU

VF driver:
  → ice_napi_poll()                        [ice_txrx.c:1702]
    → ice_clean_rx_irq()                   [ice_txrx.c:1381]
      → 檢查 DD bit
      → ice_build_skb()
      → ice_receive_skb()                  【送往協議棧】
    → ice_alloc_rx_bufs()                  [ice_txrx.c:883]
      → dma_map_single()                   【新 buffer】
      → rx_desc->pkt_addr = dma
      → ice_release_rx_desc()              【Ring doorbell】

// ========== 階段 4: Runtime - Tx ==========
協議棧要發送封包
  → ice_start_xmit()                       [ice_txrx.c:2701]
    → ice_xmit_frame_ring()                [ice_txrx.c:2586]
      → ice_tx_map()
        → dma_map_single(skb->data)        【Map 封包數據】
        → tx_desc->buf_addr = dma
        → tx_desc->cmd = ...|EOP
        → wmb()                            【Memory barrier】
        → writel(tail_reg)                 【Ring doorbell】

硬件:
  → 讀取 descriptors
  → DMA read: 從 buf_addr 讀取數據
  → 發送到網路
  → 設置 DD bit

VF driver (Tx completion):
  → ice_clean_tx_irq()
    → 檢查 DD bit
    → dma_unmap_single()
    → dev_kfree_skb_any()

// ========== 階段 5: 銷毀 ==========
用戶: echo 0 > sriov_numvfs
  → ice_free_vfs()                         [ice_sriov.c:131]
    → ice_reset_vf()
    → ice_vf_vsi_release()
    → ice_dis_vf_mappings()
    → hash_del_rcu()
    → pci_disable_sriov()                  【移除 VF 設備】
```

---

## Linus 式代碼審查

### ✅ 好品味的設計

1.  **狀態機清晰**:
    ```c
    INIT → ACTIVE → QS_ENA → RUNNING
    ```
    - 每個狀態轉換都有明確的觸發條件

2.  **Rx/Tx 的不對稱處理有道理**:
    ```c
    // Tx: Firmware 提前啟用
    // Rx: Driver 明確啟用
    ```
    - Tx 需要隨時準備發送，Rx 需要等 buffer ready

3.  **Memory barrier 使用正確**:
    ```c
    dma_rmb();  // Rx: 確保先讀 DD bit
    wmb();      // Tx: 確保 descriptor 寫入完成
    ```

### 潛在問題

*   **Switch-Case 結構**:
    *   **設計**: 使用巨大的 switch-case 處理 Mailbox 消息。
    *   **重點**: 雖然直觀，但隨著消息類型增加，函數會變得過長。建議使用函數指針陣列 (Function Pointer Table) 來分發處理函數，提升可維護性和查找效率。

2.  **錯誤處理路徑覆雜**:
    - goto error_param 跳來跳去
    - **建議**: 早期返回 (early return)

3.  **Tx 和 Rx 代碼重覆**:
    - 配置 Tx/Rx queue 的代碼幾乎一樣
    - **建議**: 提取共同邏輯

---

**文檔版本**: v2.0 - End-to-End Complete
**更新時間**: 2025-11-19
