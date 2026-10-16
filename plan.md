# Lab28 – VBot Capstone: Kế hoạch nhóm (3 người)

## Tổng quan
- **Đề:** Thiết kế hệ thống cloud cho **VBot**, trợ lý AI nội bộ giống Grok hoặc OpenAI Dots, phục vụ **2.000 DAU**.
- **VBot làm được 5 cấp tác vụ:** (1) email, hỏi đáp · (2) tài liệu, slide · (3) dữ liệu, code trong sandbox · (4) agent đọc/ghi Jira, Confluence, Notion · (5) thiết kế, sinh ảnh.
- **Connectors:** Gmail, Drive, Calendar, M365, Outlook, OneDrive, SharePoint, Teams, Notion, Jira, Confluence. Agent dùng **OAuth bằng quyền của chính user**.
- **Mục tiêu:** tổng chi phí hạ tầng/tháng **thấp nhất** nhưng vẫn đạt đủ 8 ràng buộc H1–H8 và có đủ **19 component bắt buộc**.
- **Nộp (1 Google Form, hạn mốc 03:05):**
  - `G<nn>_baogia.xlsx`: sheet `BaoGia` 5 cột (Component | Ý nghĩa | Ghi chú | Chi phí/tháng (USD) | Bằng chứng), dòng cuối là `TOTAL`, kèm sheet `BangChung` (E01…).
  - `G<nn>_slide.pdf/pptx`: ≤15 slide, ≤10 MB, thuyết trình 7 phút + 3 phút hỏi đáp.
- **Chấm 100 điểm:** Kiến trúc 25 · Báo giá + sizing 20 · Chi phí 20 · Bảo mật 15 · Lý do chọn công nghệ 10 · Trình bày 10.

## Ràng buộc cứng (vi phạm thì điểm chi phí tối đa 10/20)
| | Yêu cầu |
|---|---|
| H1 | 1 cloud chính, nêu region và lý do. Gọi API model qua private endpoint hoặc kênh cam kết không lưu dữ liệu |
| H2 | SSO (OIDC/SAML) + MFA, RBAC/ABAC, ACL theo quyền gốc của tài liệu, OAuth scope tối thiểu, token cất trong vault mã hóa |
| H3 | Tách workspace, bộ nhớ, kết quả search theo từng user. Sandbox gVisor/Firecracker/Kata: không egress, giới hạn tài nguyên, hủy sau khi chạy |
| H4 | Guardrail vào/ra, chống prompt injection, user xác nhận trước hành động ghi, có eval gate khi đổi model |
| H5 | Mã hóa khi lưu và khi truyền, DLP che PII, phân loại dữ liệu, audit log 12 tháng, nhà cung cấp không train trên dữ liệu |
| H6 | p95 thời gian ra token đầu ≤3s giờ cao điểm. Agent nền xong ≤10 phút, có thông báo |
| H7 | 99,5% uptime giờ hành chính, không có điểm lỗi đơn ở LB/gateway/DB, RTO 4h, RPO 1h, backup có test khôi phục |
| H8 | Mỗi cấp tác vụ đạt ≥95% điểm model tham chiếu (có cách đo). Quota theo người và đơn vị, cảnh báo ngân sách |

## File báo giá chung: `baogia_mau_team.xlsx`
🔗 **Google Sheet chung của team (mở và điền trực tiếp ở đây):** https://docs.google.com/spreadsheets/d/1wU9EoVmSsZzDhNj1BP4q41In6ZHEf3m1/edit?usp=sharing&ouid=101399864323228304315&rtpof=true&sd=true

| Sheet | Dùng để | Ai điền |
|---|---|---|
| **HuongDan** | Quy trình, quy tắc, bảng gợi ý dịch vụ AWS/GCP/Azure cho từng component. **Điền tên 3 người** vào ô vàng | Cả nhóm đọc trước |
| **Sizing** | Giả định tải → token → tiền LLM. Ô vàng = giả định (sửa được), ô xám = công thức (đừng sửa) | B dẫn, cả nhóm chốt |
| **BaoGia** (nộp) | 34 dòng chi phí, **tô màu theo người** (xanh dương = A, xanh lá = B, cam = C) + dòng TOTAL tự cộng. Rê chuột vào ô Ghi chú để xem gợi ý cách ghi | Mỗi người điền dòng màu mình |
| **BangChung** (nộp) | Mã ảnh E.., mô tả, link, ngày chụp, dán ảnh vào cột E | Mỗi người dùng dải mã riêng |

