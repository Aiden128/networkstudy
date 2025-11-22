# ICE Parser 不做 Field Extraction - 完整解釋

> **核心問題**：為什麼說 "Parser 不做 field extraction，只產生 metadata"？

---

## 錯誤的理解（大多數人的直覺）

```
封包進來 (Ethernet | IPv4 | TCP | Data)
     ↓
Parser 執行
     ↓
輸出：
  src_ip    = 192.168.1.100   ← 你以為 parser 會提取這些值
  dst_ip    = 10.0.0.1
  src_port  = 12345
  dst_port  = 80
  protocol  = TCP
```

**這不是 ICE parser 做的事！**

---

## 實際的設計（基於 code）

### Parser 真正的輸出

**Code**: `ice_parser.h:460-471`

```c
struct ice_parser_result {
    u16 ptype;  // Packet Type ID，例如 0x1234 = "IPv4/TCP"

    struct ice_parser_proto_off po[16];  // Protocol-offset pairs
    int po_num;  // 有幾個協議層

    u64 flags_psr;   // Parser 狀態 flags
    u16 flags_fd;    // Flow Director flags
    u16 flags_rss;   // RSS flags
};

struct ice_parser_proto_off {
    u8 proto_id;   // Protocol ID (例如：PROTO_IPV4 = 6)
    u16 offset;    // 這個協議在封包中的 offset
};
```

### 實際範例

**封包內容**（Hex dump）:
```
Offset:  0                                                      14
         ↓                                                      ↓
Bytes:   [00 50 56 C0 00 01 ... 08 00]  [45 00 00 3C ...]  [06 ...]
         └────── Ethernet header ─────┘  └── IPv4 ──┘      └TCP...
                  (14 bytes)
```

**Parser 輸出**（`ice_parser_result`）:

```c
ptype = 0x0026;  // DDP profile 定義: "IPv4/TCP, no options"

po[0] = { proto_id: PROTO_ETH  (0), offset: 0  };   // Ethernet 從 byte 0 開始
po[1] = { proto_id: PROTO_IPV4 (6), offset: 14 };   // IPv4 從 byte 14 開始
po[2] = { proto_id: PROTO_TCP  (8), offset: 34 };   // TCP 從 byte 34 開始
po_num = 3;

flags_rss = 0x0001;  // 表示「這個 PTYPE 可用於 RSS」
flags_fd  = 0x0001;  // 表示「這個 PTYPE 可用於 FDIR」
```

**Parser 沒有告訴你**：
- ❌ Src IP 是多少
- ❌ Dst IP 是多少
- ❌ Src port 是多少
- ❌ 任何欄位的「值」

**Parser 只告訴你**：
- ✅ 這是 IPv4/TCP 封包（ptype）
- ✅ IPv4 header 從 byte 14 開始
- ✅ TCP header 從 byte 34 開始
- ✅ 可以對這種封包做 RSS/FDIR

---

## 那誰提取欄位值？

**答案：Field Vector (FV) Extraction - 硬體的下一個階段**

### Field Vector 的定義

**Code**: `ice_ddp.h:23-30`

```c
struct ice_fv_word {
    u8 prot_id;   // Protocol ID（哪個協議）
    u16 off;      // Offset within protocol（協議內的相對 offset）
};

struct ice_fv {
    struct ice_fv_word ew[48];  // 最多 48 個 extraction words
};
```

### Field Vector 怎麼用 Parser 的輸出

**範例：提取 IPv4 src address**

DDP profile 定義的 FV（針對 PTYPE=0x0026）：

```c
// 這是 DDP profile 的內容，由 ice_init_pkg() 載入
fv.ew[0] = { prot_id: PROTO_IPV4 (6), off: 12 };  // IPv4 src addr byte 0-1
fv.ew[1] = { prot_id: PROTO_IPV4 (6), off: 14 };  // IPv4 src addr byte 2-3
fv.ew[2] = { prot_id: PROTO_IPV4 (6), off: 16 };  // IPv4 dst addr byte 0-1
fv.ew[3] = { prot_id: PROTO_IPV4 (6), off: 18 };  // IPv4 dst addr byte 2-3
fv.ew[4] = { prot_id: PROTO_TCP  (8), off: 0  };  // TCP src port
fv.ew[5] = { prot_id: PROTO_TCP  (8), off: 2  };  // TCP dst port
```

**硬體執行提取（偽碼）**：

```c
// Input from Parser:
//   po[1] = {PROTO_IPV4, 14}  // IPv4 在 offset 14
//   po[2] = {PROTO_TCP, 34}   // TCP 在 offset 34

for (i = 0; i < fv.num_words; i++) {
    ew = fv.ew[i];

    // 找到對應協議的 offset
    proto_offset = find_proto_offset(po, ew.prot_id);

    // 計算絕對位置
    absolute_offset = proto_offset + ew.off;

    // 從封包提取 2 bytes
    extracted[i] = packet[absolute_offset : absolute_offset+1];
}

// 結果：
extracted[0] = packet[14+12] = packet[26:27]  // Src IP byte 0-1 = 0xC0A8
extracted[1] = packet[14+14] = packet[28:29]  // Src IP byte 2-3 = 0x0164
  → Src IP = 0xC0A80164 = 192.168.1.100

extracted[2] = packet[14+16] = packet[30:31]  // Dst IP byte 0-1 = 0x0A00
extracted[3] = packet[14+18] = packet[32:33]  // Dst IP byte 2-3 = 0x0001
  → Dst IP = 0x0A000001 = 10.0.0.1

extracted[4] = packet[34+0] = packet[34:35]   // Src port = 0x3039 = 12345
extracted[5] = packet[34+2] = packet[36:37]   // Dst port = 0x0050 = 80
```

