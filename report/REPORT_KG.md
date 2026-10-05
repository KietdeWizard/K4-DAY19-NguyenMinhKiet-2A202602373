# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyen Minh Kiet  **MSSV:** 2A202602373  **Ngày:** 05/10/2026

> Số liệu lấy từ `ket_qua_benchmark_kg.txt`, chạy với `gpt-4o-mini`,
> `text-embedding-3-small`, `top_k=3`, `chunk_size=800`.

## 1. Chi phí

### Indexing (one-off)

| Pipeline | Calls | Input tokens | Output tokens | USD | Seconds |
| --- | ---: | ---: | ---: | ---: | ---: |
| Flat | 176 | 56,072 | 0 | 0.00112 | 57.0 |
| Graph | 196 | 91,958 | 4,947 | 0.00947 | 124.1 |

### Querying (mean per question)

| Pipeline | Recall | Judge | Input tokens | Output tokens | USD | Seconds |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Flat | 0.43 | 1.00 | 694 | 47 | 0.00013 | 1.45 |
| Graph | 0.94 | 1.83 | 5,875 | 80 | 0.00092 | 2.75 |

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | $0.00112 | $0.00947 | 8.46× |
| Indexing giây | 57.0 | 124.1 | 2.18× |
| Mỗi câu: USD | $0.00013 | $0.00092 | 7.08× |
| Mỗi câu: giây | 1.45 | 2.75 | 1.90× |
| Mỗi câu: input tokens | 694 | 5,875 | 8.47× |

Graph indexing thêm khoảng $0.00835 và 67.1 giây so với Flat. Khi truy vấn, graph
làm input trung bình tăng mạnh (694 lên 5,875 token), giải thích phần lớn chi phí
mỗi câu tăng. Đây là số liệu trên corpus và cấu hình của lần chạy này.

## 2. Từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa nằm trong một đoạn luật; Flat đã lấy đủ thông tin. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên hai bị cáo và mức án có trong tin; graph không tăng điểm. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Graph nối người/vụ án với tội danh, Điều 251 và khoản 1; chunks riêng lẻ không đủ. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Context đưa hành vi và khung cao nhất Điều 255 vào prompt. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Graph nối người, MDMA, khối lượng, Điều 250 và khoản 4; Flat nhầm điểm b thành khoản. |
| Q6 | aggregation | 0.00 / 1 | 0.67 / 1 | Graph về recall; judge hòa | Graph trả lời được nhiều mục hơn nhưng vẫn bỏ sót Lê Minh Thành. |

## 3. Phân tích lỗi

### Lỗi E3: một vụ ngoài đời thành nhiều node Case

- **Hiện tượng:** cùng người/vụ vận chuyển MDMA được biểu diễn bởi hai node `Case`
  có tên và nguồn tài liệu khác nhau.
- **Bằng chứng:** truy vấn trả về hai case của Cái Quang Huy:

```cypher
MATCH (p:Person)-[:INVOLVED_IN]->(k:Case)
WHERE p.name CONTAINS 'Quang Huy'
RETURN p.name AS person, k.name AS case, k.doc_id AS doc_id
ORDER BY k.name, k.doc_id;
```

```text
('Cái Quang Huy', 'Vụ vận chuyển ma túy của Cái Quang Huy',
 'news-100260918080821054')
('Cái Quang Huy', 'Vụ vận chuyển ma túy từ Đức về Việt Nam',
 'news-100260917203001265')
```

- **Nguyên nhân:** KG-2 dùng tên vụ do LLM tự tạo làm khóa `MERGE`; các bài viết mô tả
  cùng sự kiện nhưng đặt tên khác nhau nên không gộp.
- **Đề xuất sửa:** bổ sung khóa/alias sự kiện ổn định hoặc bước entity resolution để
  liên kết nhiều bài về một sự kiện, đồng thời lưu mọi `doc_id` nguồn. Cần kiểm chứng
  trước khi gộp để tránh nhập nhầm các vụ tương tự.

### Lỗi E4: phép đo recall và judge không hoàn toàn đồng thuận

- **Hiện tượng:** keyword recall và LLM-as-judge của Q6 cho kết quả khác nhau.
  Graph có recall `0.67`, judge `1`; Flat recall `0.00`, judge cũng `1`.