**Cách dùng:**
1. ✅ File đã có trên Google Drive (link ở trên). Mở link, kiểm tra mình có quyền **Chỉnh sửa** (nếu chỉ xem được thì nhắn người share cấp quyền Editor).
2. Mỗi dòng ở BaoGia cần điền **3 ô**:
   - **Ghi chú:** dịch vụ/SKU, region, đơn giá × số lượng, giả định.
   - **Chi phí:** chỉ ghi số.
   - **Bằng chứng:** mã E.. hoặc link.
3. Các dòng `llm`, `embedding` và 2 dòng `governance` **đã có công thức lấy từ sheet Sizing**. Chỉ cần điền Ghi chú và Bằng chứng, không gõ đè vào ô Chi phí.
4. Cần thêm dòng thì chèn **phía trên dòng TOTAL**, chọn Component trong dropdown. Không cần dòng nào thì xóa cả dòng.
5. Cuối buổi: Tệp › Tải xuống › .xlsx, kéo vào công cụ kiểm tra trong `vbot-capstone.html`, sửa tới khi **ĐẠT**, rồi đổi tên thành `G<nn>_baogia.xlsx`.

> File mẫu đã được thử với chính công cụ kiểm tra của đề: điền đủ thì ra ĐẠT (0 lỗi). Số trong sheet Sizing chỉ là giả định ban đầu để tranh luận, **không phải đáp án**.

## Phân công chi tiết

### 👤 A – Lead + Cloud/HA/Vận hành · Tên: ______ · Mã ảnh E01–E29
**Thiết kế**
- [ ] Chốt 1 cloud + region (H1), nêu lý do (độ trễ tới VN, giá, có dịch vụ AI qua private endpoint).
- [ ] Vẽ sơ đồ kiến trúc tổng (ghi mã component trên từng hộp) + luồng 1 request: SSO → LB → gateway → model / connector / sandbox → trả về.
- [ ] HA: LB, gateway, DB trải 2 AZ (H7). Backup PITR (RPO 1h), restore ≤4h (RTO), lịch diễn tập.
- [ ] Số FTE vận hành + ca trực.

**Dòng phải điền trong BaoGia (dòng 2–14 và 35)**
| Dòng | Component | Hạng mục | Lấy số từ đâu |
|---|---|---|---|
| 2 | compute | Phí control plane K8s (prod + staging) | 2 cụm × 730h × đơn giá |
| 3 | compute | Node pool app prod (web, API, worker, LLM gateway) | Sizing: RPS cao điểm → số node; 2 AZ; 730h nền + autoscale 220h |
| 4 | compute | Node pool staging | 1–2 node × 220h |
| 5 | load_balancer | LB HTTPS multi-AZ | phí giờ × 730 + phí theo lượng xử lý |
| 6 | network | NAT gateway | 2 NAT × 730h + GB xử lý (Sizing mục 10) |
| 7 | network | Private endpoint tới API model | số endpoint × AZ × 730h |
| 8 | network | WAF + egress internet | phí rule + GB egress (Sizing mục 10) |
| 9 | backup_dr | Backup cross-region | GB backup × đơn giá |
| 10 | monitoring | Log, metric, trace, LLM observability | GB log/tháng (Sizing mục 9) |
| 11 | monitoring | Audit log 12 tháng | GB giữ 12 tháng × lớp lưu trữ lạnh |
| 12 | cicd | Pipeline + registry + IaC | phút build + GB registry |
| 13 | ops_labor | SRE/Platform (FTE) | Bằng chứng = `NA` |
| 14 | licenses | Gói support cloud | % hóa đơn hoặc mức tối thiểu |
| 35 | contingency | Dự phòng 10% tiền LLM (tự tính) | Tùy chọn, xóa nếu không cần · Bằng chứng = `NA` |