---

## 為什麼要分兩階段？

### Bad Design（傳統做法）

```c
// Parser 直接提取所有可能的欄位
struct parser_output {
    u32 eth_src[6];
    u32 eth_dst[6];
    u32 vlan_id;
    u32 ipv4_src;
    u32 ipv4_dst;
    u32 ipv6_src[16];
    u32 ipv6_dst[16];
    u16 tcp_sport;
    u16 tcp_dport;
    u16 udp_sport;
    u16 udp_dport;
    // ... 上百個欄位
};
```

**問題**：
- 浪費！IPv4 封包不需要 `ipv6_src`
- 不可擴展！新協議（例如 QUIC）要改 parser
- 硬體成本！要 buffer 所有可能的欄位

### Good Design（ICE 的做法）

**階段 1: Parser → 產生 metadata**
```c
struct parser_output {
    u16 ptype;                      // 1 個 ID
    struct proto_off po[16];        // 最多 16 個 (proto, offset) pairs
};
```
- 小！只有 ~40 bytes
- 通用！任何協議都是 (proto_id, offset)
- 可擴展！新協議只是新的 proto_id

**階段 2: FV Extraction → 提取需要的欄位**
```c
// DDP profile 定義「對於這個 PTYPE，提取哪些欄位」
// 不同的 use case 可以有不同的 FV
```
- 靈活！RSS 只需要 5-tuple，FDIR 可能要更多
- 可程式化！更新 DDP profile 就能改提取規則
- 節省硬體！只提取真正需要的欄位

---

## 實際 Code 驗證

### Parser 產生 metadata

**Code**: `ice_parser_rt.c:727-743` - `ice_result_resolve()`

```c
static void ice_result_resolve(struct ice_parser_rt *rt,
                                struct ice_parser_result *rslt)
{
    // 1. 產生 PTYPE
    rslt->ptype = ice_ptype_resolve(rt);

    // 2. 產生 protocol-offset pairs
    memset(rslt->po, 0, sizeof(rslt->po));
    rslt->po_num = 0;
    ice_proto_off_resolve(rt, rslt);  // 填充 po[] 陣列

    // 3. 產生 flags
    rslt->flags_psr = rt->flags_psr;
    rslt->flags_pkt = rt->flags_pkt;
    rslt->flags_sw = ice_flg_resolve(rt->gpr[ICE_GPR_FLG_IDX],
                                      rt->pu->flg_msk[rt->parser_idx],
                                      rt->pu->flg_sws[rt->parser_idx]);
    rslt->flags_fd = ice_flg_resolve(rt->gpr[ICE_GPR_FLG_IDX],
                                      rt->pu->flg_msk[rt->parser_idx],
                                      rt->pu->fd_flg_msk[rt->parser_idx]);
    // ... 更多 flags
}
```

**看到了嗎？完全沒有提取 IP address 或 port！**

### FV 提取在硬體（driver 只設定 FV table）

**Code**: `ice_ddp.c` - DDP loading

Driver 下載 DDP profile 時，會載入 FV table 到硬體。實際提取發生在硬體 pipeline，driver code 看不到。

---

## 類比：地圖 vs 寶藏

### Parser = 畫地圖
```
Parser 告訴你：
  「寶藏在第 3 個島嶼，從島嶼入口走 100 步」

  → po[2] = {PROTO_TCP, offset: 34}
```

### Field Vector = 挖寶藏
```
FV 根據地圖去挖：
  「去第 3 個島嶼（PROTO_TCP），從入口（offset 34）走 0 步，挖 2 bytes」

  → extracted_port = packet[34+0 : 34+1]
```

**Parser 不挖寶藏，只畫地圖。**

---

## 總結：Metadata vs Data

| 類型 | Parser 輸出 | Field Vector 輸出 |
|------|-------------|------------------|
| **性質** | Metadata（描述資料的資料） | Data（實際的欄位值） |
| **內容** | "IPv4 header 在 byte 14" | "Src IP = 192.168.1.100" |
| **大小** | 固定（~40 bytes） | 可變（取決於 FV 定義） |
| **用途** | 告訴下一階段「去哪裡找」 | 實際用於 RSS/FDIR 比對 |
| **可程式化** | 不可（硬體 microcode） | 可（DDP profile） |

**這就是「Parser 不做 field extraction，只產生 metadata」的真正意思！**

Parser 只是個「偵探」，告訴你「證據在哪裡」，而不是直接把證據拿給你。真正拿證據的工作由 Field Vector Extraction 硬體完成。

---


這是教科書級別的硬體設計。

---

**驗證日期**: 2025-11-14
**基於**: Linux kernel `drivers/net/ethernet/intel/ice/`
