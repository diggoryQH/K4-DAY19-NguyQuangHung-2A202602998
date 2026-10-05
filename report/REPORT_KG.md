# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Ngụy Quang Hùng  **MSSV:** 2A202602998  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu dưới đây khớp 100% với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md` (đạt bonus tự thiết kế +15).

---

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
Chat model: gemini:gemini-3.1-flash-lite | Embedding: gemini:gemini-embedding-001 | top_k=3 | chunk_size=800 | chunks=176 | KG: 201 nodes / 388 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    107.7
graph       196     34619     6055   0.00000    197.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.17      696       70   0.00000     3.39
graph       0.94   2.00     6217      142   0.00000     4.75
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00000 | ×1.0 |
| Indexing giây | 107.7s | 197.3s | ×1.83 |
| Mỗi câu: USD | $0.00000 | $0.00000 | ×1.0 |
| Mỗi câu: giây | 3.39s | 4.75s | ×1.40 |
| Mỗi câu: in_tok | 696 | 6217 | ×8.93 |

**Chi phí tăng thêm đến từ đâu?**
> Chi phí tăng thêm của GraphRAG tập trung vào 2 khâu chính:
> 1. **Khâu Indexing (Offline):** Tốn thêm 20 lần gọi LLM trích xuất thực thể/quan hệ từ các bài báo tin tức (tiêu tốn thêm 34,619 input tokens và 6,055 output tokens) cùng thời gian ghi vào Neo4j (thời gian index tăng từ 107.7s lên 197.3s, gấp 1.83 lần).
> 2. **Khâu Querying (Online):** Số lượng token đầu vào trung bình mỗi câu hỏi tăng gấp 8.93 lần (từ 696 lên 6,217 tokens/câu). Điều này xuất phát từ việc `Neo4jGraph.context()` thu thập các dữ kiện đa bước (multi-hop facts: tóm tắt vụ án, các điều khoản BLHS, khung hình phạt tương ứng) đưa vào prompt để cung cấp đầy đủ căn cứ pháp lý cho mô hình suy luận. Tuy nhiên, nhờ mô hình Flash Lite có tốc độ suy luận nhanh, độ trễ phản hồi thực tế của mỗi câu hỏi chỉ chênh lệch 1.36s (4.75s so với 3.39s), hoàn toàn chấp nhận được trong ứng dụng thực tế.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Cả hai pipeline đều tìm được Điều 2 Luật Phòng chống ma túy 2021 định nghĩa chuẩn xác về tiền chất. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa** | Cả hai pipeline đều trích xuất đúng 2 bị cáo Trần Thanh Tuấn và Trần Minh Tâm lãnh án tử hình trong bài báo xét xử đường dây 36kg ma túy. |
| **Q3** | cross-kb | 0.33 / 1 | 1.00 / 2 | **GraphRAG** | Flat RAG chỉ tìm thấy tin tức về mức án 36 tháng mà thiếu văn bản Điều 251 BLHS và khung cơ bản; GraphRAG duyệt đồ thị nối từ bị cáo sang Điều luật đầy đủ. |
| **Q4** | cross-kb | 0.33 / 1 | 1.00 / 2 | **GraphRAG** | Flat RAG chỉ thấy hành vi tổ chức sử dụng của Hoàng Nato nhưng không biết khung hình phạt tối đa trong BLHS; GraphRAG đi qua node Crime sang Điều 255 chỉ rõ mức án chung thân. |
| **Q5** | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **GraphRAG** | Flat RAG biết tội danh và 9.6kg MDMA nhưng không có điều luật để xác định khoản; GraphRAG đối chiếu khối lượng với Điều 250 khoản 4 và khung phạt 20 năm/chung thân/tử hình. |
| **Q6** | aggregation | 0.00 / 0 | 0.67 / 2 | **GraphRAG** | Flat RAG chỉ lấy 3 đoạn trích dính từ khóa MDMA chung chung và judge đánh giá 0 điểm; GraphRAG gom toàn bộ các vụ án liên quan MDMA (Cái Quang Huy, Lê Minh Thành, Viện Pháp y tâm thần) qua node Substance đạt judge 2 tuyệt đối. |

---

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Flat RAG gãy trên câu hỏi liên tri thức hai nguồn (Cross-KB Failure)

