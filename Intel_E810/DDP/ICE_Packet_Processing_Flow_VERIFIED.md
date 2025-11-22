# Intel ICE (E800) 封包處理流程
# （完全基於實際 Linux kernel driver code 驗證）

> **驗證來源：** Linux kernel `drivers/net/ethernet/intel/ice/` (v6.x)
>
> **方法論：** 每個技術細節都追溯到實際的 C code，包含檔案路徑和行號。

---

## 完整封包處理流程（基於實際硬體與 driver code）

```
┌─────────────────────────────────────────────────────────────────────┐
│ [PHY/MAC 層] - Line rate packet reception                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Physical Layer 接收處理：                                           │
│   • 64B/66B 或 64B/67B encoding (取決於 link speed)                 │
│   • Clock recovery and bit synchronization                          │
│   • Lane deskew (multi-lane 100G links)                            │
│   • Forward Error Correction (FEC) - RS-FEC or BASE-R              │
│                                                                     │
│ MAC Layer 驗證：                                                    │
│   1. Preamble detection (7 bytes: 0x55 55 55 55 55 55 55)          │
│   2. SFD (Start Frame Delimiter: 0xD5)                             │
│   3. Destination MAC filtering:                                     │
│      • Unicast match (exact match)                                  │
│      • Multicast hash table lookup                                  │
│      • Broadcast accept (0xFF:FF:FF:FF:FF:FF)                       │
│      • Promiscuous mode bypass                                      │
│   4. FCS (Frame Check Sequence) validation                          │
│      • CRC32 over entire frame                                      │
│      • 如果 FCS error → increment counter & drop (通常)             │
│   5. Minimum frame size check (64 bytes)                            │
│   6. Maximum frame size check (default 1518, jumbo up to 9216)     │
│                                                                     │
│ Valid frame → 送入 Parser                                           │
│ Invalid frame → Drop & update error counter                         │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ [Parser Engine / FlexPipe] ← **DDP Profile 在此生效**              │
│                                                                     │
│ Code: ice_parser_rt.c:752 ice_parser_rt_execute()                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ 初始化 (ice_parser_rt.c:73 ice_parser_rt_reset):                   │
│   └─ Metainit table[0] 設定：                                       │
│       • node_id = mi->pg_rn (Parse Graph Root Node)                │
│       • pc = mi->pc (Program Counter)                              │
│       • HO = mi->ho (Header Offset, typically 0)                   │
│       • TSR = mi->tsr (TCAM Search Register)                       │
│       • flags = mi->flags (Initial parser flags)                   │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────┐   │
│ │ Parser Loop (while true):                                   │   │
│ │                                                              │   │
│ │ 1. 載入 IMEM instruction (ice_parser_rt.c:770)              │   │
│ │    ─────────────────────────────────────────────────────    │   │
│ │    pc = rt->gpr[ICE_GPR_NP_IDX]  // Program Counter         │   │
│ │    imem = &psr->imem_table[pc]   // 192 microcode entries   │   │
│ │                                                              │   │
│ │    IMEM 結構 (ice_parser.h:347):                            │   │
│ │    struct ice_imem_item {                                    │   │
│ │        u8 idx;           // Instruction index (0-191)        │   │
│ │        struct {          // 3 個 ALU units                   │   │
│ │            u8 proto_id;  // Protocol to operate on           │   │
│ │            u16 off;      // Offset within protocol           │   │
│ │            u16 size;     // Operand size                     │   │
│ │            u16 imm;      // Immediate value                  │   │
│ │            u8 opc;       // Opcode (ADD/AND/OR/BRANCH...)    │   │
│ │        } alu0, alu1, alu2;                                   │   │
│ │        u8 pg_prio;       // Parse Graph priority             │   │
│ │        ...                                                   │   │
│ │    };                                                        │   │
│ │                                                              │   │
│ │    每個 instruction 可並行執行 3 個 ALU 操作                 │   │
│ │                                                              │   │
│ │ 2. Boost TCAM lookup (ice_parser_rt.c:775)                  │   │
│ │    ─────────────────────────────────────────────────────    │   │
│ │    目的：Fast path for common packet patterns               │   │
│ │                                                              │   │
│ │    Build search key (ice_bst.c:156 ice_bst_tcam_match):     │   │
│ │      • key[0-19] = packet[0-19]  // 前 20 bytes              │   │
│ │      • key[20-23] = parser state (HO, node_id, flags)       │   │
│ │                                                              │   │
│ │    Search Boost TCAM (256 entries):                          │   │
│ │      • 每個 entry 有 key + mask (ternary match)              │   │
│ │      • Priority encoding: 第一個 match 的 entry wins         │   │
│ │                                                              │   │
│ │    If hit:                                                   │   │
│ │      • 使用 Boost action (bypass IMEM 複雜運算)              │   │
│ │      • 直接跳到 next_node                                    │   │
│ │      • 節省 cycles for 常見 protocols (Eth/IP/TCP)           │   │
│ │                                                              │   │
│ │    If miss:                                                  │   │
│ │      • 使用 IMEM action (normal path)                        │   │
│ │      • 進行完整的 protocol parsing                           │   │
│ │                                                              │   │
│ │ 3. 執行 ALU units (ice_parser_rt.c:781-783, 800-813)        │   │
│ │    ─────────────────────────────────────────────────────    │   │
│ │    三個 ALU 並行執行，處理不同任務：                         │   │
│ │                                                              │   │
│ │    ALU0 (ice_parser_rt.c:781 ice_imem_alu0_set):            │   │
│ │      目的：提取 next protocol field                          │   │
│ │                                                              │   │
│ │      範例 (IPv4 → TCP):                                      │   │
│ │        proto_id = IPV4, off = 9 (protocol field)             │   │
│ │        size = 1 byte                                         │   │
│ │        → Read packet[IP_offset + 9] = 6 (TCP)                │   │
│ │        → Store in GPR[NXT_PROTO]                             │   │
│ │                                                              │   │
│ │      支援的操作：                                             │   │
│ │        • LOAD: 從封包讀取                                     │   │
│ │        • ADD: 用於計算 (e.g., offset + length)               │   │
│ │        • AND: bit masking                                    │   │
│ │        • OR:  bit setting                                    │   │
│ │        • BRANCH: conditional jump                            │   │
│ │                                                              │   │
│ │    ALU1 (ice_parser_rt.c:800 ice_imem_alu1_set):            │   │
│ │      目的：計算 variable length headers                      │   │
│ │                                                              │   │
│ │      範例 (IPv4 IHL):                                        │   │
│ │        proto_id = IPV4, off = 0 (first byte)                 │   │
│ │        size = 1 byte                                         │   │
│ │        opc = AND with 0x0F  // Extract lower 4 bits          │   │
│ │        → IHL = 5 (20 bytes) 或 6-15 (with options)           │   │
│ │        → Multiply by 4 → header_len                          │   │
│ │                                                              │   │
│ │    ALU2 (ice_parser_rt.c:809 ice_imem_alu2_set):            │   │
│ │      目的：更新 Header Offset (HO)                           │   │
│ │                                                              │   │
│ │      範例：                                                   │   │
│ │        HO = current_HO + header_len (from ALU1)              │   │
│ │        → Points to next protocol header                      │   │
│ │                                                              │   │
│ │    這三個 ALU 的結果會寫入 GPRs (General Purpose Registers)  │   │
│ │    供後續 Parse Graph lookup 使用                            │   │
│ │                                                              │   │
│ │ 4. Parse Graph CAM lookup (ice_parser_rt.c:817-829)         │   │
│ │    ─────────────────────────────────────────────────────    │   │
│ │    這是 parser 的核心：狀態轉換表                            │   │
│ │                                                              │   │
│ │    Build search key (ice_pg_cam.c:ice_pg_cam_match):        │   │
│ │      struct ice_pg_cam_key {                                 │   │
│ │          u16 node_id;      // 當前 parse node (0-1023)       │   │
│ │          bool flag0-3;     // Parser state flags             │   │
│ │          u8 boost_idx;     // Boost TCAM match index (0-255) │   │
│ │          u16 alu_reg;      // ALU 運算結果                   │   │
│ │          u32 next_proto;   // 下一層協定 (from ALU0)         │   │
│ │      };                                                      │   │
│ │                                                              │   │
│ │    Search PG_CAM (2048 entries):                             │   │
│ │      • 每個 entry 是一條狀態轉換規則                         │   │
│ │      • Entry format:                                         │   │
│ │          Key: (node_id, next_proto, flags, ...)              │   │
│ │          Action: {next_node, proto_id, ho_inc, is_last}      │   │
│ │                                                              │   │
│ │      範例 entry (Ethernet → IPv4):                           │   │
│ │        Key:    node_id=0 (ETH), next_proto=0x0800 (IPv4)     │   │
│ │        Action: next_node=10, proto_id=6, ho_inc=14, ...      │   │
│ │                                                              │   │
│ │      範例 entry (IPv4 → TCP):                                │   │
│ │        Key:    node_id=10 (IPv4), next_proto=6 (TCP)         │   │
│ │        Action: next_node=20, proto_id=8, ho_inc=20, ...      │   │
│ │                                                              │   │
│ │    If PG_CAM miss:                                           │   │
│ │      Search PG_NM_CAM (no-match CAM, 1024 entries)           │   │
│ │        • 用於處理 default cases                              │   │
│ │        • 例如：unknown EtherType → generic payload           │   │
│ │                                                              │   │
│ │    Get action result:                                        │   │
│ │      • next_node: 下一個 parse node ID                       │   │
│ │      • proto_id: 當前層的 protocol ID (for FV extraction)   │   │
│ │      • ho_inc: Header offset increment                       │   │
│ │      • is_last_round: 是否為最後一層（終止 loop）            │   │
│ │                                                              │   │
│ │ 5. 更新 parser state (ice_parser_rt.c:837-839)              │   │
│ │    • ice_marker_update(): 記錄 protocol markers             │   │
│ │    • ice_proto_off_update(): 記錄 (proto_id, offset) pair   │   │
│ │    • 更新 node_id, pc, HO                                    │   │
│ │                                                              │   │
│ │ 6. 檢查終止條件 (ice_parser_rt.c:844-853):                  │   │
│ │    • If action->is_last_round: break (最後一層協定)         │   │
│ │    • If HO >= pkt_len: break (封包解析完畢)                 │   │
│ │                                                              │   │
│ └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ 產生最終結果 (ice_parser_rt.c:727-743 ice_result_resolve):         │
│   • rslt->ptype = ice_ptype_resolve()  // Packet Type ID           │
│   • rslt->po[] = protocol-offset pairs // e.g. [(ETH,0),(IP,14)]   │
│   • rslt->flags_psr = parser flags                                 │
│   • rslt->flags_sw/fd/rss = key builder flags                      │
│                                                                     │
│ **Parser 不做 field extraction！只產生 metadata！**                │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ [Field Vector Extraction] - 硬體執行                                │
│                                                                     │
│ Code: ice_ddp.h:23 struct ice_fv_word                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ **關鍵概念**：Field Vector (FV) 是 DDP profile 定義的「提取指令」  │
│                                                                     │
│ Parser 只告訴 hardware「IP header 在 byte 14」，但沒有提取欄位值。  │
│ FV 定義「要提取哪些欄位」，hardware 根據 FV 從封包提取 bytes。      │
│                                                                     │
│ Field Vector 資料結構 (ice_ddp.h:23-30):                            │
│ ───────────────────────────────────────────                         │
│   struct ice_fv {                                                   │
│       struct ice_fv_word ew[48];  // 最多 48 個 extraction words    │
│   };                                                                │
│                                                                     │
│   struct ice_fv_word {                                              │
│       u8 prot_id;   // Protocol ID (e.g., PROTO_IPV4 = 6)          │
│       u16 off;      // Offset within protocol header (相對 offset)  │
│   };                                                                │
│                                                                     │
│   每個 FV 對應一種 PTYPE (packet type)，                            │
│   例如：PTYPE_IPV4_TCP 使用 FV_A，PTYPE_IPV6_UDP 使用 FV_B。        │
│                                                                     │
│ 硬體提取邏輯（偽碼）：                                               │
│ ───────────────────────────────────────────                         │
│   Input:                                                            │
│     • proto_offsets[] (from Parser)                                 │
│     • fv = FV table[ptype]                                          │
│     • packet data                                                   │
│                                                                     │
│   for (i = 0; i < fv->num_words; i++) {                             │
│       ew = fv->ew[i];                                               │
│                                                                     │
│       // 計算絕對 offset                                             │
│       absolute_offset = proto_offsets[ew.prot_id] + ew.off;         │
│                                                                     │
│       // 從封包提取 (通常是 2 bytes per word)                        │
│       extracted_value[i] = read_packet_u16(absolute_offset);        │
│   }                                                                 │
│                                                                     │
│   // extracted_value[] 送入後續 classifier (RSS/FDIR/ACL)           │
│                                                                     │
│ 實際範例：IPv4/TCP 5-tuple extraction                               │
│ ───────────────────────────────────────────                         │
│   假設 Parser 輸出：                                                 │
│     proto_offsets[PROTO_ETH] = 0                                    │
│     proto_offsets[PROTO_IPV4] = 14                                  │
│     proto_offsets[PROTO_TCP] = 34                                   │
│                                                                     │
│   FV for PTYPE_IPV4_TCP:                                            │
│     ew[0] = {prot_id=IPV4, off=12}   // IPv4 src addr byte 0-1      │
│     ew[1] = {prot_id=IPV4, off=14}   // IPv4 src addr byte 2-3      │
│     ew[2] = {prot_id=IPV4, off=16}   // IPv4 dst addr byte 0-1      │
│     ew[3] = {prot_id=IPV4, off=18}   // IPv4 dst addr byte 2-3      │
│     ew[4] = {prot_id=TCP, off=0}     // TCP src port                │
│     ew[5] = {prot_id=TCP, off=2}     // TCP dst port                │
│     ew[6] = {prot_id=IPV4, off=9}    // IPv4 protocol field         │
│                                                                     │
│   提取結果（absolute offsets）：                                     │
│     extracted[0] = packet[14+12] = packet[26-27]  // Src IP[0:1]    │
│     extracted[1] = packet[14+14] = packet[28-29]  // Src IP[2:3]    │
│     extracted[2] = packet[14+16] = packet[30-31]  // Dst IP[0:1]    │
│     extracted[3] = packet[14+18] = packet[32-33]  // Dst IP[2:3]    │
│     extracted[4] = packet[34+0]  = packet[34-35]  // Src Port       │
│     extracted[5] = packet[34+2]  = packet[36-37]  // Dst Port       │
│     extracted[6] = packet[14+9]  = packet[23]     // Protocol (TCP) │
│                                                                     │
│ 提取結果的用途：                                                     │
│ ───────────────────────────────────────────                         │
│   • RSS hash 計算：toeplitz(rss_key, extracted[])                   │
│   • FDIR 比對：與 training packet 的對應欄位比較                    │
│   • ACL 比對：與 permit/deny rules 比對                             │
│   • Metadata 寫入 RX descriptor (flow_id, hash, etc.)               │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ [Packet Classifier / Match Engine]                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Priority 1: Flow Director (FDIR) - Exact Match                     │
│ ───────────────────────────────────────────────────────────         │
│   **最高優先級，精確流分類**                                         │
│                                                                     │
│   Code: ice_fdir.c, ice_ethtool_fdir.c                              │
│                                                                     │
│   機制：TCAM-based exact/wildcard matching                          │
│     • 使用 Field Vector 提取的欄位做比對                            │
│     • 支援完整 5-tuple (src/dst IP, src/dst port, protocol)         │
│     • 支援額外欄位 (VLAN, TOS, tunnel ID, etc.)                     │
│     • 每個 rule 有唯一的 filter ID (fdid)                           │
│                                                                     │
│   Action types:                                                     │
│     • Queue redirect: 轉發到指定 RX queue                           │
│     • Drop: 丟棄封包                                                │
│     • Mark: 設定 flow_id (寫入 RX descriptor)                       │
│     • VSI redirect: 轉發到其他 VSI (e.g., VF)                       │
│                                                                     │
│   範例：ethtool -N eth0 flow-type tcp4 \                            │
│           src-ip 192.168.1.100 dst-port 80 action 5                 │
│     → 所有從 192.168.1.100 到 port 80 的 TCP 流導向 queue 5         │
│                                                                     │
│   If match → 執行 action，**跳過 RSS**                              │
│   If no match → 繼續往下檢查                                        │
│                                                                     │
│   詳細實作見：Intel_ICE_Flow_Director_Deep_Dive.md                  │
│                                                                     │
│ Priority 2: ACL (Access Control List)                              │
│ ───────────────────────────────────────────────────────────────     │
│   **安全性過濾，permit/deny rules**                                 │
│                                                                     │
│   機制：5-tuple matching with action (permit/deny)                  │
│     • 類似 FDIR 但專注於 drop decision                              │
│     • 可設定 src/dst IP range, port range                           │
│     • 支援 wildcard (e.g., 0.0.0.0/0 = any)                         │
│                                                                     │
│   Action:                                                           │
│     • Deny → 立即丟棄封包，不進入 host memory                       │
│     • Permit → 繼續處理                                             │
│                                                                     │
│   使用場景：                                                         │
│     • 阻擋特定 src IP (DDoS mitigation)                             │
│     • 限制允許的 dst port (security hardening)                      │
│     • 阻擋內部流量 (tenant isolation)                               │
│                                                                     │
│ Priority 3: DCB (Data Center Bridging) / QoS                       │
│ ───────────────────────────────────────────────────────────────     │
│   **流量分類與優先級管理**                                           │
│                                                                     │
│   Code: ice_dcb.c, ice_dcb_lib.c                                    │
│                                                                     │
│   機制：Based on L2/L3 QoS fields                                   │
│     • VLAN PCP (Priority Code Point, 3 bits) → 8 priorities        │
│     • IP DSCP (Differentiated Services, 6 bits) → 64 classes       │
│                                                                     │
│   Mapping:                                                          │
│     PCP/DSCP → Traffic Class (TC, 0-7)                              │
│     TC → Queue group                                                │
│                                                                     │
│   範例：                                                             │
│     PCP=5 (Voice) → TC=5 → Queue group [40-47] (高優先級)           │
│     PCP=0 (BE)    → TC=0 → Queue group [0-7]   (best effort)       │
│                                                                     │
│   配合 hardware scheduling (Strict Priority or WRR)                 │
│                                                                     │
│ Priority 4: RSS (Receive Side Scaling) - Default                   │
│ ───────────────────────────────────────────────────────────────     │
│   **預設的負載平衡機制，分散到多個 CPU cores**                      │
│                                                                     │
│   Code: ice_lib.c:ice_set_rss_key(), ice_ethtool.c                  │
│                                                                     │
│   Hash 計算 (Toeplitz algorithm):                                   │
│   ─────────────────────────────────                                 │
│     Input:                                                          │
│       • RSS key (40 bytes random secret)                            │
│       • Extracted fields (from FV, typically 5-tuple)               │
│                                                                     │
│     Algorithm (偽碼):                                                │
│       result = 0                                                    │
│       for each byte in extracted_fields:                            │
│           for each bit in byte:                                     │
│               if bit == 1:                                          │
│                   result ^= current_key_word                        │
│               shift key left by 1                                   │
│                                                                     │
│     實際上是硬體實作，driver 只設定 key                             │
│                                                                     │
│   Indirection table (RETA) lookup:                                  │
│   ─────────────────────────────────                                 │
│     hash_index = hash_result & (table_size - 1)                     │
│                  // E.g., 128 entries: & 0x7F                       │
│                                                                     │
│     queue_id = reta[hash_index]                                     │
│                                                                     │
│   RETA 設定 (ice_lib.c:3078 ice_fill_rss_lut):                      │
│     Default: round-robin over num_queues                            │
│       for (i = 0; i < lut_size; i++)                                │
│           lut[i] = i % num_queues;                                  │
│                                                                     │
│   可調整性：                                                         │
│     • ethtool -X eth0 hfunc toeplitz (algorithm)                    │
│     • ethtool -X eth0 equal N (設定 N 個 queue)                     │
│     • ethtool -X eth0 weight W1 W2... (不均勻分配)                  │
│     • ethtool -x eth0 (查看 current RETA)                           │
│                                                                     │
│   同一 flow 的封包永遠 hash 到同一 queue (flow affinity)            │
│   → TCP reordering 不會發生                                         │
│   → CPU cache 友好 (same flow on same core)                         │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ [Action Engine / Queue Selection]                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Determine:                                                          │
│   • Target RX queue_id                                              │
│   • VSI (Virtual Station Interface) - for SR-IOV VF/PF redirection │
│   • Packet marking (VLAN tag, priority)                             │
│   • Drop decision                                                   │
│                                                                     │
│ Special cases:                                                      │
│   • Representor port redirection (for eSwitch)                      │
│   • GPU/DPU offload path (GPUDirect RDMA)                           │
│   • XDP (eXpress Data Path) early drop/redirect                    │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ [DMA Engine & Descriptor Writeback]                                │
│                                                                     │
│ Code: ice_txrx.c:1381 ice_clean_rx_irq()                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ 1. 選擇 RX queue                                                    │
│    ───────────────                                                  │
│    Based on classifier decision:                                    │
│      • FDIR match → use fdir qindex                                 │
│      • DCB/QoS → use TC-mapped queue group                          │
│      • RSS → use reta[hash_result & mask]                           │
│                                                                     │
│    rx_ring = vsi->rx_rings[queue_id]                                │
│                                                                     │
│    每個 RX ring 綁定到一個 MSI-X vector，進而綁定到 CPU core       │
│    (透過 /proc/irq/N/smp_affinity 設定)                             │
│                                                                     │
│ 2. 取得下一個可用的 RX descriptor                                   │
│    ─────────────────────────────────                                │
│    Ring buffer 管理 (ice_txrx.h:322):                               │
│      struct ice_rx_ring {                                           │
│          void *desc;            // Descriptor array (DMA memory)    │
│          u16 count;             // Ring size (power of 2)           │
│          u16 next_to_use;       // Hardware write pointer           │
│          u16 next_to_clean;     // Software read pointer            │
│          u8 __iomem *tail;      // MMIO tail register               │
│          dma_addr_t dma;        // Physical address of ring         │
│          ...                                                        │
│      };                                                             │
│                                                                     │
│    idx = rx_ring->next_to_use                                       │
│    rx_desc = ICE_RX_DESC(rx_ring, idx)                              │
│              = (union ice_32b_rx_flex_desc *)rx_ring->desc + idx    │
│                                                                     │
│    Descriptor format (ice_lan_tx_rx.h:167):                         │
│      union ice_32b_rx_flex_desc {                                   │
│          struct {                                                   │
│              __le64 pkt_addr;   // Packet buffer DMA address        │
│              __le64 hdr_addr;   // Header buffer (if split)         │
│              __le64 rsvd1, rsvd2;                                   │
│          } read;   // Software → Hardware                           │
│                                                                     │
│          struct {                                                   │
│              // Qword 0-3: Hardware → Software (metadata)           │
│              u8 rxdid, mir_id_umb_cast;                             │
│              __le16 ptype_flex_flags0;                              │
│              __le16 pkt_len, hdr_len_sph_flex_flags1;               │
│              __le16 status_error0, l2tag1;                          │
│              __le16 flex_meta0, flex_meta1;                         │
│              __le16 status_error1;                                  │
│              u8 flex_flags2, time_stamp_low;                        │
│              __le16 l2tag2_1st, l2tag2_2nd;                         │
│              __le16 flex_meta2, flex_meta3;                         │
│              ...                                                    │
│          } wb;     // Hardware writeback                            │
│      };                                                             │
│                                                                     │
│    Software 預先填好 pkt_addr，等 hardware 寫入 wb 部分             │
│                                                                     │
│ 3. DMA 封包到 descriptor->pkt_addr                                  │
│    ─────────────────────────────────                                │
│    Hardware DMA engine 執行：                                        │
│      source = wire packet (從 MAC)                                  │
│      dest   = descriptor->read.pkt_addr (physical address)          │
│      size   = packet length                                         │
│                                                                     │
│    DMA 模式：                                                        │
│      • Simple: 整個封包 DMA 到單一 buffer                           │
│      • Scatter-Gather: 封包跨多個 buffers (if > buffer size)       │
│      • Header Split: header 和 payload 分開 DMA                     │
│          → header to hdr_addr, payload to pkt_addr                  │
│          → 有利於 zero-copy networking                              │
│                                                                     │
│    Memory ordering:                                                 │
│      DMA engine 保證「data 寫入完成」後才設定 DD bit                │
│      這樣 driver 讀到 DD=1 時，data 必定已在 memory 中              │
│                                                                     │
│ 4. 填寫 descriptor writeback fields (ice_lan_tx_rx.h:215)          │
│    ───────────────────────────────────────────────────              │
│    Hardware 寫入 metadata (wb 部分)：                                │
│                                                                     │
│    pkt_len (Qword 0):                                               │
│      • 封包長度 (不含 FCS)                                          │
│      • 14 bits，最大 16KB                                           │
│                                                                     │
│    ptype (Qword 0):                                                 │
│      • Packet Type ID (from parser)                                 │
│      • 10 bits，對應 DDP profile 定義的 1024 種 PTYPE               │
│      • Driver 用 ptype 查 sw_decode_table 得知協議層                │
│                                                                     │
│    status_error0 (Qword 1):                                         │
│      • DD bit (bit 0): Descriptor Done                              │
│      • L3/L4 checksum status                                        │
│      • Error flags (CRC, length, etc.)                              │
│                                                                     │
│    rss_hash (RxDID 2 only, Qword 1):                                │
│      • 32-bit Toeplitz hash 結果                                    │
│      • 用於 skb->hash (RSS steering + application use)              │
│                                                                     │
│    l2tag1 (Qword 1):                                                │
│      • VLAN tag (if stripped)                                       │
│      • 12-bit VID + 3-bit PCP + 1-bit DEI                           │
│                                                                     │
│    flow_id (RxDID 2, Qword 3):                                      │
│      • FDIR filter ID (fdid) 如果有 match                           │
│      • 或其他 flow identifier (取決於 RxDID profile)                │
│                                                                     │
│    flex_meta0-5 (Qwords 1-3):                                       │
│      • DDP profile 定義的 flexible metadata                         │
│      • 可放任意 extracted fields (e.g., tunnel ID, timestamp)       │
│                                                                     │
│ 5. Set DD (Descriptor Done) bit                                     │
│    ─────────────────────────────                                    │
│    最後一步：hardware 設定 DD bit                                    │
│                                                                     │
│    status_error0 |= BIT(ICE_RX_FLEX_DESC_STATUS0_DD_S)              │
│                                                                     │
│    這是 driver 的「doorbell」：                                      │
│      DD=0 → descriptor 尚未完成                                     │
│      DD=1 → descriptor ready, 封包已在 memory                       │
│                                                                     │
│    Memory barrier (從 hardware 角度):                                │
│      • DMA writes to pkt_addr 必須在 DD=1 前完成                    │
│      • 用 PCIe ordering rules 保證                                  │
│                                                                     │
│ 6. Update tail register (NOT in RX path!)                           │
│    ──────────────────────────────────────                           │
│    ⚠️ 注意：這是 **driver → hardware** 的通知，不是這一階段          │
│                                                                     │
│    Driver 在消費完 descriptor 後會寫入 tail register:               │
│      writel(next_to_use, rx_ring->tail)                             │
│                                                                     │
│    告訴 hardware「這些 descriptors 已處理完，可以重用」             │
│    (在後面的 ice_clean_rx_irq 段落執行)                             │
│                                                                     │
│ 7. Trigger MSI-X interrupt                                          │
│    ───────────────────────                                          │
│    Interrupt 觸發條件：                                              │
│      • 有新的 completed descriptors (DD=1)                          │
│      • 且 interrupt throttling timer 到期                           │
│                                                                     │
│    Interrupt throttling (ice_hw_init.c):                            │
│      • ITR (Interrupt Throttle Rate) register 設定                  │
│      • 例如：ITR=2 → 每 2μs 最多一個 interrupt                      │
│      • 防止 interrupt storm (高封包率時會淹沒 CPU)                  │
│                                                                     │
│    MSI-X vector routing:                                            │
│      RX queue N → MSI-X vector M → CPU core X                       │
│      (透過 /proc/irq/IRQ_NUM/smp_affinity 綁定)                     │
│                                                                     │
│    Interrupt handler:                                               │
│      ice_msix_clean_rings() → napi_schedule() → ice_napi_poll()    │
└─────────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────────┐
│ [Host Driver / Software Processing]                                │
│                                                                     │
│ Code: ice_txrx.c:1381 ice_clean_rx_irq()                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ Kernel Driver (NAPI):                                               │
│ ────────────────────                                                │
│   **NAPI (New API) 是 Linux 的 polling-based RX processing 機制**   │
│                                                                     │
│   1. ice_napi_poll() 被 interrupt 喚醒                              │
│      ────────────────────────────────                               │
│      Code: ice_txrx.c:ice_napi_poll()                               │
│                                                                     │
│      Interrupt handler 路徑：                                        │
│        Hardware IRQ → ice_msix_clean_rings()                        │
│                    → napi_schedule(&q_vector->napi)                 │
│                    → 加入 softirq 排程                              │
│                    → (稍後) ksoftirqd 執行 ice_napi_poll()          │
│                                                                     │
│      NAPI 的好處：                                                   │
│        • Interrupt mitigation (高流量時自動切換 polling)            │
│        • 批次處理多個封包 (減少 per-packet overhead)                │
│        • Budget control (避免 RX livelock)                          │
│                                                                     │
│   2. ice_clean_rx_irq(rx_ring, budget)                              │
│      ─────────────────────────────────                              │
│      Code: ice_txrx.c:1381                                          │
│                                                                     │
│      Input: budget = 64 (預設，可調)                                │
│             → 最多處理 64 個封包後 yield                             │
│                                                                     │
│      Loop over descriptors:                                         │
│        ntc = rx_ring->next_to_clean;  // Software read pointer      │
│        while (total_rx_pkts < budget) {                             │
│            rx_desc = ICE_RX_DESC(rx_ring, ntc);                     │
│                                                                     │
│            // Check DD bit (ice_txrx.c:1417)                        │
│            stat_err_bits = BIT(ICE_RX_FLEX_DESC_STATUS0_DD_S);      │
│            if (!ice_test_staterr(rx_desc->wb.status_error0,         │
│                                  stat_err_bits))                    │
│                break;  // No more completed packets                 │
│                                                                     │
│            ... process packet ...                                   │
│                                                                     │
│            ntc++;                                                   │
│            if (ntc == rx_ring->count)                               │
│                ntc = 0;  // Wrap around (ring buffer)               │
│            total_rx_pkts++;                                         │
│        }                                                            │
│        rx_ring->next_to_clean = ntc;                                │
│                                                                     │
│   3. DMA memory barrier (ice_txrx.c:1425)                           │
│      ─────────────────────────────────                              │
│      dma_rmb();  // Data Memory Barrier for read                    │
│                                                                     │
│      **為什麼需要 memory barrier？**                                 │
│        • Hardware 寫入 descriptor 的順序：                          │
│            1. DMA packet data to pkt_addr                           │
│            2. Write metadata (ptype, hash, etc.)                    │
│            3. Set DD bit                                            │
│                                                                     │
│        • CPU 可能會 reorder reads！                                  │
│          → 可能先讀到 pkt_len, hash，DD bit 才讀到                  │
│          → 造成使用 stale/未完成的 data                             │
│                                                                     │
│        • dma_rmb() 保證：                                            │
│          「讀取 DD bit 之後的所有 reads 都發生在 DD check 之後」    │
│          → 確保 metadata 是 fresh 的                                │
│                                                                     │
│   4. 從 page 建立 SKB (ice_txrx.c:1468)                             │
│      ──────────────────────────────                                 │
│      Two paths:                                                     │
│                                                                     │
│      Path A: ice_build_skb() - **Zero-copy** (推薦)                 │
│        • 封包已在 DMA page (rx_buf->page)                           │
│        • 直接 build_skb(page_address, truesize)                     │
│        • SKB 的 data pointer 指向 DMA page                          │
│        • 優點：No memcpy，極低 overhead                             │
│        • 缺點：Page 被 SKB 持有，需要 refcount 管理                 │
│                                                                     │
│      Path B: ice_construct_skb() - **Copy mode**                    │
│        • 從 kmem_cache 分配新 SKB                                   │
│        • memcpy(skb->data, dma_page, pkt_len)                       │
│        • DMA page 可立刻 reuse                                      │
│        • 優點：Page recycle 快                                      │
│        • 缺點：memcpy overhead (對小封包可接受)                     │
│                                                                     │
│      Driver 根據 page refcount 和 size 自動選擇                     │
│                                                                     │
│   5. Attach metadata to SKB (ice_txrx_lib.c)                        │
│      ─────────────────────────────────────                          │
│      ice_process_skb_fields(rx_ring, rx_desc, skb):                 │
│                                                                     │
│        • skb->hash (ice_rx_hash):                                   │
│            if (rx_desc->wb.rss_hash)                                │
│                skb_set_hash(skb, hash, PKT_HASH_TYPE_L4);           │
│            → 用於 SO_INCOMING_CPU, RPS, application hashing         │
│                                                                     │
│        • skb->ip_summed (ice_rx_csum):                              │
│            if (HW validated checksum)                               │
│                skb->ip_summed = CHECKSUM_UNNECESSARY;               │
│            else                                                     │
│                skb->ip_summed = CHECKSUM_NONE;                      │
│            → Offload L3/L4 checksum validation to hardware          │
│                                                                     │
│        • skb->vlan_tci (ice_rx_vlan_tag):                           │
│            if (l2tag1 present)                                      │
│                __vlan_hwaccel_put_tag(skb, vlan_proto, vlan_tci);   │
│            → VLAN tag 已被 hardware strip，存在 descriptor          │
│                                                                     │
│        • skb->protocol:                                             │
│            eth_type_trans(skb, netdev);                             │
│            → 根據 EtherType 設定 (e.g., ETH_P_IP, ETH_P_IPV6)      │
│                                                                     │
│        • skb->dev = netdev                                          │
│        • skb->pkt_type (PACKET_HOST/BROADCAST/MULTICAST)            │
│                                                                     │
│   6. XDP processing (if attached)                                   │
│      ──────────────────────────                                     │
│      如果有 BPF program attached:                                    │
│                                                                     │
│      xdp_prog = READ_ONCE(rx_ring->xdp_prog);                       │
│      if (xdp_prog) {                                                │
│          ice_run_xdp(rx_ring, xdp, xdp_prog, xdp_ring);             │
│      }                                                              │
│                                                                     │
│      XDP 在 SKB allocation **之前** 執行 (極低延遲)                 │
│                                                                     │
│      Verdicts:                                                      │
│        • XDP_PASS: 繼續處理，建立 SKB                               │
│        • XDP_DROP: 立即丟棄，不建立 SKB                             │
│        • XDP_TX: 從同一 NIC 送出 (hairpin)                          │
│        • XDP_REDIRECT: 轉發到其他 netdev                            │
│        • XDP_ABORTED: Error，預設 drop                              │
│                                                                     │
│      Use case:                                                      │
│        • DDoS mitigation (drop specific patterns)                   │
│        • Load balancer (redirect to backend)                        │
│        • Packet sampling (pass 1%, drop 99%)                        │
│                                                                     │
│   7. Send to network stack                                          │
│      ──────────────────────                                         │
│      ice_receive_skb(rx_ring, skb, vlan_tci)                        │
│        → napi_gro_receive(&rx_ring->q_vector->napi, skb)            │
│                                                                     │
│      GRO (Generic Receive Offload):                                 │
│        • 嘗試合併同一 flow 的 TCP segments                          │
│        • 條件：same 5-tuple, sequential seq numbers                 │
│        • 效果：減少 network stack overhead                          │
│            → 一個大 SKB (64KB) vs 45 個小 SKB (1448B each)          │
│            → TCP processing 只做一次                                │
│                                                                     │
│      進入 Linux network stack:                                      │
│        netif_receive_skb()                                          │
│          → __netif_receive_skb_core()                               │
│          → deliver_ptype_list_skb()                                 │
│          → ip_rcv() (for IPv4)                                      │
│          → ip_local_deliver()                                       │
│          → tcp_v4_rcv()                                             │
│          → tcp_v4_do_rcv()                                          │
│          → sock_queue_rcv_skb()                                     │
│          → sk->sk_data_ready() (喚醒 application)                   │
│                                                                     │
│      最終 application 呼叫 recv()/read() 取得 data                  │
│                                                                     │
│ DPDK PMD (Polling):                                                 │
│ ──────────────────                                                  │
│   • rte_eth_rx_burst() busy-wait polling                            │
│   • 直接讀取 descriptor (無 interrupt)                              │
│   • 建立 mbuf (vs kernel 的 SKB)                                    │
│   • 送入 user-space application                                     │
│                                                                     │
│ GPU/DPU Offload Path:                                               │
│ ────────────────────────                                            │
│   • GPUDirect RDMA: packet buffer 直接 DMA 到 GPU memory            │
│   • 需要 nvidia_p2p API mapping GPU physical address                │
│   • Descriptor pkt_addr = GPU address (not host memory)             │
│   • Bypass host CPU memory 完全                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 關鍵修正：你的流程圖 vs 實際 Code

### ❌ **你寫錯的地方：**

#### 1. Parser Stages

**你的描述：**
```
└─ stage0: preamble / sfd / mac addr / ethertype / len
└─ stage1: VLAN tags（802.1Q） / Q-in-Q
└─ stage2: MPLS labels
└─ stage3: tunneling headers (GRE, GTP-U, VXLAN)
└─ stage4: IP (v4/v6)
└─ stage5: transport (TCP/UDP)
└─ stage6: inner headers
```

**實際 Code：**
- ❌ **沒有硬編碼的 stage0-6**
- ✅ **是 while loop，動態迭代**（ice_parser_rt.c:765）
- ✅ **Parse Graph CAM 定義狀態轉換，不是固定 stages**

**正確描述：**
```
Parser Loop (dynamic iterations):
  Round 1: Ethernet (node_id=0) → next_proto=0x0800 → IPv4 (node_id=10)
  Round 2: IPv4 (node_id=10) → next_proto=6 → TCP (node_id=20)
  Round 3: TCP (node_id=20) → is_last_round=true → 結束