**Slide:** Kiến trúc tổng thể · Luồng một request · HA/backup/DR · Monitoring & FinOps · Chi phí (tổng, USD/DAU, top 5) · Rủi ro & DAU ×2.
**Chuẩn bị trả lời:** "Một AZ mất điện lúc 10h?" · "DAU tăng gấp đôi thì hóa đơn thế nào?"

### 👤 B – Model serving + AI governance · Tên: ______ · Mã ảnh E30–E59
**Thiết kế**
- [ ] **Dẫn sizing GĐ1** trong sheet Sizing: request/người/ngày, tỉ lệ 5 cấp (tổng = 100%), token vào/ra mỗi cấp, % chạy sandbox mỗi cấp, cache hit.
- [ ] Chọn model Nhỏ/Lớn cho từng cấp + **model tham chiếu** và cách đo ≥95% (H8).
- [ ] So sánh API, tự host GPU và hybrid: tính điểm hòa vốn (token/s cao điểm ở Sizing mục 5 so với throughput 1 GPU × 730h).
- [ ] Gateway: quota theo người và đơn vị, rate limit, fallback, cảnh báo ngân sách.
- [ ] Governance: guardrail vào/ra, tách nội dung email/tài liệu khỏi chỉ dẫn, xác nhận trước hành động ghi, DLP, eval gate, cam kết không train.

**Điền trong sheet Sizing:** mục 2–4 (giả định), mục 3 (tên model + giá input / cached / output + mã ảnh), giá ảnh, giá embedding, mục 6 (guardrail, eval).

**Dòng phải điền trong BaoGia (dòng 15–23)**
| Dòng | Component | Hạng mục | Lấy số từ đâu |
|---|---|---|---|
| 15 | llm | Model nhỏ cấp 1, 2, 5 | **Công thức sẵn** ← Sizing |
| 16 | llm | Model lớn cấp 3, 4 | **Công thức sẵn** ← Sizing |
| 17 | llm | Sinh ảnh cấp 5 | **Công thức sẵn** ← Sizing |
| 18 | embedding | Embedding tài liệu + câu hỏi | **Công thức sẵn** ← Sizing mục 7 |
| 19 | gateway | LLM gateway | Tự host → `0` + "nằm trong compute", hoặc giá dịch vụ managed |
| 20 | gateway | API gateway backend | triệu request × đơn giá, hoặc "nằm trong" |
| 21 | governance | Guardrail / chống prompt injection | **Công thức sẵn** (model nhỏ), hoặc thay bằng giá dịch vụ managed |
| 22 | governance | DLP + phân loại dữ liệu | GB quét × đơn giá, hoặc tự host "nằm trong compute" |
| 23 | governance | Eval gate | **Công thức sẵn** ← Sizing mục 6 |

**Slide:** Giả định & ước lượng tải · Model và routing · Lý do chọn công nghệ (so sánh API và tự host, có số) · AI governance.
**Chuẩn bị trả lời:** "Email chứa lệnh ẩn gửi hợp đồng ra ngoài?" · "Giá token +30% / −50% thì sao?"

### 👤 C – Data/Workspace + Sandbox + Identity · Tên: ______ · Mã ảnh E60–E89
**Thiết kế**
- [ ] Connectors: gọi API trực tiếp hay sync trước vào index; OAuth ủy nhiệm theo user, scope tối thiểu, token mã hóa trong vault.
- [ ] Cô lập user (H3): prefix + khóa riêng ở object storage, lọc vector search theo ACL gốc và user_id, sandbox chỉ mount workspace của đúng người.
- [ ] Sandbox: gVisor/Firecracker/Kata, không egress, chặn metadata endpoint, giới hạn CPU/RAM/thời gian, hủy sau khi chạy.
- [ ] SSO (OIDC/SAML) + MFA; RBAC/ABAC + ACL tài liệu.

**Điền trong sheet Sizing:** mục 7 (số tài liệu, chunk), mục 8 (thời gian/lượt, vCPU/node → số node sandbox), mục 9 (dung lượng), mục 10 (lệnh gọi connector).

