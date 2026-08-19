# Technical Explanation — GraphRAG vs Flat RAG

1. **Coreference sai ở tình huống nào?**
   Sai nguy hiểm nhất khi một đại từ hoặc mô tả chung có từ hai antecedent hợp lý trong cùng chunk.
   Nếu resolver ép chọn một entity, triple extractor có thể tạo false edge. Pipeline này chỉ resolve khi antecedent rõ;
   trường hợp mơ hồ giữ nguyên và log `unresolved_mentions`.

2. **Entity threshold bao nhiêu, vì sao?**
   Embedding threshold hiện tại là **0.90**. Embedding chỉ tạo candidate; quyết định merge còn qua
   type-aware lexical guard. Company cho phép khác suffix pháp nhân, Person yêu cầu first/last name phù hợp,
   Technology dùng guard chặt hơn để tránh nhập nhầm product.

3. **Candidate similarity cao nhưng không nên merge?**
   Không có REJECT_GUARD trong sample hiện tại.

4. **Top 3 super-node và degree?**
   Apple (degree=10); Databricks Lakehouse for Manufacturing (degree=10); Microsoft (degree=7)

5. **Vì sao ưu tiên edge mới nhất có thể đúng/sai?**
   Đúng khi câu hỏi cần trạng thái hiện tại hoặc quan hệ đã thay đổi theo thời gian. Sai khi câu hỏi mang tính lịch sử,
   khi `published_date` thiếu/không chuẩn, hoặc khi bài mới chỉ nhắc lại tin cũ. Vì vậy provenance luôn được giữ để audit.

6–7. **Flat RAG thắng nhóm nào / GraphRAG thắng nhóm nào?**
- cross-doc: GraphRAG (Flat=1.00, Graph=3.00)
- factoid: Tie (Flat=1.00, Graph=1.00)
- multi-hop: Tie (Flat=1.00, Graph=1.00)

8. **Latency/token trade-off?**
   Mean latency: Flat=3.764s, Graph=6.293s. Mean tokens: Flat=1319.4, Graph=1879.8. GraphRAG thường tốn thêm traversal + context assembly, đổi lại có thể tăng coverage cho câu multi-hop.

9. **AI Coding Agent đề xuất gì mà không dùng, vì sao?**
   Không dùng pairwise cosine toàn bộ document/entity (`O(N²)`) cho near-dedup. Thay vào đó dùng MinHash/LSH để sinh
   candidate rồi exact-Jaccard verify, giúp scale tốt hơn và vẫn có audit table cho false positives.

10. **Scale 350MB: bottleneck đầu tiên là gì?**
    Bottleneck thực tế thường là throughput/cost của coreference + triple extraction qua LLM trước khi Neo4j trở thành
    vấn đề. Cách scale: streaming input, exact/near dedup trước LLM, batching, checkpoint/resume, giới hạn extraction sample,
    và bulk `UNWIND` khi ghi graph.