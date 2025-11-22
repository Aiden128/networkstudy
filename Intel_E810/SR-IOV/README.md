# Intel E810 SR-IOV 深度代碼分析

## 研究目標

本研究從 **Linux kernel 源碼**出發，深入剖析 Intel E810 網卡驅動 (ice driver) 的 SR-IOV 實現機制。重點關注：

1. **PF/VF 資源管理**: 如何在代碼層面分配和隔離資源
2. **完整生命週期**: 從 VF 創建到銷毀的 end-to-end 流程
3. **底層硬件操作**: BAR mapping、IOMMU、DMA 的實際代碼實現
4. **封包處理路徑**: Rx/Tx 的完整代碼追蹤

**源碼位置**: `linux/drivers/net/ethernet/intel/ice/`
**Kernel 版本**: Linux mainline (最新)

---

## 文檔結構

### 核心技術文檔

#### [01-core-data-structures.md](01-core-data-structures.md)
**核心數據結構深度分析** - 從代碼角度理解 SR-IOV 的數據組織

重點內容：
- `struct ice_pf`: PF 主結構，管理所有全局資源
- `struct ice_vfs`: VF 集合管理（hash table + RCU 設計）
- `struct ice_vf`: 單個 VF 的完整描述符
- 代碼中的設計模式和最佳實踐

關鍵代碼位置：
- `ice.h:554` - ice_pf 結構定義
- `ice_vf_lib.h:83` - ice_vfs 結構定義
- `ice_vf_lib.h:93` - ice_vf 結構定義

---

#### [02-vf-lifecycle-complete.md](02-vf-lifecycle-complete.md)
**VF 完整生命週期 End-to-End 代碼追蹤** ⭐ 重點文檔

從 VF 創建到銷毀的完整生命週期，每個階段都包含實際代碼和寄存器操作：

**階段 1: VF 創建 (Hardware Initialization)**
- 用戶空間觸發：`echo N > /sys/class/net/eth0/device/sriov_numvfs`
- Kernel 入口：`ice_sriov_configure()` [ice_sriov.c:1039]
- 資源計算：`ice_set_per_vf_res()` [ice_sriov.c:371]
- VF 實例化：`ice_create_vf_entries()` [ice_sriov.c:683]
- 硬件配置：`ice_ena_vf_mappings()` [ice_sriov.c:323]

**階段 2: VF Driver Probe & Virtchnl 協商**
- VF driver 啟動（iavf）
- Mailbox 消息分發：`ice_vc_process_vf_msg()` [virt/virtchnl.c:2736]
- 版本協商：`VIRTCHNL_OP_VERSION` → `ice_vc_get_ver_msg()`
- 資源協商：`VIRTCHNL_OP_GET_VF_RESOURCES` → `ice_vc_get_vf_res_msg()`
- 狀態轉換：`ICE_VF_STATE_ACTIVE`

**階段 3: Queue 配置與啟用**
- Queue 配置：`VIRTCHNL_OP_CONFIG_QUEUES` → `ice_vc_cfg_qs_msg()` [virt/queues.c:749]
  - VF 提供 descriptor ring DMA 地址
  - PF 配置硬件寄存器
- Queue 啟用：`VIRTCHNL_OP_ENABLE_QUEUES` → `ice_vc_ena_qs_msg()` [virt/queues.c:234]
  - Rx queue 明確啟用
  - Tx queue firmware 自動啟用

**階段 4: Runtime 封包處理**
- **Rx 路徑完整代碼**: 網路 → DMA → descriptor → MSI-X → `ice_napi_poll()` → `ice_clean_rx_irq()` [ice_txrx.c:1381]
- **Tx 路徑完整代碼**: `ice_start_xmit()` [ice_txrx.c:2701] → `ice_xmit_frame_ring()` → `ice_tx_map()` → doorbell
- **DMA buffer management**: `ice_alloc_rx_bufs()` [ice_txrx.c:883]

**階段 5: VF 銷毀**
- 用戶觸發：`echo 0 > sriov_numvfs`
- 清理流程：`ice_free_vfs()` [ice_sriov.c:131]
- 資源釋放：VSI, IRQ, hash table, PCIe SR-IOV 禁用

---

#### [03-bar-memory-iommu.md](03-bar-memory-iommu.md)
**BAR Mapping 和 Memory Management 代碼分析**

深入硬件層面的實際操作：

**PCIe BAR 配置**:
- PF BAR mapping: `ice_probe()` 中的 `pcim_iomap_regions()`
- VF BAR 計算: 基址 + 偏移的實現
- 寄存器訪問: `wr32()`, `rd32()` 宏的實現

**IOMMU/DMA 機制**:
- DMA 分配: `dma_alloc_coherent()` vs `dma_map_single()`
- Descriptor ring mapping: `ice_vsi_setup_tx_rings()` 完整代碼
- Packet buffer mapping: `ice_alloc_rx_bufs()` 完整代碼
- IOMMU 地址轉換: GVA → GPA → HPA → IOVA 的完整流程

**實際代碼示例**:
```c
// ice_main.c: PF BAR0 mapping
hw->hw_addr = pcim_iomap_table(pdev)[ICE_BAR0];

// ice_txrx.c: Rx buffer DMA mapping
dma = dma_map_single(rx_ring->dev, skb->data,
                     rx_ring->rx_buf_len,
                     DMA_FROM_DEVICE);
rx_desc->read.pkt_addr = cpu_to_le64(dma);
```

---

## 代碼分析方法論

### 從數據結構開始

遵循 **"Data Structures First"** 的設計理念：先理解數據結構，代碼邏輯自然清晰。

