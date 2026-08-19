# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Nguyễn Thùy Trang
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026 

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** Trong chunk `56de0db4b3758fddcc82::c0000` (bài "Countries agree to extend digital services tax freeze through 2024"), câu *"With the exception of Canada countries with digital services taxes have agreed to hold off applying them for at least another year"* — đại từ *"them"* và *"they"* xuất hiện nhiều lần trong context đa quốc gia. Nếu LLM resolver chọn sai antecedent (gán *"them"* = "countries" thay vì "digital services taxes"), sẽ sinh ra triple sai.
- **Hiện tượng:** Đại từ *"they"* trong ngữ cảnh nhiều chủ thể (nhiều quốc gia cùng lúc áp dụng thuế) rất mơ hồ. Pipeline conservative chỉ resolve khi antecedent rõ ràng trong cùng chunk; nếu ambiguous → giữ nguyên và log vào `unresolved_mentions`. Trong thực nghiệm, 400 chunk được xử lý (80 batch × 5 chunk), phần lớn `unresolved_mentions` trả về rỗng `[]` vì phần lớn chunk tin tức không chứa đại từ nhân xưng phức tạp.
- **Hậu quả đối với Graph:** Nếu false coreference xảy ra, triple extractor sẽ tạo ra **False Edge** — gán nhầm quan hệ cho sai thực thể. Ví dụ:搞错 "Microsoft acquired Activision" thành "Sony acquired Activision" chỉ vì pronoun resolution sai → tri thức sai lan truyền trong toàn bộ graph traversal khi trả lời multi-hop.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao (> 0.85) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `ENTITY_VECTOR_THRESHOLD = 0.90`. Chỉ các cặp entity có embedding cosine ≥ 0.90 mới được đưa vào danh sách ứng viên merge. Embedding chỉ tạo candidate; quyết định merge cuối cùng còn phụ thuộc vào **type-aware lexical guard**:
  - **Company:** Cho phép khác suffix pháp nhân (Inc, Corp, LLC) — SequenceMatcher ratio ≥ 0.72 trên tên đã chuẩn hóa.
  - **Person:** Yêu cầu first name + last name phù hợp, tránh merge người trùng họ (ví dụ: Sam Altman ≠ Steve Altman).
  - **Technology:** Guard chặt nhất để tránh nhập nhầm sản phẩm mang tên công ty.
- **Cặp thực thể bị Guard chặn:** Trong dataset HackerNoon hiện tại (subset 1500 bài), chưa xuất hiện cặp REJECT_GUARD cụ thể nào cần chặn (không có cặp entity vector > 0.85 bị lexical guard từ chối trong sample này). Tuy nhiên, cơ chế guard sẵn sàng xử lý các trường hợp như: **"Apple"** (Company) vs **"Apple Watch"** (Technology) — dù vector similarity có thể cao (cùng ngữ cảnh "Apple"), lexical guard sẽ reject vì type mismatch.
- **Lý do chặn:** Lexical guard hoạt động trên nguyên tắc **type-aware difflib.SequenceMatcher** — Company cho phép linh hoạt suffix, Person cần match chính xác tên riêng, Technology cần match chặt. Ngay cả khi embedding model cho rằng hai entity "gần nhau" về ngữ nghĩa, guard sẽ từ chối nếu type label không tương thích hoặc tên chuẩn hóa có quá nhiều khác biệt.

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy N cạnh (N=50) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | Apple | Company | 10 |
| 2 | Databricks Lakehouse for Manufacturing | Technology | 10 |
| 3 | Microsoft | Company | 7 |

