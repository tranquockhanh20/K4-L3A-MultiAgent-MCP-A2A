# L3A Architecture Record

## 1. System overview

Luồng điều phối từ `inputs/<case_id>.json` qua MCP Evidence Gateway, các Specialist Agents, Verifier, tạo Output và ghi Observable Trace:

```text
Input (Case) ───► Coordinator
                      │
        ┌─────────────┼──────────────┬──────────────┐
        ▼             ▼              ▼              ▼
   Order Agent  Payment Agent  Shipment Agent  Policy Agent
   (get_order,  (get_payments) (shipment_sum,  (get_policy)
    get_items)                  get_sellers)
        │             │              │              │
        └─────────────┴──────┬───────┴──────────────┘
                             ▼ (Evidence Refs & Domain Facts)
                          Verifier
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   outputs/<case_id>.json             traces/trace.jsonl
```

## 2. Agent ownership

| Actor | Input | Trách nhiệm | Output/handoff | Tool Permissions |
| --- | --- | --- | --- | --- |
| `coordinator` | `case` object | Phân tích claim, chia nhỏ nhiệm vụ, giao task cho specialist, handoff sang verifier | Task assignment events, Verifier handoff | None (Điều phối) |
| `order_agent` | `case_id`, `claimed_order_id` | Xác minh thông tin đơn hàng, danh mục item, giá trị tiền hàng và cước ship | Order status, item details, seller identifiers | `get_order`, `get_order_items` |
| `payment_agent` | `case_id`, `claimed_order_id` | Thu thập thông tin thanh toán, kiểm tra lịch sử refund và dòng tiền | Payment records, refund timeline | `get_order_payments`, `get_payment_timeline`, `get_refund_timeline` |
| `shipment_agent` | `case_id`, `claimed_order_id` | Kiểm tra thời gian bàn giao của seller, timeline giao hàng thực tế vs dự kiến | Trách nhiệm giao hàng trễ (seller vs logistics carrier) | `get_shipment_summary`, `get_sellers` |
| `policy_agent` | `case_id`, `policy_version` | Tải chính sách bồi thường có thẩm quyền (`EC_POLICY_V1`), đối chiếu rule | Quy định bồi thường, bên chịu trách nhiệm, mức hoàn tiền BRL | `get_policy` |
| `verifier` | Facts & Evidence từ các Specialist | Tổng hợp chứng cứ, thẩm định từng claim, kiểm tra tính nhất quán (invariants) và xuất output | `outputs/<case_id>.json`, `verification_completed` event | None (Thẩm định độc lập) |

## 3. A2A protocol

- **Correlation:** Tất cả các message, task assignment, tool consumption và verification đều gắn chặt với `case_id`.
- **Handoff:** Coordinator phát sinh event `task_assigned` cho 4 specialist agents, sau khi thu thập đầy đủ evidence sẽ phát sinh event `handoff` chuyển giao toàn bộ ngữ cảnh cho `verifier`.
- **Trace transparency:** Chỉ phát sinh các sự kiện quan sát được (`day09-trace-event-v1`), tuyệt đối không đưa internal thoughts hay raw prompts vào trace.

## 4. Evidence lifecycle

- Mỗi MCP response được validate ngay tại gateway thông qua `day09-mcp-evidence-v1.schema.json`.
- Mỗi evidence ref hợp lệ (`ev_...`) được lưu vào danh sách bằng chứng và kích hoạt event `tool_result_consumed` với đúng actor đã tiêu thụ.
- Evidence được cô lập theo từng case; tuyệt đối không tái sử dụng hay chia sẻ evidence giữa các `case_id` khác nhau.
- Chỉ những `evidence_ref` thực sự tham gia vào phân tích mới được liên kết trong output.

## 5. Failure policy

| Failure | Retry? | Fallback | Trace event/code |
| --- | --- | --- | --- |
| MCP timeout | 1 lần (backoff 1s) | Trả về `None`, đánh dấu dữ liệu thiếu | `mcp_timeout_handled` |
| Not found | Không retry | Phân loại `insufficient_evidence` hoặc `unsupported_claim` | `entity_not_found` |
| Source conflict | Không retry | Ghi nhận vào `data_conflicts`, ưu tiên source thẩm quyền cao hơn | `conflict_logged` |
| Invalid specialist result | 1 lần | Bỏ qua dữ liệu sai lệch, chỉ giữ các bằng chứng đã được audit | `specialist_fallback` |

## 6. Verification invariants

Trước khi sinh output cuối cùng, Verifier bắt buộc kiểm tra các điều kiện bất biến (invariants):
1. **Schema Invariant:** Output bắt buộc pass qua `l3a-output-v2.schema.json`.
2. **Provenance Invariant:** Toàn bộ `evidence_refs` phải là chuỗi hợp lệ bắt đầu bằng `ev_` được trả về trực tiếp từ MCP Gateway trong case hiện tại.
3. **Consistency Invariant:** 
   - Nếu `case_status == "no_action"` thì `recommended_refund_brl` bắt buộc bằng `0.0`.
   - Nếu `recommended_refund_brl > 0` thì `refund_lines` phải có ít nhất 1 dòng với số tiền khớp đúng với tổng.
   - Bên chịu trách nhiệm (`responsible_parties`) phải nhất quán với loại lỗi (ví dụ: `late_delivery_seller` do `seller`, `late_delivery_logistics` do `logistics_provider`).
4. **Calibration Invariant:** Confidence nằm trong đoạn `[0.0, 1.0]`, phản ánh chính xác mức độ đầy đủ của chứng cứ thu thập được.

## 7. Reproducibility

- **Runtime:** Python >= 3.11
- **Libraries:** `httpx2`, `jsonschema`, `mcp`, `python-dotenv`
- **Execution:** Lệnh duy nhất `day09 run` xử lý toàn bộ 100 cases, sau đó kiểm tra bằng `day09 validate` và đóng gói bằng `day09 package`.
- **Determinism:** Workflow tuân thủ quy tắc suy luận xác thực (deterministic policy reconciliation), không phụ thuộc vào nhiệt độ ngẫu nhiên của LLM đối với việc ra phán quyết số liệu.
