# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Văn Việt  **MSSV:** 2A202602904  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu dưới đây lấy từ `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nằm tại `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Hai bảng kết quả benchmark:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112    230.3
graph       196     91958     4630   0.00928    322.5

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     4.37
graph       0.83   1.83     5639       79   0.00089     8.86
```

| Chỉ số          |    Flat |   Graph | Graph / Flat |
| ----------------- | ------: | ------: | -----------: |
| Indexing USD      | 0,00112 | 0,00928 |       8,29× |
| Indexing giây    |   230,3 |   322,5 |       1,40× |
| Mỗi câu: USD    | 0,00013 | 0,00089 |       6,85× |
| Mỗi câu: giây  |    4,37 |    8,86 |       2,03× |
| Mỗi câu: in_tok |     694 |   5.639 |       8,13× |

**Chi phí tăng thêm đến từ đâu?**

GraphRAG có thêm 20 lần gọi LLM để trích xuất 20 bài báo thành graph, làm chi phí indexing tăng thêm 0,00816 USD. Khi truy vấn, prompt GraphRAG chứa cả top-k chunks và dữ kiện multi-hop từ graph nên input token trung bình tăng từ 694 lên 5.639, kéo chi phí và độ trễ mỗi câu lên tương ứng 6,85× và 2,03×. Với cấu hình hiện tại không có điểm hòa vốn thuần về tiền vì cả chi phí cố định lẫn chi phí biên của GraphRAG đều cao hơn; phần chi phí thêm được đổi lấy độ chính xác tốt hơn ở câu cross-KB.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại              | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu)                                                                                                                                                          |
| ---- | ------------------ | ------------------- | -------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Q1   | single-hop-law     | 1,00 / 2            | 1,00 / 2             | Hòa   | Định nghĩa tiền chất nằm gọn trong Điều 2 nên một chunk của Flat RAG đã đủ; graph chỉ bổ sung số khoản.                                               |
| Q2   | single-hop-news    | 1,00 / 2            | 1,00 / 2             | Hòa   | Tên hai bị cáo và án tử hình cùng nằm trong một bài báo nên không cần traversal nhiều bước.                                                             |
| Q3   | cross-kb           | 0,00 / 0            | 1,00 / 2             | Graph  | Flat trả lời “Không đủ thông tin”, còn graph nối Lê Minh Thành qua tội danh tới Điều 251 và khoản 1 từ 02 đến 07 năm.                               |
| Q4   | cross-kb           | 0,00 / 0            | 1,00 / 2             | Graph  | Graph nối alias Hoàng Nato tới Điều 255 và lấy khoản 4 để trả lời mức tối đa là 20 năm hoặc chung thân.                                                |
| Q5   | cross-kb-multi-hop | 0,60 / 1            | 1,00 / 2             | Graph  | Flat biết MDMA và khung án nhưng ghi “khoản b”, thiếu Điều 250; graph kết hợp tội danh, chất và khối lượng để xác định đúng khoản 4 Điều 250. |
| Q6   | aggregation        | 0,00 / 1            | 0,00 / 1             | Hòa   | Cả hai pipeline nhận ra ba nhóm vụ MDMA nhưng dùng tên vụ/tên gọi rút gọn, không nêu đủ ba tên riêng trong`must_include`.                             |

Quy luật quan sát được là Flat RAG đủ tốt khi đáp án nằm trong một tài liệu hoặc một chunk (Q1–Q2), còn GraphRAG thắng rõ khi phải nối người/vụ trong tin tức với tội danh, Điều và khoản trong luật (Q3–Q5). Graph không tự bảo đảm câu aggregation đúng cách trình bày nếu context và prompt không buộc LLM xuất tên thực thể chuẩn (Q6).

## 3. Phân tích lỗi (20 điểm)

### Lỗi E3: Trùng thực thể do khóa định danh chưa được chuẩn hóa

- **Hiện tượng:** cùng một chất ngoài đời được tạo thành hai node chỉ vì khác chữ hoa/chữ thường. Ngoài ra, nhiều bài viết về cùng một chuỗi sự kiện Hoàng Nato tạo các `Case` có tên khác nhau; riêng Dương Minh Tuấn nối tới bốn `Case`.
- **Bằng chứng:**

```cypher
MATCH (s:Substance)
WITH toLower(s.name) AS normalized, collect(s.name) AS names, count(*) AS n
WHERE n > 1
RETURN normalized, names, n
ORDER BY normalized;
```

```text
normalized          names                                  n
ketamine            ["Ketamine", "ketamine"]              2
methamphetamine     ["methamphetamine", "Methamphetamine"] 2
```

Truy vấn bổ sung cho vụ việc:

```cypher
MATCH (p:Person)-[:INVOLVED_IN]->(k:Case)
WHERE p.name = 'Dương Minh Tuấn'
RETURN p.name, p.aliases, collect({case_name: k.name, doc_id: k.doc_id}) AS cases;
```

```text
Dương Minh Tuấn, ["Hoàng Nato"]
- Vụ bắt giang hồ 'Hoàng Nato' và 126 người liên quan 8 đường dây ma túy
- Vụ bắt giữ TikToker Phannhibeauty và giang hồ 'Hoàng Nato'
- Vụ sử dụng ma túy etomidate của Hoàng Nato và Phan Kim Nhi
- Vụ triệt phá 8 đường dây ma túy tại TP.HCM
```

- **Nguyên nhân:** lỗi nằm ở thiết kế ontology và bước trích xuất tin. `MERGE (sub:Substance {name: s.name})` so khớp chuỗi phân biệt hoa/thường, trong khi tên chất do LLM sinh chưa được đưa qua một hàm chuẩn hóa. `Case` dùng `name` do LLM tự đặt làm khóa nên mỗi bài có thể đặt một tên khác cho cùng một sự kiện hoặc chuỗi điều tra.
- **Đề xuất sửa:** trước `add_news_case`, chuẩn hóa chất bằng bảng alias không phân biệt hoa/thường, ví dụ ánh xạ mọi biến thể về tên chuẩn trong `SUBSTANCES`, rồi `MERGE` theo `canonical_name`. Với `Case`, tạo `case_id` ổn định từ các tín hiệu như người chính, ngày, địa điểm và loại hành vi, sau đó thực hiện entity resolution giữa các bài. Việc này cần thêm luật ghép hoặc một lượt LLM/entity matching, làm indexing chậm và đắt hơn; ghép quá mạnh cũng có nguy cơ nhập nhầm hai vụ khác nhau.

### Lỗi E5: Câu aggregation không sử dụng hết tên thực thể đã có trong graph

- **Hiện tượng:** ở Q6, GraphRAG mô tả đúng ba nhóm vụ liên quan MDMA nhưng gọi chúng bằng tên vụ chung như “Vụ góp tiền mua ma túy tại Hà Nội” và “Vụ vận chuyển ma túy từ Đức về Việt Nam”. Câu trả lời không nêu đầy đủ `Cái Quang Huy`, `Lê Minh Thành` và `Pháp y tâm thần`, nên recall bằng 0 và judge chỉ bằng 1 mặc dù các tên người tương ứng có trong graph.
- **Bằng chứng từ `ket_qua_benchmark_kg.txt` — Q6 GraphRAG:**

```text
1. Vụ góp tiền mua ma túy tại Hà Nội: Trong vụ này, có liên quan đến 5 viên MDMA.
2. Vụ tổ chức sử dụng ma túy tại Sầm Sơn: Tại đây, có thu giữ 0,686g ma túy MDMA.
3. Vụ vận chuyển ma túy từ Đức về Việt Nam: Vụ này liên quan đến 9,6kg MDMA.
```

Graph thực tế có các tên mà câu trả lời bỏ sót:

```cypher
MATCH (:Substance {name:'MDMA'})<-[:INVOLVES]-(k:Case)
OPTIONAL MATCH (p:Person)-[:INVOLVED_IN]->(k)
RETURN k.name AS case_name, k.doc_id AS doc_id,
       collect(DISTINCT p.name) AS people