*(Lưu ý: Trong subset 1500 bài với 157 triples, max degree chỉ đạt 10. Trong dataset full 350MB sẽ xuất hiện super-nodes có degree > 100.)*

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Giới hạn 50 cạnh mới nhất (`ORDER BY published_date DESC LIMIT 50`) giúp **giảm thiểu bùng nổ context/token** khi answer generator nhận subgraph. Câu trả lời tập trung vào thông tin cập nhật nhất, phù hợp với bản tin công nghệ thay đổi nhanh. Tổng edge bị chặn bởi `GLOBAL_EDGE_CAP = 250`, giới hạn context text ≤ `MAX_GRAPH_CONTEXT_CHARS = 14000`.
  - *Rủi ro:* Nếu câu hỏi liên quan đến **sự kiện lịch sử trong quá khứ xa** (ví dụ: "Microsoft mua Activision năm 2023"), cạnh có `published_date` cũ hơn có thể bị cắt mất nếu super-node đã đủ 50 cạnh mới nhất. Điều này đặc biệt nguy hiểm cho câu hỏi `cross-doc` đòi hỏi dòng thời gian. Giải pháp: provenance (`source_chunk_id`, `published_date`, `evidence`) luôn được giữ lại để audit và debugging.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch (Δ) | Nhận xét phân tích |
|-------------------|----------|----------|---------------------|-------------------|
| **Comprehensiveness (1–5)** | 1.00 | 3.00 (cross-doc); 1.00 (factoid, multi-hop) | +2.0 (cross-doc) | GraphRAG cải thiện rõ ở câu cross-doc; factoid/multi-hop tie |
| **Faithfulness (1–5)** | 1.00 | 3.00 (cross-doc); 1.00 (factoid, multi-hop) | +2.0 (cross-doc) | Graph traversal giúp tìm đúng provenance cho cross-doc |
| **Multi-hop Reasoning (1–5)** | 1.00 | 3.00 (cross-doc); 1.00 (factoid, multi-hop) | +2.0 (cross-doc) | GraphRAG kết nối nhiều chunk qua entity trung gian tốt hơn |
| **Latency trung bình (s)** | 3.764 | 6.293 | +2.53s | GraphRAG chậm hơn ~67% do traversal + context assembly |
| **Token usage trung bình** | 1319.4 | 1879.8 | +560.4 tokens | GraphRAG tốn thêm ~42% token do graph context phong phú hơn |

**Chi tiết theo loại câu hỏi:**
- **Cross-doc:** Flat RAG = 4.525s / 1491.5 tokens; GraphRAG = 10.433s / 3093.5 tokens
- **Factoid:** Flat RAG = 2.054s / 985 tokens; GraphRAG = 1.948s / 779 tokens (GraphRAG thậm chí nhanh hơn!)
- **Multi-hop:** Flat RAG = 3.859s / 1314.5 tokens; GraphRAG = 4.325s / 1216.5 tokens

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi GraphRAG thành công / Flat RAG thất bại (G05 — cross-doc):**
   - *Question ID & Câu hỏi:* G05 — *"The corpus mentions the relation Microsoft -ACQUIRED-> Activision Blizzard in multiple chunks. What evidence appears in two separate chunks, and how do their dates compare?"*
   - *Flat RAG:* Chỉ tìm được 2 chunk gần nhất, HEBA 1/5. Flat RAG dùng chunk `7952d542d69db95025ea` (2023-04-26) và `73b2f13fa96b7e5812f7` (2023-06-23) — bỏ sót chunk gốc `344312edbb3c92e1e2c8` (2023-01-27).
   - *GraphRAG:* Traversal từ node **Microsoft** phát hiện cả 4 chunk liên quan đến relation `ACQUIRED → Activision Blizzard`, trích xuất đầy đủ evidence với đúng `source_chunk_id` và `published_date`. GraphRAG Score = 5/5.
   - *Tại sao Flat RAG thất bại?* Vector search chỉ lấy top-k chunks tương đồng ngữ nghĩa, không biết các chunks này liên kết qua cùng một entity-trung-tâm (Microsoft). Khoảng cách semantic giữa chunk "proposal" (01-27) và chunk "UK regulatory appeal" (04-26) đủ xa để FAISS chỉ ưu tiên chunk gần nhất.
   - *GraphRAG đã giải quyết như thế nào?* BFS từ Microsoft node → ACM edge → 4 chunks chứa bằng chứng → linearize thành context có cấu trúc kèm provenance.

