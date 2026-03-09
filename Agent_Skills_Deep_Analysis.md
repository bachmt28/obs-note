# Agent Skills Specification — Phân Tích Sâu & Hướng Dẫn Implement

## Mục lục

1. [Tổng quan kiến trúc](#1-tổng-quan-kiến-trúc)
2. [Specification chi tiết — Từng field một](#2-specification-chi-tiết)
3. [Progressive Disclosure — Triết lý thiết kế cốt lõi](#3-progressive-disclosure)
4. [Internal Architecture — Cách hoạt động bên trong](#4-internal-architecture)
5. [Execution Lifecycle — Vòng đời thực thi](#5-execution-lifecycle)
6. [Hai cách tích hợp Skills vào hệ thống](#6-hai-cách-tích-hợp)
7. [Design Patterns khi viết Skills](#7-design-patterns)
8. [Ánh xạ vào dự án Ecommerce Agent](#8-ánh-xạ-ecommerce)
9. [So sánh: Skills vs Tools vs MCP vs Prompt Engineering](#9-so-sánh)
10. [Implementation Blueprint — Kế hoạch triển khai](#10-implementation-blueprint)

---

## 1. Tổng quan kiến trúc

### 1.1. Agent Skills là gì — Định nghĩa chính xác

Agent Skills **KHÔNG phải là executable code**. Chúng **KHÔNG chạy** Python hay JavaScript trực tiếp. Chúng cũng **KHÔNG được hardcode** vào system prompt.

> **Agent Skills = Prompt Template + Conversation Context Injection + Execution Context Modification + Optional Scripts/References/Assets**

Nói cách khác, Skills là **"prompt-based conversation and execution context modifiers"** — chúng sửa đổi bối cảnh hội thoại và bối cảnh thực thi của agent, thay vì trực tiếp thực hiện hành động.

### 1.2. Vị trí của Skills trong AI Agent Stack

```
┌─────────────────────────────────────────────────────────┐
│                    USER REQUEST                          │
│        "Tìm giúp tôi giày Nike màu đỏ size 42"         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
│   │   SKILLS    │  │    TOOLS     │  │     MCP      │  │
│   │  (Biết      │  │  (Có khả    │  │  (Kết nối    │  │
│   │  CÁCH LÀM)  │  │  năng LÀM)  │  │   dữ liệu)  │  │
│   │             │  │              │  │              │  │
│   │ "Khi tư vấn │  │ search_rag() │  │ Database     │  │
│   │  sản phẩm,  │  │ create_order │  │ Connection   │  │
│   │  hãy hỏi    │  │ check_stock  │  │ External API │  │
│   │  size, màu, │  │              │  │              │  │
│   │  ngân sách" │  │              │  │              │  │
│   └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  │
│          │                │                  │          │
│          ▼                ▼                  ▼          │
│   ┌─────────────────────────────────────────────────┐   │
│   │              LLM (Claude, GPT, etc.)            │   │
│   │         Reasoning + Action Selection            │   │
│   └─────────────────────────────────────────────────┘   │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │          STATE MANAGEMENT (LangGraph)            │   │
│   │     messages, products, orderData, nextAction    │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**Phân biệt rõ 3 tầng:**

| Tầng       | Vai trò                                      | Ví dụ cụ thể                                    |
| ---------- | -------------------------------------------- | ----------------------------------------------- |
| **Skills** | Dạy agent **CÁCH** thực hiện tác vụ          | "Khi tư vấn sản phẩm, luôn hỏi ngân sách trước" |
| **Tools**  | Cho agent **KHẢ NĂNG** thực hiện hành động   | Hàm `search_product()`, `create_order()`        |
| **MCP**    | Cho agent **KẾT NỐI** đến dữ liệu và dịch vụ | Postgres connection, Stripe API                 |

---

## 2. Specification chi tiết

### 2.1. Directory Structure — Cấu trúc thư mục

```
skill-name/                    # Tên folder = tên skill
├── SKILL.md                   # BẮT BUỘC — File chính
├── LICENSE.txt                # Tùy chọn — Giấy phép
├── scripts/                   # Tùy chọn — Code thực thi
│   ├── search_product.py
│   ├── validate_order.py
│   └── format_response.sh
├── references/                # Tùy chọn — Tài liệu tham khảo
│   ├── REFERENCE.md
│   ├── product-categories.md
│   └── shipping-rules.md
└── assets/                    # Tùy chọn — Tài nguyên tĩnh
    ├── response-template.json
    ├── order-form-template.html
    └── brand-guidelines.md
```

**Quy tắc quan trọng:**
- Tên folder **PHẢI** khớp với field `name` trong SKILL.md
- File SKILL.md là file DUY NHẤT bắt buộc
- Các thư mục scripts/, references/, assets/ chỉ được load **khi cần** (progressive disclosure)

### 2.2. SKILL.md Format — Phân tích từng field

File SKILL.md gồm 2 phần: **Frontmatter** (YAML) + **Body** (Markdown)

```yaml
---
# ═══════════════════════════════════════════
#  PHẦN 1: FRONTMATTER (Metadata — cấu hình)
# ═══════════════════════════════════════════

name: ecommerce-sales-advisor          # BẮT BUỘC
# Quy tắc:
#   - Tối đa 64 ký tự
#   - Chỉ lowercase, số, và dấu gạch ngang
#   - KHÔNG bắt đầu/kết thúc bằng "-"
#   - KHÔNG có "--" liên tiếp
#   - PHẢI khớp với tên folder cha

description: >                          # BẮT BUỘC
  Tư vấn sản phẩm ecommerce, giúp khách hàng tìm kiếm,
  so sánh, và chọn sản phẩm phù hợp. Sử dụng khi khách hàng
  hỏi về sản phẩm, cần tư vấn, hoặc muốn so sánh các lựa chọn.
# Quy tắc:
#   - Tối đa 1024 ký tự
#   - MÔ TẢ CẢ: skill làm gì + KHI NÀO dùng
#   - Nên chứa keywords giúp agent nhận diện task
#   - Đây là thứ DUY NHẤT agent đọc ở giai đoạn discovery

license: Apache-2.0                     # TÙY CHỌN

compatibility: >                         # TÙY CHỌN (tối đa 500 ký tự)
  Requires pgvector, langchain, and internet access for RAG queries
# Khi nào cần: skill có yêu cầu đặc biệt về môi trường

metadata:                                # TÙY CHỌN
  author: your-team
  version: "1.0"
  domain: ecommerce
# Map string → string, dùng cho metadata tùy ý

allowed-tools: >                         # TÙY CHỌN (Experimental)
  Bash(python:*) Read Write
# Danh sách tools được pre-approve (không cần hỏi user)
# Space-delimited, hỗ trợ wildcards

---
```

### 2.3. Body Content — Nội dung hướng dẫn

```markdown
# Ecommerce Sales Advisor

## Mục đích
Hướng dẫn agent tư vấn sản phẩm chuyên nghiệp cho khách hàng
ecommerce, bao gồm tìm kiếm, so sánh, và đề xuất sản phẩm.

## Quy trình tư vấn

### Bước 1: Phân tích nhu cầu khách hàng
- Xác định loại sản phẩm khách quan tâm
- Hỏi về ngân sách (nếu chưa đề cập)
- Hỏi về size/màu sắc/thương hiệu ưa thích

### Bước 2: Tìm kiếm sản phẩm
Sử dụng RAG tool để query sản phẩm liên quan:
`python {baseDir}/scripts/search_product.py --query "$USER_QUERY"`

Tham khảo danh mục sản phẩm tại:
[Danh mục sản phẩm](references/product-categories.md)

### Bước 3: Trình bày kết quả
- Hiển thị tối đa 5 sản phẩm phù hợp nhất
- Nêu ưu/nhược điểm của từng sản phẩm
- Đề xuất top pick dựa trên nhu cầu khách

### Bước 4: Xử lý so sánh
Nếu khách muốn so sánh, tạo bảng so sánh gồm:
tên, giá, đặc điểm nổi bật, đánh giá

## Edge Cases
- Nếu không tìm thấy sản phẩm: đề xuất sản phẩm tương tự
- Nếu khách chưa rõ nhu cầu: hỏi thêm 1-2 câu gợi ý
- Nếu sản phẩm hết hàng: thông báo và đề xuất thay thế

## Output Format
Luôn trả lời bằng tiếng Việt, thân thiện, chuyên nghiệp.
Giới hạn mỗi câu trả lời dưới 200 từ trừ khi so sánh chi tiết.

## Resources
- Scripts: {baseDir}/scripts/
- Danh mục: {baseDir}/references/product-categories.md
- Template response: {baseDir}/assets/response-template.json
```

### 2.4. Phân tích constraint của Specification

| Constraint | Giá trị | Lý do |
|-----------|---------|-------|
| SKILL.md body < 5000 tokens | ~800 dòng | Tránh overwhelm context window |
| SKILL.md < 500 dòng | Hard limit | Chuyển chi tiết sang references/ |
| Metadata ~100 tokens/skill | name + description | Progressive disclosure Phase 1 |
| References loaded on demand | Nhỏ hơn = tốt hơn | Tối ưu context usage |
| `{baseDir}` cho paths | Không hardcode | Portability across environments |
| name khớp folder name | Validation rule | Consistency & discovery |

---

## 3. Progressive Disclosure — Triết lý thiết kế cốt lõi

Đây là concept QUAN TRỌNG NHẤT để hiểu tại sao Agent Skills hiệu quả. Nó giải quyết bài toán **Context Window Management** — bài toán mà bất kỳ ai build AI Agent đều phải đối mặt.

### 3.1. Ba giai đoạn tải thông tin

```
Giai đoạn 1: DISCOVERY (~100 tokens/skill)
═══════════════════════════════════════════
Agent chỉ load name + description của TẤT CẢ skills.

Nếu có 10 skills → ~1000 tokens = rất nhỏ so với context window

┌─────────────────────────────────────────────┐
│ "ecommerce-sales-advisor": Tư vấn sản phẩm │
│ "order-management": Quản lý đặt hàng        │
│ "general-support": Hỗ trợ chung             │
└─────────────────────────────────────────────┘


Giai đoạn 2: ACTIVATION (< 5000 tokens)
═══════════════════════════════════════════
Agent CHỌN skill phù hợp → Load TOÀN BỘ SKILL.md

┌─────────────────────────────────────────────┐
│ # Ecommerce Sales Advisor                   │
│ ## Quy trình tư vấn                         │
│ ### Bước 1: Phân tích nhu cầu...            │
│ ### Bước 2: Tìm kiếm sản phẩm...           │
│ ### Bước 3: Trình bày kết quả...            │
│ ## Edge Cases...                             │
│ ## Output Format...                          │
└─────────────────────────────────────────────┘


Giai đoạn 3: EXECUTION (as needed)
═══════════════════════════════════════════
Trong quá trình thực thi, agent load scripts/references/assets KHI CẦN

┌─────────────────────────────────────────────┐
│ Read: references/product-categories.md       │
│ Execute: scripts/search_product.py           │
│ Reference: assets/response-template.json     │
└─────────────────────────────────────────────┘
```

### 3.2. Tại sao Progressive Disclosure quan trọng

**Không có Progressive Disclosure:**
- 10 skills × 5000 tokens/skill = 50,000 tokens loaded upfront
- Chiếm ~25% context window của Claude (200K)
- Tăng chi phí API, giảm accuracy

**Có Progressive Disclosure:**
- Startup: 10 skills × 100 tokens = 1,000 tokens
- Khi cần: +5,000 tokens cho skill được chọn
- Runtime: +vài nghìn tokens cho references cụ thể
- Tổng: ~7,000 tokens thay vì 50,000 → **tiết kiệm 86%**

### 3.3. So sánh với cách tiếp cận trong dự án Ecommerce hiện tại

Trong document State Machine, phần Prompt Engineering mô tả việc tạo prompt → test → improve nhiều lần. Mỗi prompt cho Supervisor, Sales Agent, Order Agent đều được load đầy đủ vào context mỗi lần gọi. Với Agent Skills, ta có thể:

- **Supervisor**: Chỉ load metadata của Sales Skill, Order Skill, General Skill (~300 tokens)
- **Khi route sang Sales**: Mới load full Sales Skill (~3000-5000 tokens)
- **Khi Sales cần RAG**: Mới load references/product-categories.md

→ **Context window sạch hơn, agent tập trung hơn, kết quả chính xác hơn**

---

## 4. Internal Architecture — Cách hoạt động bên trong

### 4.1. Skill Tool — Meta-Tool Architecture

Trong implementation thực tế (ví dụ Claude Code), Skills hoạt động qua một **meta-tool** gọi là "Skill tool" (viết hoa). Đây là một tool đặc biệt nằm trong `tools` array cùng với các tool khác (Read, Write, Bash...).

```
API Request Structure:
{
  "model": "claude-sonnet-4-5-20250929",
  "system": "You are an AI Agent...",           // System prompt
  "messages": [...],                            // Lịch sử hội thoại
  "tools": [
    {
      "name": "Skill",                          // ← Meta-tool
      "description": "Execute a skill...\n
        <available_skills>
          'sales-advisor': Tư vấn sản phẩm...
          'order-mgmt': Quản lý đơn hàng...
        </available_skills>",
      "input_schema": {
        "properties": {
          "command": { "type": "string" }        // ← Tên skill
        }
      }
    },
    { "name": "search_product", ... },          // Tool thường
    { "name": "create_order", ... },            // Tool thường
  ]
}
```

**Điểm mấu chốt:**
- Skills **KHÔNG** nằm trong system prompt
- Skills nằm trong `tools` array, cụ thể trong `description` của Skill meta-tool
- Agent **TỰ QUYẾT ĐỊNH** chọn skill nào qua LLM reasoning (không có algorithmic matching)

### 4.2. Two-Message Pattern — Kỹ thuật dual-channel

Khi skill được invoke, hệ thống inject **2 messages** vào conversation:

```
Message 1: Metadata (isMeta: false → USER THẤY ĐƯỢC)
─────────────────────────────────────────────────────
<command-message>Skill "sales-advisor" đang load</command-message>
<command-name>sales-advisor</command-name>

→ Ngắn gọn (~50-200 ký tự), cho user biết đang xảy ra gì


Message 2: Full Prompt (isMeta: true → USER KHÔNG THẤY)
─────────────────────────────────────────────────────
"Bạn là chuyên gia tư vấn sản phẩm ecommerce.

Quy trình:
1. Phân tích nhu cầu khách...
2. Sử dụng RAG để tìm sản phẩm...
3. Trình bày kết quả...
..."

→ Dài (~500-5000 từ), chỉ gửi cho LLM API
```

**Tại sao cần 2 messages thay vì 1?**

| Nếu chỉ 1 message visible | Nếu chỉ 1 message hidden | 2 messages (giải pháp) |
|---|---|---|
| User thấy 5000 từ instructions → UI unusable | User không biết gì → mất transparency | Message 1: transparency cho user |
| Lộ implementation details | Không biết skill nào đang chạy | Message 2: instructions cho LLM |

### 4.3. Execution Context Modification

Ngoài conversation context (messages), Skills còn thay đổi **execution context**:

```javascript
contextModifier(context) {
  // 1. Pre-approve tools (không cần hỏi user)
  allowedTools: ["Bash(python:*)", "Read", "Write"]

  // 2. Override model (dùng model mạnh hơn nếu cần)
  model: "claude-opus-4-20250514"

  // 3. Điều chỉnh thinking tokens
  thinkingTokens: adjustedValue
}
```

---

## 5. Execution Lifecycle — Vòng đời thực thi

### 5.1. Full Lifecycle cho 1 request

Áp dụng vào bài toán Ecommerce — User hỏi: **"Tìm cho tôi giày Nike màu đỏ"**

```
PHASE 1: DISCOVERY (Startup)
═══════════════════════════════
Agent scan skills directories
→ Parse frontmatter từ mỗi SKILL.md
→ Build danh sách:
   "sales-advisor": Tư vấn sản phẩm ecommerce...
   "order-management": Quản lý đặt hàng...
   "general-support": Hỗ trợ chung...
→ Embed vào Skill tool description


PHASE 2: SKILL SELECTION (LLM Reasoning)
═══════════════════════════════
User: "Tìm cho tôi giày Nike màu đỏ"
→ LLM đọc <available_skills>
→ LLM reasoning: "User muốn tìm sản phẩm
   → match với 'sales-advisor'"
→ LLM trả về: { tool: "Skill", command: "sales-advisor" }

⚠️ KHÔNG CÓ algorithmic matching
⚠️ KHÔNG CÓ keyword matching
⚠️ KHÔNG CÓ embedding similarity
→ 100% LLM reasoning qua transformer forward pass


PHASE 3: SKILL EXECUTION (Meta-tool)
═══════════════════════════════
3a. Validation
   ✓ Skill "sales-advisor" tồn tại
   ✓ Type = "prompt"
   ✓ disableModelInvocation = false

3b. Permission Check
   → User approve (hoặc auto-allow nếu configured)

3c. Load SKILL.md
   → Parse full markdown content
   → Resolve {baseDir} paths

3d. Inject Messages
   → Message 1: <command-message> (visible)
   → Message 2: Full prompt (hidden, isMeta: true)

3e. Modify Execution Context
   → Pre-approve allowed-tools
   → Override model nếu specified


PHASE 4: TASK EXECUTION (With Skill Context)
═══════════════════════════════
LLM nhận được context mới bao gồm:
- Original user message
- Skill instructions (từ SKILL.md)
- Pre-approved tools

LLM theo instructions trong skill:
1. "Phân tích nhu cầu" → Nike, đỏ, giày
2. "Sử dụng RAG" → Gọi search_product tool
3. "Trình bày kết quả" → Format theo template

→ Trả kết quả cho user
→ Kết thúc hoặc return to supervisor
```

### 5.2. Sequence Diagram

```
User          Agent         Skill Meta-Tool      LLM API        Tools
 │              │                │                  │              │
 │──"Tìm giày"→│                │                  │              │
 │              │──check skills──│                  │              │
 │              │←─skill list────│                  │              │
 │              │                │                  │              │
 │              │──────────────send to API──────────│              │
 │              │               (+ skill list       │              │
 │              │                in Skill tool desc) │              │
 │              │                │                  │              │
 │              │←─────tool_use: Skill("sales")─────│              │
 │              │                │                  │              │
 │              │──invoke skill─→│                  │              │
 │              │                │──load SKILL.md   │              │
 │              │                │──inject msgs     │              │
 │              │                │──modify context   │              │
 │              │←──result + new messages────────────│              │
 │              │                │                  │              │
 │              │──────────────send to API──────────│              │
 │              │               (+ skill prompt     │              │
 │              │                + original msg)     │              │
 │              │                │                  │              │
 │              │←───tool_use: search_product()─────│              │
 │              │                │                  │──RAG query──→│
 │              │                │                  │←──products───│
 │              │                │                  │              │
 │              │←───────formatted response─────────│              │
 │──response───←│                │                  │              │
```

---

## 6. Hai cách tích hợp Skills vào hệ thống

### 6.1. Filesystem-based (Khuyến nghị cho dự án có computer environment)

Agent hoạt động trong môi trường có bash/unix. Skills được activate khi model gọi lệnh shell.

```
Workflow:
1. Agent startup → scan directories cho SKILL.md files
2. Parse frontmatter → build skill registry
3. Inject metadata vào system prompt dạng XML:

<available_skills>
  <skill>
    <name>sales-advisor</name>
    <description>Tư vấn sản phẩm ecommerce...</description>
    <location>/path/to/skills/sales-advisor/SKILL.md</location>
  </skill>
  <skill>
    <name>order-management</name>
    <description>Quản lý đặt hàng và giao hàng...</description>
    <location>/path/to/skills/order-management/SKILL.md</location>
  </skill>
</available_skills>

4. Khi cần activate → agent gọi: cat /path/to/skills/sales-advisor/SKILL.md
5. Resources truy cập qua shell commands
```

**Ưu điểm:** Mạnh nhất, hỗ trợ đầy đủ scripts, references, assets

### 6.2. Tool-based (Phù hợp cho custom agent frameworks)

Agent không có môi trường computer riêng. Thay vào đó, implement tools để model trigger skills.

```
Workflow:
1. Tạo một "Skill" tool trong tools array
2. Description của tool chứa danh sách skills
3. Khi model invoke Skill tool → code bên ngoài load SKILL.md
4. Inject nội dung skill vào conversation context
5. Tiếp tục conversation với context mới
```

**Ưu điểm:** Hoạt động với bất kỳ agent nào có thể đọc files, linh hoạt hơn

### 6.3. Token Budget và Security

```
Token Budget:
─────────────
- Metadata phase:       ~50-100 tokens/skill
- SKILL.md body:        < 5,000 tokens (khuyến nghị)
- Tổng available_skills: < 15,000 characters (default limit)

Security Considerations:
────────────────────────
- Scripts chạy trực tiếp trên local machine → KHÔNG có sandbox mặc định
- Luôn review skill scripts trước khi dùng (đặc biệt từ bên thứ 3)
- Cân nhắc chạy trong container (Docker) để giới hạn access
- Hiện chưa có built-in human-in-the-loop cho script execution
```

---

## 7. Design Patterns khi viết Skills

### 7.1. Pattern: Script Automation

**Khi nào dùng:** Tác vụ phức tạp cần logic chính xác, lặp lại

```markdown
# Trong SKILL.md:
Sử dụng script tìm kiếm sản phẩm:
`python {baseDir}/scripts/search_product.py --query "$USER_QUERY" --limit 5`

Parse kết quả JSON và trình bày cho khách hàng.
```

### 7.2. Pattern: Read → Process → Write

**Khi nào dùng:** Đọc input, xử lý theo instructions, tạo output

```markdown
# Quy trình xử lý đơn hàng:
1. Đọc thông tin đơn hàng từ state
2. Validate theo quy tắc trong references/shipping-rules.md
3. Format theo template assets/order-confirmation.json
4. Trả kết quả cho khách
```

### 7.3. Pattern: Search → Analyze → Report

**Khi nào dùng:** Tìm kiếm thông tin, phân tích, báo cáo

```markdown
# Quy trình tư vấn sản phẩm:
1. Dùng RAG tool tìm sản phẩm liên quan
2. Đọc chi tiết từng sản phẩm
3. So sánh dựa trên tiêu chí khách hàng
4. Tạo báo cáo so sánh có cấu trúc
```

### 7.4. Pattern: Wizard-Style Multi-Step

**Khi nào dùng:** Quy trình thu thập thông tin nhiều bước (ví dụ đặt hàng)

```markdown
# Quy trình đặt hàng — Thu thập thông tin

### Bước 1: Xác nhận sản phẩm
- Liệt kê sản phẩm khách chọn
- Xác nhận số lượng, variant (size/màu)
- Chờ khách xác nhận trước khi tiếp

### Bước 2: Thông tin giao hàng
- Hỏi tên người nhận
- Hỏi số điện thoại
- Hỏi địa chỉ giao hàng
- Chờ xác nhận

### Bước 3: Tạo đơn hàng
- Tổng hợp thông tin
- Gọi create_order tool
- Xác nhận mã đơn với khách
```

### 7.5. Pattern: Iterative Refinement

**Khi nào dùng:** Cần nhiều lượt xử lý với độ sâu tăng dần

```markdown
# Phân tích nhu cầu khách hàng

### Lượt 1: Nhận diện nhanh
- Xác định category sản phẩm
- Nhận diện keyword chính

### Lượt 2: Phân tích sâu
- Hỏi thêm về use case
- Xác định ngân sách
- Tìm hiểu preference

### Lượt 3: Đề xuất cá nhân hóa
- Kết hợp tất cả thông tin
- Đề xuất TOP 3 sản phẩm
- Giải thích lý do chọn
```

---

## 8. Ánh xạ vào dự án Ecommerce Agent

### 8.1. Từ Multi-Agent Architecture → Skills Architecture

Dự án Ecommerce hiện tại (theo document State Machine) có kiến trúc:

```
Hiện tại: Multi-Agent (3 agents riêng biệt)
═══════════════════════════════════════════
Supervisor Agent → route → Sales Agent
                         → Order Agent
                         → General Agent

Mỗi agent có prompt riêng, tools riêng, logic riêng
```

**Với Agent Skills, có 2 hướng tiếp cận:**

#### Hướng A: Skills bổ trợ cho Multi-Agent (Khuyến nghị — evolutionary)

Giữ nguyên kiến trúc Multi-Agent, dùng Skills để chuẩn hóa hành vi mỗi agent:

```
skills/
├── supervisor-routing/
│   └── SKILL.md         # Quy tắc phân loại, routing logic
├── sales-advisor/
│   ├── SKILL.md         # Quy trình tư vấn
│   ├── scripts/
│   │   └── rag_search.py
│   └── references/
│       ├── product-categories.md
│       └── upsell-rules.md
├── order-management/
│   ├── SKILL.md         # Quy trình đặt hàng
│   ├── scripts/
│   │   ├── validate_order.py
│   │   └── create_order.py
│   └── references/
│       └── shipping-rules.md
└── general-support/
    ├── SKILL.md         # Hỗ trợ chung
    └── references/
        └── store-policies.md
```

**Lợi ích:**
- Version control được prompt/instructions qua Git
- Có thể chia sẻ, reuse skills giữa các dự án
- Tách biệt rõ ràng giữa "cách làm" (skills) và "khả năng" (tools)
- Test và improve skills INDEPENDENTLY mà không ảnh hưởng code

#### Hướng B: Single Agent + Dynamic Skills (Revolutionary — tương lai)

Thay vì 3 agents riêng biệt, dùng 1 agent runtime + load skills theo context:

```
Single Agent Runtime
    │
    ├── Load "supervisor-routing" skill
    │   → Phân loại nhu cầu khách
    │
    ├── Nếu cần tư vấn → Load "sales-advisor" skill
    │   → Tư vấn sản phẩm
    │
    ├── Nếu cần đặt hàng → Load "order-management" skill
    │   → Thu thập thông tin, tạo đơn
    │
    └── Nếu câu hỏi chung → Load "general-support" skill
        → Trả lời chính sách, FAQ
```

**Lợi ích:** Kiến trúc đơn giản hơn, dễ maintain, dễ mở rộng

### 8.2. Thiết kế Skills cụ thể cho Ecommerce

#### Skill 1: `supervisor-routing`

```yaml
---
name: supervisor-routing
description: >
  Phân loại và điều hướng yêu cầu khách hàng ecommerce.
  Sử dụng khi nhận tin nhắn mới từ khách để quyết định
  chuyển đến sales, order, hay general support.
---

# Supervisor Routing

## Quy tắc phân loại

### Chuyển sang Sales khi:
- Khách hỏi về sản phẩm (tìm kiếm, tư vấn, so sánh)
- Khách nhắc đến tên sản phẩm, category, giá
- Khách muốn xem/tìm hàng

### Chuyển sang Order khi:
- Khách muốn mua/đặt hàng/chốt đơn
- Khách đề cập đến giỏ hàng, thanh toán
- Khách cung cấp thông tin giao hàng (địa chỉ, SĐT)
- Khách hỏi về đơn hàng đã đặt

### Chuyển sang General khi:
- Khách hỏi chính sách (đổi trả, bảo hành)
- Khách chat linh tinh, không liên quan sản phẩm/đơn hàng
- Khách cần hỗ trợ kỹ thuật website

## Context Awareness
- Xem xét toàn bộ lịch sử chat, không chỉ tin nhắn cuối
- Nếu khách vừa tư vấn xong sản phẩm rồi nói "lấy cái đó"
  → đây là Order, không phải Sales
- Set returnToSupervisor = true để tiếp tục phục vụ sau khi
  sub-task hoàn thành

## Output Format
Trả về JSON: { "nextAction": "sales|order|general",
               "reasoning": "...",
               "returnToSupervisor": true|false }
```

#### Skill 2: `sales-advisor`

```yaml
---
name: sales-advisor
description: >
  Tìm kiếm, tư vấn, so sánh sản phẩm ecommerce. Sử dụng khi
  khách hàng hỏi về sản phẩm, cần gợi ý, muốn so sánh, hoặc
  tìm kiếm sản phẩm theo nhu cầu cụ thể.
allowed-tools: Read Write
metadata:
  author: ecommerce-team
  version: "1.2"
---

# Sales Advisor

## Vai trò
Bạn là nhân viên tư vấn bán hàng chuyên nghiệp. Mục tiêu
là giúp khách tìm được sản phẩm phù hợp NHẤT, không phải
đắt nhất.

## Quy trình tư vấn

### 1. Phân tích nhu cầu
- Xác định: loại sản phẩm, use case, ngân sách, sở thích
- Nếu thiếu thông tin: hỏi TỐI ĐA 2 câu, tránh tra hỏi

### 2. Tìm kiếm sản phẩm
- Sử dụng search_product tool với query ngữ nghĩa
- Kết hợp filter: category, price range, availability
- Tham khảo danh mục tại references/product-categories.md

### 3. Trình bày kết quả
- Hiển thị 3-5 sản phẩm phù hợp nhất
- Mỗi sản phẩm: tên, giá, 1-2 điểm nổi bật
- Highlight "TOP PICK" với lý do

### 4. So sánh (nếu khách yêu cầu)
- Tạo bảng so sánh: tên, giá, ưu điểm, nhược điểm
- Đưa ra khuyến nghị cuối cùng

### 5. Upsell thông minh
- Tham khảo quy tắc upsell tại references/upsell-rules.md
- Chỉ gợi ý nếu THỰC SỰ phù hợp
- Không push sản phẩm đắt không cần thiết

## Lưu trữ State
- Lưu products tìm thấy vào state.products
- Lưu tất cả sản phẩm đã nhắc vào state.allProductsInContext
- Phục vụ tham chiếu chéo: "lấy cái đầu tiên" → tra lại context

## Edge Cases
- Không tìm thấy → đề xuất tương tự + hỏi mở rộng điều kiện
- Hết hàng → thông báo rõ + đề xuất thay thế
- Khách muốn mua → set returnToSupervisor = true, nextAction = "order"

## Tone & Style
- Tiếng Việt, thân thiện, tự nhiên
- Gọi khách là "bạn" hoặc "anh/chị"
- Mỗi response < 200 từ (trừ so sánh chi tiết)
```

#### Skill 3: `order-management`

```yaml
---
name: order-management
description: >
  Quản lý đặt hàng, thu thập thông tin giao hàng, tương tác
  giỏ hàng, tạo đơn hàng. Sử dụng khi khách muốn mua hàng,
  thêm vào giỏ, hoặc kiểm tra đơn hàng đã đặt.
allowed-tools: Read Write
metadata:
  author: ecommerce-team
  version: "1.0"
---

# Order Management

## Vai trò
Bạn là nhân viên hỗ trợ đặt hàng. Nhiệm vụ: thu thập đầy đủ
thông tin, xác nhận, và tạo đơn hàng chính xác.

## Quy trình đặt hàng (Wizard-style)

### Bước 1: Xác nhận sản phẩm
- Kiểm tra state.allProductsInContext nếu khách tham chiếu
  (ví dụ "lấy cái đầu tiên")
- Xác nhận: tên SP, variant (size/màu), số lượng
- PHẢI có confirmation từ khách trước khi tiếp tục

### Bước 2: Thu thập thông tin giao hàng
Cần thu thập đủ:
- Tên người nhận
- Số điện thoại (validate format VN: 10 số, bắt đầu 0)
- Địa chỉ giao hàng (đầy đủ: số nhà, đường, phường/xã,
  quận/huyện, tỉnh/thành phố)

Kỹ thuật thu thập:
- Hỏi TỪNG thông tin, KHÔNG hỏi tất cả một lúc
- Nếu khách cung cấp nhiều thông tin cùng lúc → accept tất cả
- Validate ngay khi nhận → hỏi lại nếu sai format

### Bước 3: Xác nhận đơn hàng
- Tổng hợp: SP, SL, giá, tổng tiền, thông tin ship
- Yêu cầu khách xác nhận TOÀN BỘ
- Chỉ tạo đơn khi khách nói "OK/Đồng ý/Xác nhận"

### Bước 4: Tạo đơn hàng
- Gọi create_order tool với data đã validate
- Trả mã đơn hàng cho khách
- Hỏi: "Bạn cần tư vấn thêm gì không?"
  → Nếu có: set returnToSupervisor = true

## Lưu trữ State
- Cập nhật state.orderData ở MỖI bước
- Đảm bảo state được persist qua LangGraph checkpointer

## Xử lý từ chối
- Khách muốn huỷ → xác nhận, reset orderData
- Khách muốn sửa → quay lại bước tương ứng
- Khách muốn thêm SP → set returnToSupervisor = true,
  nextAction = "sales"

## Shipping Rules
Tham khảo: references/shipping-rules.md
```

### 8.3. Cấu trúc thư mục hoàn chỉnh

```
ecommerce-agent/
├── src/
│   ├── graph/                    # LangGraph definitions
│   │   ├── supervisor.ts
│   │   ├── sales.ts
│   │   └── order.ts
│   ├── tools/                    # Tool implementations
│   │   ├── search_product.ts
│   │   ├── create_order.ts
│   │   └── check_inventory.ts
│   ├── state/                    # State schemas
│   │   └── agent-state.ts
│   └── skills/                   # ★ AGENT SKILLS ★
│       ├── supervisor-routing/
│       │   └── SKILL.md
│       ├── sales-advisor/
│       │   ├── SKILL.md
│       │   ├── scripts/
│       │   │   └── format_comparison.py
│       │   └── references/
│       │       ├── product-categories.md
│       │       └── upsell-rules.md
│       ├── order-management/
│       │   ├── SKILL.md
│       │   ├── scripts/
│       │   │   └── validate_phone.py
│       │   └── references/
│       │       └── shipping-rules.md
│       └── general-support/
│           ├── SKILL.md
│           └── references/
│               └── store-policies.md
├── package.json
└── tsconfig.json
```

---

## 9. So sánh: Skills vs Tools vs MCP vs Prompt Engineering

| Dimension | Agent Skills | Tools | MCP | Prompt Engineering |
|-----------|-------------|-------|-----|-------------------|
| **Bản chất** | Prompt template + context modifier | Executable function | Protocol kết nối | Text instructions |
| **Đóng gói** | Folder (SKILL.md + scripts + refs) | Function definition | Server + client | String trong code |
| **Portability** | Cross-platform (open standard) | Framework-specific | Protocol-agnostic | Copy-paste |
| **Version Control** | Git-friendly (files) | Cần đóng gói code | Server versioning | Thường inline |
| **Progressive Disclosure** | ✅ Built-in 3 phases | ❌ Load tất cả | ❌ (đang develop) | ❌ Load tất cả |
| **Idempotent** | ✅ Scripts + instructions | Phụ thuộc code | Phụ thuộc server | ❌ Non-deterministic |
| **Token Efficiency** | Rất cao (~100 tokens/skill startup) | Vừa phải | Thấp (hàng nghìn tokens) | Thấp |
| **Reusable** | ✅ Write once, use everywhere | Có, nhưng framework-bound | Có, qua protocol | Hạn chế |
| **Shareable** | ✅ Folder → share/publish | Cần package | Cần deploy server | Copy-paste |

### 9.1. Khi nào dùng gì?

```
"Tôi muốn agent BIẾT CÁCH tư vấn sản phẩm chuyên nghiệp"
→ Agent Skills (procedural knowledge)

"Tôi muốn agent CÓ THỂ gọi API tìm kiếm sản phẩm"
→ Tools (executable capability)

"Tôi muốn agent KẾT NỐI ĐƯỢC đến database sản phẩm"
→ MCP (data connectivity)

"Tôi muốn agent TRẢ LỜI theo tone thân thiện"
→ Prompt Engineering (behavioral guidance)
```

**Chúng bổ trợ nhau, KHÔNG thay thế nhau.**

---

## 10. Implementation Blueprint — Kế hoạch triển khai

### Phase 1: Tách Prompt thành Skills (1-2 ngày)

Bước đầu tiên, không cần thay đổi kiến trúc. Chỉ cần di chuyển prompt hiện tại thành SKILL.md files.

```
Trước:
  supervisor_prompt = "Bạn là supervisor, phân loại yêu cầu..."
  sales_prompt = "Bạn là tư vấn viên bán hàng..."
  order_prompt = "Bạn là nhân viên hỗ trợ đặt hàng..."

Sau:
  skills/supervisor-routing/SKILL.md → chứa supervisor prompt
  skills/sales-advisor/SKILL.md → chứa sales prompt
  skills/order-management/SKILL.md → chứa order prompt

Lợi ích ngay:
  ✅ Git version control cho prompts
  ✅ Dễ review, diff, rollback
  ✅ Team members có thể edit prompt mà không cần code
```

### Phase 2: Implement Skill Loader (2-3 ngày)

Viết module load skills từ filesystem:

```typescript
// Pseudocode — SkillLoader
class SkillLoader {
  // Phase 1: Discovery — chỉ load metadata
  async discoverSkills(skillsDir: string): SkillMetadata[] {
    const skills = scanDir(skillsDir);
    return skills.map(s => ({
      name: parseFrontmatter(s).name,
      description: parseFrontmatter(s).description,
      path: s.path
    }));
  }

  // Phase 2: Activation — load full SKILL.md
  async activateSkill(skillName: string): SkillContent {
    const skill = this.findSkill(skillName);
    return parseMarkdown(readFile(skill.path));
  }

  // Phase 3: Load resources on demand
  async loadReference(skillName: string, refPath: string): string {
    return readFile(`${this.skillDir}/${skillName}/${refPath}`);
  }

  // Generate <available_skills> XML cho system prompt
  generateSkillsPrompt(): string {
    return this.skills.map(s =>
      `<skill>
        <name>${s.name}</name>
        <description>${s.description}</description>
        <location>${s.path}</location>
      </skill>`
    ).join('\n');
  }
}
```

### Phase 3: Integrate vào LangGraph (3-5 ngày)

```typescript
// Trong LangGraph node, inject skill content vào context
async function supervisorNode(state: AgentState) {
  const skillLoader = new SkillLoader('./skills');

  // Load supervisor skill
  const routingSkill = await skillLoader.activateSkill('supervisor-routing');

  // Inject vào prompt
  const messages = [
    { role: 'system', content: routingSkill.content },
    ...state.messages
  ];

  // LLM reasoning với skill context
  const result = await llm.invoke(messages);

  // Parse routing decision
  return { nextAction: result.nextAction };
}
```

### Phase 4: Tinh chỉnh & Mở rộng (Liên tục)

- Test từng skill independently (unit test cho skills)
- Track metrics qua Langfuse: accuracy, token usage, latency
- Improve skills dựa trên real user feedback
- Thêm scripts/ cho các tác vụ cần idempotent
- Tách references/ cho content dài (shipping rules, product categories)
- Version control & CI/CD cho skills changes

---

## Tóm tắt

Agent Skills giải quyết 3 bài toán lớn mà bất kỳ AI Agent developer nào cũng gặp:

1. **Idempotency** — Chuẩn hóa hành vi agent qua instructions + scripts, kết quả nhất quán hơn
2. **Context Window Efficiency** — Progressive Disclosure tiết kiệm đến 86% tokens
3. **Knowledge Portability** — Write once, use everywhere; Git-friendly, team-shareable

Với dự án Ecommerce Agent hiện tại, Agent Skills không yêu cầu phải thay đổi kiến trúc Multi-Agent (Supervisor/Sales/Order). Thay vào đó, Skills bổ trợ bằng cách đóng gói "procedural knowledge" — cách làm — thành các module có thể version control, test riêng, và cải tiến liên tục mà không ảnh hưởng code logic.