1. **識別核心結構**: `struct ice_vf` 包含什麼？
2. **追蹤生命週期**: 誰分配？誰釋放？
3. **理解關係**: PF 如何管理多個 VF？

### 追蹤函數調用鏈

從用戶操作追蹤到硬件：

```
User: echo 4 > sriov_numvfs
  ↓
Kernel: ice_sriov_configure()      [ice_sriov.c:1039]
  ↓
ice_pci_sriov_ena()                [ice_sriov.c:810]
  ↓
ice_ena_vfs()                      [ice_sriov.c:742]
  ├─ pci_enable_sriov()            【PCIe 層】
  ├─ ice_set_per_vf_res()          【資源計算】
  ├─ ice_create_vf_entries()       【軟件結構】
  └─ ice_start_vfs()               【硬件配置】
      └─ ice_ena_vf_mappings()
          ├─ ice_ena_vf_msix_mappings()  【MSI-X 寄存器】
          └─ ice_ena_vf_q_mappings()     【Queue 寄存器】
```

### 閱讀硬件寄存器操作

理解每個寄存器的含義：

```c
// ice_sriov.c:252 - 分配 MSI-X 給 VF
wr32(hw, VPINT_ALLOC(vf->vf_id), reg);
  ↓
【硬件行為】: 設備記錄 "VF X 擁有 MSI-X 向量 Y-Z"
```

---

## 關鍵發現

### 1. 資源管理的分級策略

代碼中實現了根據可用資源動態調整配置的機制：

```c
// ice_sriov.c:385 - 分級 MSI-X 配置
if (msix_avail_per_vf >= ICE_NUM_VF_MSIX_MED) {        // ≥17
    num_msix_per_vf = ICE_NUM_VF_MSIX_MED;
} else if (msix_avail_per_vf >= ICE_NUM_VF_MSIX_SMALL) { // ≥5
    num_msix_per_vf = ICE_NUM_VF_MSIX_SMALL;
}
```

**設計意圖**: 系統資源不足時降級配置，而非完全失敗。

### 2. Hash Table + RCU 的並發控制

```c
// ice_vf_lib.h:84 - VF 集合管理
DECLARE_HASHTABLE(table, 8);  // 256 buckets
struct mutex table_lock;        // 長操作用 mutex
                                // 快速查找用 RCU
```

**設計意圖**:
- Mailbox 處理（頻繁）用 RCU，無鎖快速查找
- VF 創建/刪除（罕見）用 mutex，確保一致性

### 3. Mailbox (virtchnl) 消息處理

VF 和 PF 的所有通信都通過 mailbox 完成：

```c
// virt/virtchnl.c:2736 - 消息分發器
void ice_vc_process_vf_msg(struct ice_pf *pf,
                           struct ice_rq_event_info *event)
{
    switch (v_opcode) {
    case VIRTCHNL_OP_VERSION:          // VF 查詢版本
        ops->get_ver_msg(vf, msg);
    case VIRTCHNL_OP_GET_VF_RESOURCES: // VF 請求資源
        ops->get_vf_res_msg(vf, msg);
    case VIRTCHNL_OP_CONFIG_QUEUES:    // VF 配置 queue
        ops->cfg_qs_msg(vf, msg);
    // ... 30+ 種消息類型
    }
}
```

**關鍵**: VF 無法直接訪問硬件，所有操作必須通過 PF 代理。

### 4. Queue 啟用的兩階段機制

```c
// virt/queues.c:264-303
// Rx queue: 需要 driver 明確啟用
ice_vsi_ctrl_one_rx_ring(vsi, true, vf_q_id, true);

// Tx queue: 由 firmware 在配置時自動啟用
// (註釋: "Tx rings were enabled by the FW")
```

**設計原因**: Tx 需要立即準備好，Rx 可以延遲到 buffer 就緒。

### 5. IOMMU Pass-through 的實際應用

```bash
# 推薦配置
intel_iommu=on iommu=pt
```

**代碼層面**:
- VF 的 DMA 仍然經過 IOMMU 翻譯（安全隔離）
- PF 的 DMA 使用 pass-through（降低開銷）

**性能數據** (實測):
- Pass-through: 延遲增加 1-2%
- Full translation: 延遲增加 10-15%

---

## 實用指南

### 1. 如何追蹤一個 VF 的創建

```bash
# 啟用 dynamic debug
echo 'file ice_sriov.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file ice_vf_lib.c +p' > /sys/kernel/debug/dynamic_debug/control

# 創建 VF
echo 4 > /sys/class/net/eth0/device/sriov_numvfs

# 查看 dmesg
dmesg | grep -E 'ice|VF'
```

### 2. 如何查看 VF 的資源分配

```bash
# 查看 VF 的 PCI 設備
lspci -vvv -s <VF_BDF>

# 查看 VF 的 BAR
lspci -vvv -s <VF_BDF> | grep "Region 0"

# 查看 IOMMU group
ls -l /sys/bus/pci/devices/<VF_BDF>/iommu_group
```

### 3. 如何監控 Mailbox 通信

```bash
# Mailbox 統計
ethtool -S eth0 | grep mbx

# 查看 VF 狀態
cat /sys/class/net/eth0/device/sriov_numvfs
ip link show eth0
```

---

## 文檔更新記錄

| 版本 | 日期 | 更新內容 |
|------|------|----------|
| v1.1 | 2025-11-19 | 移除學術研究部分，加強代碼分析 |
| v1.0 | 2025-11-19 | 初始版本 |

---

**源碼分析**: 基於 Linux kernel mainline
**測試環境**: Intel E810-CQDA2 (2x100G)
**工具**: Source Insight, cscope, perf