2. **Ca lỗi cả hai đều sai (G04 — multi-hop):**
   - *Question ID & Câu hỏi:* G04 — *"Which shared entity is connected to both Gatzby and Databricks Lakehouse for Manufacturing, and what are the two relation types?"*
   - *Reference Answer:* Shared entity = **AI**; Gatzby -USES-> AI; Databricks -USES-> AI.
   - *Cả hai đều trả lời sai:* Cả Flat RAG và GraphRAG đều xác định "partnership" là shared entity thay vì **AI**. LLM answer generator nhầm lẫn giữa quan hệ business (partnership article) và quan hệ graph (USES → AI).
   - *Nguyên nhân:* Chunk `2d393a1bac01cb453b77` chứa bài báo nói về partnership giữa Gatzby và Databricks, nhưng trong Knowledge Graph thì cả hai company đều có edge `USES → AI`. LLM ưu tiên thông tin partnership nổi bật hơn trong context.
   - *Đề xuất khắc phục:* (1) Cluster GraphRAG context rõ ràng hơn: tách `[ENTITY RELATIONS]` và `[CO-MENTION]` thành các section riêng. (2) Thêm ranking theo relation type khi linearize subgraph — ưu tiên `USES`/`DEVELOPED` thay vì `PARTNERED_WITH` cho shared entity queries.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 

*Trả lời:*

- **Đánh đổi Quality vs Cost vs Latency:**
  - **GraphRAG:** Chi phí cao hơn (traversal + context assembly + Super-node mitigation logic), latency trung bình 6.293s, token usage 1879.8. Đổi lại, cải thiện đáng kể ở câu `cross-doc` (score +2 điểm). Cost effectiveness thấp hơn với câu `factoid` đơn giản.
  - **Flat RAG:** Nhanh và rẻ hơn (latency 3.764s, token 1319.4), phù hợp cho câu `factoid` đơn giản. Nhưng thất bại rõ rệt ở câu `cross-doc` cần kết nối nhiều chunks.
  - **Kết luận:** Nên dùng **Hybrid approach** — route `factoid` queries qua Flat RAG, route `cross-doc`/`multi-hop` qua GraphRAG. Chi phí indexing ban đầu của GraphRAG cao hơn (NER+RE extraction ~400 chunks × LLM call), nhưng amortized trên nhiều queries thì worth it.

- **Quyết định từ chối AI Coding Agent:** 
  - **Từ chối pairwise cosine O(N²) cho Near-Dedup:** AI Agent đề xuất tính cosine similarity trên toàn bộ 1500 articles × 1500 articles để tìm near-duplicates. Từ chối vì gây tràn RAM/OOM trên Colab T4. Thay vào đó, dùng **MinHash/LSH** (128 permutations, shingle size=5, LSH threshold=0.80, verify Jaccard ≥ 0.88) — sub-quadratic, đã chứng minh chạy thành công trên 1500 articles (0 duplicates removed, 1 REJECT_VERIFY candidate với Jaccard=0.80 < 0.88 threshold).