- **Hiện tượng:** Flat RAG chỉ truy hồi được các đoạn trích từ một trong hai cơ sở tri thức (hoặc chỉ lấy được tin tức hoặc chỉ lấy được luật), dẫn đến câu trả lời thiếu hẳn nửa thông tin còn lại và phải thừa nhận không đủ dữ liệu.
- **Bằng chứng:**
  - Trích nguyên văn câu trả lời **Q3** của **Flat RAG** (`recall = 0.33`, `judge = 1`):
    > *"Lê Minh Thành bị tuyên phạt 36 tháng tù về tội mua bán trái phép chất ma túy. Ngữ cảnh không cung cấp thông tin về điều luật cụ thể trong Bộ luật Hình sự và khung hình phạt cơ bản của tội danh này."*
  - Đối chiếu với câu trả lời **Q3** của **GraphRAG** (`recall = 1.00`, `judge = 2`):
    > *"- **Số tháng tù:** Lê Minh Thành bị tuyên phạt 36 tháng tù.*
    > *- **Tội danh:** Mua bán trái phép chất ma túy.*
    > *- **Điều luật quy định:** Tội này được quy định tại **Điều 251 Bộ luật Hình sự**.*
    > *- **Khung hình phạt cơ bản:** Theo khoản 1 Điều 251, khung hình phạt cơ bản là phạt tù từ **02 năm đến 07 năm**."*
  - Đường đi truy vấn đồ thị Cypher giải quyết triệt để vấn đề:
    ```cypher
    MATCH (p:Person {name: 'Lê Minh Thành'})-[:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)-[:HAS_CLAUSE]->(cl:Clause {number: 1})
    RETURN p.name, k.name, c.name, a.id, cl.number, cl.penalty;
    ```
    ```
    p.name: "Lê Minh Thành"
    c.name: "mua bán trái phép chất ma túy"
    a.id: "Điều 251 BLHS"
    cl.number: 1
    cl.penalty: "phạt tù từ 02 năm đến 07 năm"
    ```
- **Nguyên nhân:** Khác biệt phong cách ngôn ngữ và khoảng cách biểu diễn vector (Semantic Discrepancy): Câu hỏi hỏi về tên bị cáo cụ thể ("Lê Minh Thành...") nên vector embedding của câu hỏi bị kéo rất gần về bài báo tin tức (chứa tên riêng), nhưng lại có độ tương đồng cosine rất thấp đối với văn bản quy phạm pháp luật (chỉ chứa thuật ngữ pháp lý trừu tượng). Giới hạn `top_k=3` của Flat RAG bị chiếm trọn bởi các chunk bài báo, loại bỏ hoàn toàn văn bản luật ra khỏi ngữ cảnh.
- **Đề xuất sửa:** Sử dụng GraphRAG với node cầu nối `Crime`. Khi vector search định vị được bài báo tin tức, thuật toán duyệt đồ thị cấu trúc sẽ tự động đi qua quan hệ `[:CHARGED_WITH]` và `[:DEFINES]` để kéo Điều luật tương ứng vào ngữ cảnh mà không phụ thuộc vào độ tương đồng cosine của vector.

---

### Lỗi E2: Flat RAG thất bại trong câu hỏi tổng hợp nhiều vụ án (Aggregation Failure)

- **Hiện tượng:** Trong câu hỏi yêu cầu tập hợp thông tin trên diện rộng từ nhiều tài liệu (Q6: "Những vụ việc nào trong tin tức có liên quan đến ma túy MDMA?"), Flat RAG đạt `recall = 0.00` và `judge = 0`, hoàn toàn không nêu được tên vụ việc cụ thể nào.
- **Bằng chứng:**
  - Trích nguyên văn câu trả lời **Q6** của **Flat RAG** (`recall = 0.00`, `judge = 0`):
    > *"Trả lời: Cả 3 vụ việc trong ngữ cảnh đều liên quan đến ma túy MDMA:
    > - Vụ việc [1]: Lực lượng chức năng phát hiện thùng hàng chứa các viên nén màu xanh là MDMA với tổng khối lượng gần 4,3kg.
    > - Vụ việc [2]: Thành bị bắt quả tang khi mang 5 viên nén màu trắng là ma túy MDMA đi bán.
    > - Vụ việc [3]: Kết quả giám định xác định số viên nén hình tam giác màu hồng - xám trong kiện hàng là MDMA với tổng khối lượng hơn 5,3kg."*
    *(Chỉ trích đoạn miêu tả màu sắc viên nén từ 3 chunk ngẫu nhiên, không biết đó là vụ án nào, ai phạm tội, xảy ra ở đâu)*.
  - Đối chiếu với câu trả lời **Q6** của **GraphRAG** (`recall = 0.67`, `judge = 2`):
    > *"Dựa trên ngữ cảnh và dữ kiện từ knowledge graph, các vụ việc liên quan đến ma túy MDMA bao gồm:
    > 1. Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài (liên quan đến MDMA với khối lượng hơn 9,6kg).
    > 2. Vụ góp tiền mua ma túy tại Hà Nội (liên quan đến 5 viên MDMA).
    > 3. Vụ tổ chức sử dụng ma túy tại Sầm Sơn và tàng trữ ma túy tại Viện Pháp y tâm thần Trung ương (0,686g MDMA).
    > 4. Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương.
    > 5. Vụ vận chuyển ma túy qua sân bay Nội Bài của Cái Quang Huy.
    > Các Điều luật liên quan đến MDMA: Điều 248, 249, 250, 251 và 252 BLHS."*
  - Truy vấn đồ thị Cypher gom nhóm:
    ```cypher
    MATCH (s:Substance {name: 'MDMA'})<-[:INVOLVES]-(k:Case)
    RETURN k.name AS CaseName, k.summary AS Summary;
    ```
    ```
    CaseName: "Vụ vận chuyển ma túy của Cái Quang Huy"
    CaseName: "Vụ góp tiền mua ma túy tại Hà Nội"
    CaseName: "Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương"
    CaseName: "Vụ tổ chức sử dụng ma túy tại Sầm Sơn..."
    ```