ORDER BY case_name;
```

```text
Vụ góp tiền mua ma túy tại Hà Nội
  people: [Lê Minh Thành, Trịnh Vũ Kiên, Kim Xuân Tuấn, Nguyễn Quang Hưng]

Vụ vận chuyển ma túy từ Đức về Việt Nam
  people: [Cái Quang Huy, Nguyễn Tiến Đạt]

Vụ án tại Viện Pháp y tâm thần Trung ương
  people: [Bùi Thị Thanh Thủy, ..., Lê Văn Đông, Trần Quốc An, Nguyễn Thị Mai Anh, ...]

Vụ tổ chức sử dụng ma túy tại Sầm Sơn
  people: [Nguyễn Thị Mai Anh, Trần Quốc An, ..., Lê Văn Đông]
```

- **Nguyên nhân:** lỗi nằm ở KG-3 và prompt trả lời. Với câu aggregation, `context()` lấy tóm tắt `Case`, sau đó thêm nhiều khoản luật trước các cạnh `Person-[:INVOLVED_IN]`; giới hạn `max_facts` có thể làm tên người bị đẩy ra khỏi context. Hai node vụ Viện Pháp y tâm thần/Sầm Sơn còn biểu diễn các góc nhìn chồng lặp, khiến LLM ưu tiên một tên vụ chung thay vì tên chuẩn trong đáp án.
- **Đề xuất sửa:** phát hiện ý định aggregation qua các cụm “những vụ”, “các vụ”, rồi dùng Cypher chuyên biệt trả đúng một fact cho mỗi vụ gồm `case_name`, `source_title`, danh sách người và lượng MDMA; bỏ phần mở rộng khoản luật vì Q6 không hỏi hình phạt. Prompt cần yêu cầu nêu nguyên tên người/tổ chức và không thay bằng tên vụ chung. Cách này giảm token luật và tăng độ chính xác, nhưng cần thêm logic phân loại câu hỏi và entity resolution để gộp hai bài cùng sự kiện.

## 4. Kết luận (5 điểm)

Knowledge Graph đáng dùng khi dữ kiện phải đi qua nhiều nguồn và nhiều bước, ví dụ từ người trong tin tức sang tội danh, Điều luật, khoản và khung hình phạt. Trong benchmark này, GraphRAG đạt recall/judge 1,00/2 ở cả Q3–Q5, trong khi Flat lần lượt chỉ đạt 0,00/0; 0,00/0; và 0,60/1. Đổi lại, GraphRAG tốn 0,00928 USD để indexing thay vì 0,00112 USD, và mỗi câu tốn trung bình 0,00089 USD cùng 8,86 giây, cao hơn Flat 6,85× về tiền và 2,03× về thời gian.

Flat RAG phù hợp hơn khi đáp án nằm trọn trong một điều luật hoặc một bài báo như Q1–Q2: cả hai pipeline đều được judge 2 và recall 1,00, nên phần chi phí graph không tạo thêm chất lượng đáng kể. KG nên được chọn khi hệ thống thường xuyên nhận câu cross-KB, cần quan hệ có cấu trúc hoặc cần truy vết nguồn; Flat phù hợp cho tra cứu single-hop, tải lớn và yêu cầu phản hồi rẻ/nhanh. Với aggregation như Q6, cần thêm truy vấn chuyên biệt và chuẩn hóa thực thể; chỉ có graph nhưng context/prompt chưa đúng vẫn không bảo đảm câu trả lời đầy đủ.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed, 1 warning in 0.07s
```