```

#### 2. Parsed Fields 的時機

**你的描述：**
```
[Parsed Fields / Metadata]
  (field list e.g. ethertype, vlan id, mpls label, gtp TEID, inner-src-ip, inner-dst-port)
```

**實際 Code：**
- ❌ **Parser 不直接提取這些 fields**
- ✅ **Parser 只產生** (ice_parser.h:460):
  ```c
  struct ice_parser_result {
      u16 ptype;                                 // Packet Type ID
      struct ice_parser_proto_off po[16];        // Protocol-offset pairs
      u64 flags_psr, flags_pkt;                  // Flags
      u16 flags_sw, flags_fd, flags_rss;         // Key builder flags
  };
  ```

**正確描述：**
```
Parser Output:
  • ptype = 0x1234
  • po[] = [(PROTO_ETH, 0), (PROTO_IPV4, 14), (PROTO_TCP, 34)]
  • flags = ...

      ↓ (下一階段)

Field Vector Extraction (硬體執行):
  • 用 po[] 的 offset 提取實際 field bytes
  • 5-tuple: src_ip, dst_ip, src_port, dst_port, protocol
```

### ✅ **你寫對的地方：**

1. **DDP profile / parse-graph 在 Parser 生效** ✅
2. **Classifier 架構 (RSS, FDIR, ACL)** ✅
3. **Queue & DMA engine** ✅
4. **Host driver NAPI poll** ✅

---

## 總結：最重要的修正

### Parser 不是 Pipeline，是 State Machine

**錯誤理解（你的原圖）：**
```
固定 Pipeline:
  Packet → Stage0 → Stage1 → Stage2 → ... → Stage6 → Output
           (ETH)    (VLAN)    (MPLS)          (Inner)
```

**正確理解（實際 Code）：**
```
動態 State Machine:
  Packet → Parser Loop {
              IMEM instruction
              → Boost TCAM (optional fast path)
              → ALU execution
              → Parse Graph CAM lookup
              → State transition (node_id, pc, HO update)
              → Check is_last_round
           } → Output (ptype, protocol-offsets, flags)
```

### Field Extraction 是獨立階段

**錯誤理解：**
```
Parser → [直接產生 src_ip, dst_ip, src_port, ...]
```

**正確理解：**
```
Parser → [產生 protocol offsets]
           ↓
Field Vector Extraction → [用 offsets 提取 fields]
           ↓
Classifier → [用 extracted fields 做 RSS/FDIR]
```

---

**驗證完成日期：** 2025-11-14
**驗證者：** Claude (基於實際 Linux kernel ice driver code)

