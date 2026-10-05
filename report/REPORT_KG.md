# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Thu Trang  **MSSV:** 2A202602435  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu trong báo cáo này khớp 100% với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

---

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    117.8
graph       196     34619     5616   0.00000    185.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       74   0.00000     1.71
graph       1.00   2.00     4550      151   0.00000     2.30
```

| Chỉ số | Flat | Graph | Graph / Flat |
| :--- | :--- | :--- | :--- |
| **Indexing USD** | $0.00000 | $0.00000 | ×1.0 |
| **Indexing giây** | 117.8s | 185.3s | ×1.57 |
| **Mỗi câu: USD** | $0.00000 | $0.00000 | ×1.0 |
| **Mỗi câu: giây** | 1.71s | 2.30s | ×1.35 |
| **Mỗi câu: in_tok** | 696 | 4,550 | ×6.54 |

**Chi phí tăng thêm đến từ đâu?**
> Chi phí tăng thêm của GraphRAG ở giai đoạn **Indexing** (thêm ~67.5 giây và 34,619 input tokens, 5,616 output tokens) đến từ việc phải gọi LLM trích xuất có cấu trúc (JSON extraction) cho 20 bài báo tin tức để dựng các node `Case`, `Person`, `Location` và các mối quan hệ. Ở giai đoạn **Querying**, số token đầu vào mỗi câu hỏi tăng gấp 6.54 lần (4,550 so với 696 tokens) do prompt phải chứa thêm danh sách facts phong phú trích xuất từ các đường đi multi-hop trên đồ thị (bao gồm tóm tắt vụ án, Điều luật, và nội dung chi tiết các khoản liên quan), kéo theo độ trễ tăng nhẹ từ 1.71s lên 2.30s (+34.5%).

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| :---: | :--- | :---: | :---: | :---: | :--- |
| **Q1** | single-hop-law | 1.00 / 2 | 1.00 / 2 | **Hòa** | Định nghĩa tiền chất nằm trọn trong Điều 2 Luật PCMT 2021 nên Flat RAG tìm kiếm vector là đủ để trả lời chính xác. |
| **Q2** | single-hop-news | 1.00 / 2 | 1.00 / 2 | **Hòa** | Thông tin 2 bị cáo tuyên tử hình nằm gọn trong 1 bài báo xét xử 36kg ma túy, cả hai pipeline đều trích xuất hoàn hảo. |
| **Q3** | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ biết mức án 36 tháng tù nhưng không biết Điều luật nào; GraphRAG duyệt qua node cầu nối `Crime` tìm đúng Điều 251 khoản 1 (02–07 năm). |
| **Q4** | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG không có thông tin khung phạt tối đa; GraphRAG kết nối hành vi sang Điều 255 khoản 4 xác định mức phạt tối đa 20 năm hoặc chung thân. |
| **Q5** | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Flat RAG không biết khoản luật và khung phạt cho MDMA; GraphRAG đối chiếu lượng 9,6kg MDMA sang khoản 4 Điều 250 (20 năm, chung thân hoặc tử hình). |
| **Q6** | aggregation | 0.00 / 1 | 1.00 / 2 | **Graph** | Flat RAG chỉ trích dẫn chung chung 3 đoạn văn mà không nêu được tên bị cáo/vụ việc; GraphRAG gom đủ cả 3 vụ việc từ quan hệ liên kết với node `MDMA`. |

---

## 3. Phân tích lỗi (20 điểm)

#### Lỗi E1: Cầu nối gãy (vụ án không nối được sang luật)

- **Hiện tượng:** Một số vụ việc trong tin tức sau khi trích xuất không tạo được quan hệ `CHARGED_WITH` tới bất kỳ node `Crime` nào, khiến đường đi sang KB Luật bị đứt hoàn toàn.
- **Bằng chứng:** Truy vấn Cypher kiểm tra các `Case` không có quan hệ `CHARGED_WITH`:

```cypher
MATCH (k:Case)
WHERE NOT (k)-[:CHARGED_WITH]->()
RETURN k.name AS name, k.doc_id AS doc_id;
```

```
╒══════════════════════════════════════════════════════════════════════════╤══════════════════════════╕
│name                                                                      │doc_id                    │
╞══════════════════════════════════════════════════════════════════════════╪══════════════════════════╡
│"Vụ vận chuyển vũ khí và hơn 800kg chất nghi ma túy tại Preah Sihanouk"   │"news-100260924145818945" │
│"Vụ tông cảnh sát giao thông tại An Giang"                                │"news-100260926112415229" │
│"Triệt phá chuyên án A3-626P"                                             │"news-100261002184934505" │
└──────────────────────────────────────────────────────────────────────────┴──────────────────────────┘
```

- **Nguyên nhân:** Nằm ở khâu **Crawl dữ liệu và Prompt trích xuất**:
  1. Vụ tại Preah Sihanouk (`news-100260924145818945`) xảy ra tại Campuchia và báo chí mới ghi nhận là *"chất nghi ma túy"*, chưa có kết luận giám định chính thức nên không có tội danh tương ứng trong BLHS Việt Nam.
  2. Vụ tông CSGT tại An Giang (`news-100260926112415229`) tập trung mô tả hành vi chống người thi hành công vụ trong quá trình tẩu thoát thay vì nêu tội danh ma túy cụ thể.
  3. Vụ chuyên án A3-626P (`news-100261002184934505`) lấy tên theo bí số chuyên án trinh sát ban đầu, hành vi chưa được khởi tố với tội danh chuẩn thuộc Chương XX BLHS.
  Do đó, LLM không thể ánh xạ về bất kỳ tội danh nào trong danh mục 13 tội danh BLHS, dẫn đến trường `charges` bị rỗng và không tạo được quan hệ `CHARGED_WITH`.
- **Đề xuất sửa:** Trong prompt trích xuất, hướng dẫn LLM: nếu bài báo chỉ nói *"chất nghi ma túy"* hoặc chưa có tội danh chính thức, không tạo quan hệ giả định; đồng thời bổ sung cơ chế fallback nối `Case` trực tiếp tới node `Substance` hoặc `Location` để khi người dùng hỏi vẫn có thể định tuyến ngữ cảnh qua tang vật.

---

### Lỗi E3: Trùng thực thể & Nguy cơ phân mảnh đối tượng qua Biệt danh (Aliases)

- **Hiện tượng:** Các đối tượng trong vụ án ma túy thường có biệt danh giang hồ (aliases). Khóa định danh hiện tại chỉ `MERGE` theo thuộc tính `name`, dẫn đến nguy cơ nếu bài báo khác gọi đối tượng bằng biệt danh thay vì tên thật thì đồ thị sẽ bị tách thành các node riêng biệt hoặc không truy vấn được.
- **Bằng chứng:** Truy vấn Cypher kiểm tra các đối tượng có bí danh / biệt danh trong đồ thị thực tế:

```cypher
MATCH (p:Person)
WHERE size(p.aliases) > 0
RETURN p.name AS name, p.aliases AS aliases;
```

```
╒═════════════════════════╤═════════════════╕
│name                     │aliases          │
╞═════════════════════════╪═════════════════╡
│"Dương Minh Tuấn"        │["Hoàng Nato"]   │
│"Phan Kim Nhi"           │["Phannhibeauty"]│
│"Nguyễn Minh Đức"        │["Đức Cộng"]     │
│"Nguyễn Thị Mai Anh"     │["bà trùm"]      │
└─────────────────────────┴─────────────────┘
```

- **Nguyên nhân:** Nằm ở **Thiết kế Ontology (Khóa định danh)**. Báo chí tiếng Việt có thói quen gọi nhân vật bằng biệt danh (ví dụ: *"bà trùm"*, *"Đức Cộng"*, *"Hoàng Nato"*, *"Phannhibeauty"*). Vì schema hiện tại quy định `CONSTRAINT FOR (p:Person) REQUIRE p.name IS UNIQUE`, nếu có bài báo khác chỉ viết *"Bà trùm bị bắt"* hoặc *"Hoàng Nato khai nhận"*, LLM sẽ trích `name: "bà trùm"` hoặc `name: "Hoàng Nato"`, tạo thành node `Person` mới hoàn toàn tách rời khỏi node mang tên thật *"Nguyễn Thị Mai Anh"* hay *"Dương Minh Tuấn"*.
- **Đề xuất sửa:** Bổ sung cơ chế Phân giải thực thể (Entity Resolution): Trước khi `MERGE` một `Person` mới, thực hiện truy vấn đối chiếu xem tên mới có trùng với bất kỳ phần tử nào trong mảng `aliases` của các node đã có hay không. Nếu trùng, thực hiện gộp thuộc tính vào node đã có thay vì tạo node mới. Đánh đổi: tăng số lượng câu lệnh Cypher kiểm tra trong quá trình indexing.

---

## 4. Kết luận (5 điểm)

Từ số liệu thực nghiệm đo đếm được ở Mục 1 và Mục 2:

1. **Khi nào Flat RAG là đủ?**
   - Khi hệ thống phục vụ các câu hỏi mang tính **cục bộ, đơn nguồn (single-hop)** như Q1 và Q2 (cả hai đều đạt Recall 1.00 và Judge 2.00).
   - Với các câu hỏi này, Flat RAG tối ưu hơn hẳn: **nhanh hơn 34.5%** (1.71s so với 2.30s), **tiết kiệm token input gấp 6.54 lần** (696 so với 4,550 tokens) và **không tốn chi phí dựng đồ thị ban đầu** (117.8s so với 185.3s).

2. **Khi nào nên đầu tư làm Knowledge Graph (GraphRAG)?**
   - Khi dữ liệu nằm phân tán ở **nhiều nguồn tri thức khác nhau** và câu hỏi đòi hỏi **liên kết đa chặng (cross-KB, multi-hop)** như Q3, Q4, Q5 (Flat RAG thất bại với recall chỉ đạt 0.33 – 0.40, trong khi GraphRAG đạt tuyệt đối 1.00).
   - Khi cần thực hiện các câu hỏi **tổng hợp toàn cục (aggregation)** như Q6 (Flat RAG đạt recall 0.00 do context window bị giới hạn không lấy đủ chunk từ 3 bài báo khác nhau, trong khi GraphRAG đạt recall 1.00 nhờ liên kết tập trung qua node `Substance: MDMA`).
   - Tóm lại: Knowledge Graph hoàn toàn "đáng tiền" khi giá trị của độ chính xác và khả năng kết nối tri thức xuyên nguồn quan trọng hơn chi phí token bổ sung lúc truy vấn.

---

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.02s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 15 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.  
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy** (Vụ vận chuyển hơn 9,6kg MDMA qua sân bay Nội Bài).

---

## Vấn đề gặp phải (không tính điểm)

- **Hiện tượng:** Khi chạy `bench_kg.py --judge` lần đầu với Google Gemini Free Tier, quá trình bị ngắt giữa chừng ở câu Q4 do lỗi Rate Limit `429 RESOURCE_EXHAUSTED` (vượt quá hạn mức 15 requests/phút).
- **Cách xử lý:** Đã bổ sung cơ chế tự động thử lại kèm thời gian chờ (exponential backoff & retry with sleep) trong hàm `chat` và `embed` của `src/llm.py`. Quá trình chạy lại sau đó đã hoàn tất 100% trơn tru và sinh đầy đủ kết quả benchmark.