**Dòng phải điền trong BaoGia (dòng 24–34)**
| Dòng | Component | Hạng mục | Lấy số từ đâu |
|---|---|---|---|
| 24 | connectors | Worker connector + webhook | Sizing mục 10. API SaaS không tính phí theo lượt thì ghi "nằm trong compute" (**đừng viết "miễn phí"**) |
| 25 | database | PostgreSQL HA multi-AZ | instance × 2 × 730h + GB |
| 26 | database | DB staging | instance nhỏ |
| 27 | vector_db | Vector index lọc ACL | pgvector → `0` + "nằm trong database", hoặc dịch vụ riêng (Sizing mục 7) |
| 28 | object_storage | Workspace riêng | GB thực dùng (Sizing mục 9) × đơn giá |
| 29 | cache_queue | Redis HA | node × 2 × 730h |
| 30 | cache_queue | Hàng đợi agent | triệu message × đơn giá |
| 31 | sandbox | Node pool sandbox | số node (Sizing mục 8) × giờ × đơn giá |
| 32 | authn | SSO + MFA | IdP sẵn có → `0` + "nằm trong license IdP hiện có" |
| 33 | authz | Policy engine + ACL | OPA tự host → `0` + "nằm trong compute" |
| 34 | secrets_kms | KMS + token vault | số khóa + số secret + lệnh gọi API |

**Slide:** Connectors & phân quyền · Workspace riêng & sandbox · Data governance (mã hóa, thời hạn lưu).
**Chuẩn bị trả lời:** "Agent của A có đọc được file của B không? Chứng minh." · "Code trong sandbox gọi metadata endpoint?"

### 👥 Cả nhóm
- [ ] **25 phút đầu GĐ1 ngồi chung**, chốt 3 thứ: cloud + region, số request và tỉ lệ 5 cấp (sheet Sizing), API hay tự host.
- [ ] Mỗi lần lấy giá: chụp ảnh ngay, dán vào BangChung đúng dải mã, ghi mã vào cột Bằng chứng. Mã không dùng thì xóa.
- [ ] Cuối GĐ2 đọc chéo phần của nhau (hội đồng hỏi bất kỳ ai, bất kỳ mảng nào).
- [ ] Danh sách đòn bẩy tiết kiệm kèm số USD (routing model nhỏ, prompt cache, scale về 0 ngoài giờ, reserved, pgvector thay vector DB riêng…).

## Timeline
| Mốc | Việc |
|---|---|
| 00:25–01:15 | GĐ1 Sizing (B dẫn), chốt hướng LLM |
| 01:25–02:35 | GĐ2: sơ đồ + báo giá song song, **chụp ảnh giá ngay khi lấy** |
| 02:35–03:05 | GĐ3: tối ưu, hoàn thiện slide, công cụ kiểm tra báo ĐẠT, **nộp trước 03:05** |

## ⚠️ Lưu ý khi điền báo giá (công cụ kiểm tra bắt lỗi)
1. **Không viết các chữ** `free tier`, `trial`, `miễn phí`, `dùng thử`, `khuyến mãi`, `credit` ở bất kỳ cột nào, **kể cả trong URL và câu phủ định**.
2. Ô chi phí chỉ ghi số: `1250.50`. Không có `$`, không có dấu phẩy hàng nghìn.
3. Dòng chi phí 0 thì Ghi chú phải có chữ "**nằm trong** <dịch vụ X>", vẫn cần bằng chứng.
4. Ghi chú ≥15 ký tự, ghi rõ SKU, region, đơn giá × số lượng. Ví dụ: `4 node (2/AZ) × 730h × 0.20 USD`.
5. Cột Bằng chứng của `ops_labor` và `contingency` ghi `NA`. Dòng khác ghi link hoặc `E01` (mã phải có trong sheet BangChung).
6. Ảnh bằng chứng phải thấy **tên dịch vụ, SKU, region, đơn giá**. Không dùng free tier, credit hay giá đàm phán.
7. Có nhắc tới **staging** và **HA** (multi-AZ, replica) trong báo giá. Xóa hết dòng `[VÍ DỤ]` của file mẫu.