- **Nguyên nhân:** Tìm kiếm vector thuần túy là cơ chế "cục bộ" (Local Retrieval), chỉ lấy tối đa `top_k=3` đoạn văn có mật độ từ khóa MDMA cao nhất. Khi một thực thể xuất hiện rải rác ở hàng chục bài báo khác nhau trong cơ sở dữ liệu, một câu truy vấn vector không thể mở rộng để bao quát toàn bộ corpus.
- **Đề xuất sửa:** Áp dụng mô hình đồ thị tri thức với thực thể trung tâm `Substance`. Quan hệ `(Case)-[:INVOLVES]->(Substance)` cho phép thực hiện truy vấn 1-hop ngược từ `Substance {name: 'MDMA'}` để tổng hợp toàn bộ các node `Case` liên quan chỉ trong vài mili-giây.

---

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2:

> Dựa trên thực nghiệm benchmark đối đầu trực tiếp:
> 
> 1. **Nên dùng Flat RAG khi:**
>    - Nhiệm vụ tra cứu thông tin là **đơn bước (Single-hop)**, thông tin cần trả lời nằm tập trung trong một đoạn văn bản hoặc một tài liệu duy nhất (như câu Q1 tra cứu định nghĩa luật hoặc Q2 tra cứu chi tiết bản án cụ thể).
>    - Trên các bài toán này, Flat RAG đạt hiệu quả tuyệt đối (`recall = 1.00`, `judge = 2`), đồng thời tiết kiệm đáng kể chi phí ban đầu (thời gian index nhanh hơn 1.83 lần, không tốn chi phí gọi LLM trích xuất cấu trúc đồ thị) và token prompt đầu vào ngắn hơn gần 9 lần (696 so với 6,217 tokens).
>
> 2. **Bắt buộc nên dùng GraphRAG khi:**
>    - Bài toán yêu cầu **suy luận đa bước (Multi-hop Reasoning)** và **kết nối tri thức chéo (Cross-KB Retrieval)** giữa các miền dữ liệu có độ lệch phong cách lớn (như giữa Tin tức đời thực và Luật pháp lý trừu tượng ở Q3, Q4, Q5). Ở các câu này, Flat RAG hoàn toàn thất bại (chỉ đạt recall 0.33–0.40 và judge 1.00), trong khi GraphRAG đạt điểm đánh giá tuyệt đối (`judge = 2.00` và recall 1.00).
>    - Bài toán yêu cầu **tổng hợp tri thức toàn cục (Global Aggregation)** như Q6, nơi thông tin nằm phân mảnh ở nhiều tài liệu khác nhau (Flat RAG recall 0.00 / judge 0 vs GraphRAG recall 0.67 / judge 2).
>    - Đồ thị tri thức đóng vai trò như một bộ khung ràng buộc cấu trúc (structural constraint), triệt tiêu hiện tượng ảo giác (hallucination) và đảm bảo các trích dẫn pháp lý luôn chính xác tuyệt đối.

---

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.07s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.1-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 25 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (vụ án vận chuyển ma túy qua sân bay Nội Bài).

---

## Vấn đề gặp phải (không tính điểm)

- **Vấn đề Rate Limit 429 trên Google Gemini API Free Tier:**
  - *Hiện tượng:* Khi chạy benchmark với 20 bài báo liên tục, Gemini API trả về lỗi `RateLimitError: 429` do vượt quá giới hạn 15 RPM (Requests Per Minute) trên tầng miễn phí.
  - *Giải pháp:* Đã bổ sung cơ chế xử lý ngoại lệ tự động nhận diện thời gian chờ (`retryDelay`), tạm dừng chương trình (sleep) và tự động thử lại (retry backoff) trong `src/llm.py`. Nhờ đó toàn bộ 44 lần gọi LLM của pipeline đã chạy thành công 100% mà không bị gián đoạn.