- **Bằng chứng:** `must_include` trong `data/benchmark_kg.json` là `Cái Quang Huy`,
  `Lê Minh Thành`, `Pháp y tâm thần`. Câu Graph Q6 nêu Cái Quang Huy và “Viện Pháp y
  tâm thần Trung ương” nhưng không nêu Lê Minh Thành; judge cho điểm 1. Flat cũng được
  judge 1 dù không chứa đủ từ khóa bắt buộc.
- **Nguyên nhân:** `keyword_recall` kiểm tra chuỗi con theo cách máy móc, trong khi
  judge chấm ý nghĩa và có thể chấp nhận câu trả lời chưa đủ. Hai chỉ số dùng tiêu chuẩn
  khác nhau.
- **Đề xuất sửa:** báo cáo đồng thời hai chỉ số; đánh giá riêng từng thực thể/sự kiện
  và xem thủ công trường hợp bất đồng. Đánh đổi là cần thêm nhãn hoặc lượt đánh giá.

### Lỗi E5: câu trả lời bỏ sót dữ kiện có trong graph

- **Hiện tượng:** GraphRAG Q6 không nhắc Lê Minh Thành dù người và liên kết vụ–MDMA
  có trong graph.
- **Bằng chứng:** kết quả Q6 trong `ket_qua_benchmark_kg.txt` nêu “Vụ góp tiền mua ma
  túy tại Hà Nội” nhưng không nêu tên người. Truy vấn cho thấy người đó nằm trong vụ:

```cypher
MATCH (k:Case)-[:INVOLVES]->(:Substance {name:'MDMA'})
OPTIONAL MATCH (p:Person)-[:INVOLVED_IN]->(k)
RETURN k.name AS case, collect(DISTINCT p.name) AS people
ORDER BY case;
```

Kết quả cho `Vụ góp tiền mua ma túy tại Hà Nội` có `Lê Minh Thành` trong `people`.

- **Nguyên nhân:** KG-3 truy xuất được dữ kiện, nhưng bước sinh câu trả lời không đảm
  bảo liệt kê đầy đủ người/vụ phù hợp.
- **Đề xuất sửa:** tạo danh sách kết quả có cấu trúc từ Cypher cho câu hỏi tổng hợp;
  yêu cầu LLM diễn đạt nhưng giữ nguyên mọi mục, rồi xác thực đầu ra có đủ tên/vụ.
  Thêm bước kiểm tra nhưng giảm nguy cơ bỏ sót.

## 4. Kết luận

Với câu hỏi đơn bước (Q1, Q2), Flat RAG đạt cùng điểm với chi phí thấp hơn. Với câu
hỏi xuyên tin tức và luật (Q3–Q5), GraphRAG tăng judge từ 0/1 lên 2/2 và recall lên
1.00; phần chi phí indexing cao hơn khoảng 8.46× và chi phí mỗi câu cao hơn khoảng
7.08× có thể đáng trả khi độ đầy đủ liên KB quan trọng. Q6 cho thấy GraphRAG vẫn bỏ
sót tên trong câu trả lời tổng hợp, nên cần xác minh đầu ra với dữ kiện Cypher.

## 5. Tự kiểm

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed
```

```text
$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00077.
```

Benchmark cuối: `ket_qua_benchmark_kg.txt` — graph có 206 node / 387 quan hệ.

Ảnh Neo4j:
- `report/img/kg_count.png`
- `report/img/kg_cross_kb.png`
- `report/img/kg_my_case.png`

**Trạng thái ảnh:** cả ba ảnh đã được lưu. `kg_count.png` thấy truy vấn và bảng kết
quả. `kg_cross_kb.png` và `kg_my_case.png` hiện chỉ thấy Graph/Results overview, chưa
thấy ô truy vấn; cần chụp lại hai ảnh này với truy vấn hiển thị để đáp ứng quy cách.

**Người đã chọn cho `kg_my_case.png`:** Trần Thanh Tuấn (không phải Lê Minh Thành).

## Vấn đề gặp phải

- Kết quả trích xuất LLM có thể khác giữa các lần build; số liệu báo cáo được giữ theo
  lần benchmark cuối ở trên.
- Q6 còn sai khác giữa graph facts, câu trả lời và judge; xem E4–E5.