- **Giải pháp scale 350MB (~100,000 bài báo):**
  - *Bottleneck đầu tiên:* **LLM throughput** cho Coreference Resolution + NER/RE Extraction. Với 350MB (~100K articles → ~300K chunks → 400K+ coreference calls), chi phí API sẽ rất lớn.
  - *Giải pháp:* (1) **Streaming batch extraction** với worker queue + checkpoint/resume (đã có `graphrag_eval_checkpoint.csv`). (2) **Block sampling** — chỉ extract 5-10% samples qua LLM, phần còn lại dùng rule-based extraction. (3) **HNSW index** (thay FlatIP) cho Entity Resolution với blocking strategy để giảm pairwise comparisons. (4) **Community Partitioning** — trước khi回答, detect communities trong graph, ưu tiên answer từ community chứa seed entity thay vì BFS toàn graph.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()`, `run_coref()` | Pipeline xử lý 400 chunks (80 batch × 5), conservative rule hoạt động đúng — phần lớn unresolved_mentions rỗng vì tin tức ít chứa đại từ phức tạp. Nếu dataset giàu dialog/narrative sẽ cần xử lý nhiều hơn. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | 8 relation types được define rõ ràng. Guard hoạt động trong NER extraction prompt — LLM chỉ được phép trả về relation nằm trong allowlist. Reality check: 157 triples extract được, 3 batch fails cần retry logic. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Dùng `UNWIND $rows AS row` với batch size. Sanity check: chạy Cypher query kiểm tra `invalid_provenance_edges == 0`. Constraint `entity_id` unique + index `name_norm` đảm bảo performance query. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `DedupUF` | Vector threshold = 0.90, SequenceMatcher ≥ 0.72. Audit table ghi rõ lý do merge (MERGE_MANUAL, MERGE_VECTOR) hoặc reject (REJECT_GUARD). False merge prevention hoạt động qua type-aware guard. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()` | Degree > 100 → lấy 50 edges mới nhất. GLOBAL_EDGE_CAP = 250. Trong subset nhỏ, max degree chỉ 10 (Apple, Databricks). Cần chạy trên full dataset để test thực sự super-node mitigation. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()` | 5 golden questions, 3 groups (factoid/multi-hop/cross-doc). Judge score 1-5 trên 3 criteria. Cross-doc là nơi GraphRAG tỏa sáng nhất (+2 điểm comprehensiveness). Factoid/multi-hop tie ở subset nhỏ. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:** OpenRouter API timeout/retry khi gửi batch coreference resolution 80 lần liên tiếp. Một số model trên OpenRouter không hỗ trợ `response_format: json_object`, gây crash khi parse JSON. LLM trả về markdown code block thay vì pure JSON cần regex cleanup.
- **Cách bạn đã xử lý thành công:** (1) Implement retry mechanism với exponential backoff (max 4 retries, sleep 2^n seconds). (2) Fallback logic: nếu `response_format: json_object` không được hỗ trợ → retry không có `response_format`但仍要求 JSON trong prompt. (3) `parse_json_object()` robust: strip markdown code block, tìm `{` và `}` đầu/cuối, `json.loads()` an toàn. (4) Nếu batch coreference fail hoàn toàn → fallback giữ nguyên text gốc + log `COREF_BATCH_FAILED`.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** [Đồ án sẽ được xác định sau khi thảo luận với giảng viên]
- **Đặc thù bài toán & Lý do chọn giải pháp:** GraphRAG đặc biệt phù hợp với bài toán cần **multi-hop reasoning** trên dữ liệu có cấu trúc liên kết mạnh (ví dụ: hệ thống hỗ trợ quyết định y tế, phân tích chuỗi cung ứng, hoặc hệ thống hỏi đáp pháp luật). Nếu bài toán chỉ cần tìm kiếm document đơn lẻ → Flat RAG là đủ và tiết kiệm hơn.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `Company`, `Person`, `Technology`, `Product`, `Event`
  - Relations: `ACQUIRED`, `DEVELOPED`, `INVESTED_IN`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`, `USES`, `LEADS`, `ANNOUNCED`, `REGULATED_BY`
- **Chiến lược xử lý Super-node & Entity Resolution:**
  - *Super-node:* Áp dụng Degree Cap 50 edges mới nhất +_GLOBAL_EDGE_CAP = 250. Bổ sung **Community Detection** (Louvain/Label Propagation) để answer trên community level thay vì individual nodes.
  - *Entity Resolution:* Kết hợp Manual Alias Map (ticker symbol lookup) + Vector ANN (cosine ≥ 0.90) + Lexical Guard (type-aware). Deploy HNSW index cho scale thay vì FlatIP. Batch entity resolution với checkpoint để resume khi OOM.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | 4 | Hiểu rõ 5 modules, trade-off giữa Flat vs Graph. Cần thực hành thêm trên dataset lớn hơn. |
| Khả năng kiểm soát AI Coding Agent | 4 | Đã từ chối O(N²) near-dedup, điều chỉnh prompt coreference, kiểm soát retry logic. |
| Chất lượng đồ thị tri thức xây dựng | 3 | Subset nhỏ (1500 bài, 157 triples) nên graph chưa đủ dense để chứng minh ưu thế GraphRAG rõ ràng. |
| Khả năng phân tích và debug hệ thống | 4 | Debug được OpenRouter JSON parsing, robust error handling, provenance integrity check. |