Lần chạy `python bench_kg.py --check` trước khi hoàn thành KG-4 đã xác nhận:

```text
[OK] KG-3 context: 13 dữ kiện, có Điều 251
```

Sau khi hoàn thành KG-4, phần hợp đồng cuối của `--check` đã được tái hiện cục bộ trên cùng graph và đạt:

```text
bench KG-4 contract reproduction: OK
prompt_contains_article_251=yes
prompt_contains_chunk_36_months=yes
```

> Cần chạy lại `python bench_kg.py --check` một lần ở trạng thái code cuối và thay khối trên bằng toàn bộ 7 dòng `[OK]` trước khi nộp. Lệnh gọi LLM trên một bài báo nên chưa được tự động chạy lại trong lúc viết báo cáo.

Ảnh Neo4j:

- `report/img/kg_count.png`
- `report/img/kg_cross_kb.png`
- `report/img/kg_my_case.png`

Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy**.

## Vấn đề gặp phải (không tính điểm)

- Môi trường hiện dùng Python 3.14.4 thay vì Python 3.11 như hướng dẫn. Toàn bộ 48 test vẫn đạt, nhưng nên dùng Python 3.11 để đúng môi trường chấm.
- Pytest cảnh báo không ghi được `.pytest_cache` (`WinError 183`); cảnh báo không ảnh hưởng kết quả 48 test.
- Lần đầu chạy `--check`, KG-3 đã đạt nhưng lệnh dừng ở `NotImplementedError` của KG-4. Sau đó `GraphRAGAgent.answer` đã được triển khai, 2 test KG-4 và toàn bộ 48 test đều đạt.
