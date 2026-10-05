## Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Văn Đại  
**MSSV:** 2A202602477  
**Ngày:** 05/10/2026

---

## 1. Chi phí (10 điểm)

Kết quả từ `ket_qua_benchmark_kg.txt`:

```text
Chat model: openai:gpt-4o-mini | Embedding: openai:text-embedding-3-small
top_k=3 | chunk_size=800 | chunks=176 | KG: 206 nodes / 384 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     81.7
graph       196     91958     4832   0.00940    149.1

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     1.43
graph       0.89   1.67     3902       75   0.00062     2.05
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0,00112 | 0,00943 | ×8,42 |
| Indexing giây | 62,9 | 120,7 | ×1,92 |
| Mỗi câu: USD | 0,00013 | 0,00063 | ×4,85 |
| Mỗi câu: giây | 1,43 | 2,14 | ×1,50 |
| Mỗi câu: in_tok | 694 | 3.947 | ×5,69 |

Chi phí dựng GraphRAG tăng chủ yếu do 20 lần gọi chat để trích xuất thực thể/quan hệ từ tin tức: số call tăng từ 176 lên 196, phát sinh 4.876 output token và tổng input tăng 35.886 token. Khi truy vấn, prompt graph dài hơn vì chứa các cạnh một-hop và các khoản luật được mở rộng. Phần indexing tăng thêm 0,00831 USD, tương đương khoảng 17 lần phần chi phí truy vấn tăng thêm (0,00050 USD/câu); đây là mốc quy mô chi phí, không phải điểm GraphRAG trở nên rẻ hơn vì chi phí mỗi câu của GraphRAG vẫn cao hơn.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | ---: | ---: | --- | --- |
| Q1 | single-hop-law | 1,00 / 2 | 1,00 / 2 | Hòa | Định nghĩa nằm gọn trong một chunk luật nên vector retrieval đã đủ. |
| Q2 | single-hop-news | 1,00 / 2 | 1,00 / 2 | Hòa | Hai tên bị cáo cùng nằm trong một bài báo; graph không bổ sung thông tin cần thiết. |
| Q3 | cross-kb | 0,00 / 0 | 0,67 / 1 | Graph | Graph nối vụ của Lê Minh Thành qua tội danh tới Điều 251 và khoản 1, dù trích sai mức án 24 thay vì 36 tháng. |
| Q4 | cross-kb | 0,00 / 0 | 0,67 / 1 | Graph | Graph tìm được hành vi và Điều 255, nhưng chỉ đưa khoản 1 nên trả sai mức tối đa. |
| Q5 | cross-kb-multi-hop | 0,60 / 1 | 1,00 / 2 | Graph | Đường đi vụ → chất → tội → Điều 250 cung cấp đủ MDMA, khoản 4 và tử hình. |
| Q6 | aggregation | 0,00 / 1 | 0,67 / 1 | Graph | Graph tổng hợp được nhiều node `Case` có MDMA nhưng thiếu tên Lê Minh Thành và có các vụ trùng/đặt tên khác. |

Quy luật quan sát được: câu single-hop hòa nhau (Q1–Q2), còn GraphRAG thắng cả bốn câu cần nối nguồn, nhiều bước hoặc tổng hợp (Q3–Q6). Trung bình GraphRAG tăng recall từ 0,43 lên 0,83 và judge từ 1,00 lên 1,50, nhưng không tự loại bỏ lỗi trích xuất và lỗi chọn ngữ cảnh.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật khi hỏi mức phạt tối đa

- **Hiện tượng:** ở Q4, GraphRAG tìm đúng hành vi và Điều luật nhưng trả lời “tối đa 7 năm theo Điều 255 BLHS khoản 1”, trong khi đáp án đúng là 20 năm hoặc tù chung thân ở khoản 4. Q4 chỉ đạt recall 0,67 và judge 1.
- **Bằng chứng:** câu trả lời nguyên văn: “Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Hành vi này có thể bị phạt tù tối đa 7 năm theo Điều 255 BLHS khoản 1.” Graph thật có đủ mọi khoản:

```cypher
MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number AS clause, cl.penalty AS penalty ORDER BY cl.number;
```

```text
1 | phạt tù từ 02 năm đến 07 năm
2 | phạt tù từ 07 năm đến 15 năm
3 | phạt tù từ 15 năm đến 20 năm
4 | phạt tù 20 năm hoặc tù chung thân
5 | phạt tiền ...
```

- **Nguyên nhân:** lỗi nằm ở bước retrieval KG-3. `context()` giữ khoản 1 và các khoản nhắc tới chất mà vụ án `INVOLVES`. Các vụ liên quan Hoàng Nato được trích xuất với `etomidate`, trong khi Điều 255 không có cạnh `MENTIONS` tới chất này, nên khoản 4 không vào prompt dù graph lưu đầy đủ.
- **Đề xuất sửa:** nhận diện ý định “tối đa/khung cao nhất” và lấy toàn bộ khoản của Điều luật, hoặc cấu trúc hóa cận trên và loại hình phạt thành property để Cypher chọn khung cao nhất. Chỉ lọc theo chất đối với câu hỏi ngưỡng khối lượng như Q5.

### Lỗi E4: Recall từ khóa và LLM-as-judge đo hai khía cạnh khác nhau

- **Hiện tượng:** Q6 của Flat RAG có recall 0,00 nhưng judge 1; GraphRAG có recall 0,67 nhưng cũng judge 1. Vì vậy chỉ nhìn một thước đo sẽ dẫn đến kết luận thiếu chính xác.
- **Bằng chứng:** `must_include` của Q6 là `Cái Quang Huy`, `Lê Minh Thành`, `Pháp y tâm thần`. Flat trả lời bằng các tên rút gọn “vụ việc của Đức”, “Thành”, “Đông”, nên không khớp chuỗi nào và recall bằng 0, nhưng judge vẫn cho 1 vì nhận ra nội dung liên quan MDMA. Graph nêu `Cái Quang Huy` và `Viện Pháp y tâm thần Trung ương` nhưng thiếu `Lê Minh Thành`, nên recall 2/3 và judge vẫn là 1.

```text
Q6 flat:  recall=0.00, judge=1
Q6 graph: recall=0.67, judge=1
```

- **Nguyên nhân:** keyword recall đo khớp chuỗi bắt buộc, nhạy với cách gọi tên; judge đo mức đúng nghĩa nhưng có tính chủ quan. Ngoài ra ontology `MERGE Case` theo tên do LLM đặt nên cùng bài `news-100260918080821054` tạo cả “Vụ góp tiền mua ma túy tại Hà Nội” và “Vụ vận chuyển ma túy của Cái Quang Huy”, làm phép tổng hợp dễ thừa/thiếu.
- **Đề xuất sửa:** chấm theo entity đã chuẩn hóa hoặc alias thay vì chuỗi thô, bổ sung precision cho câu aggregation và kiểm tra số vụ duy nhất theo `doc_id`/case identity. Báo cáo đồng thời recall và judge, không dùng một chỉ số đơn lẻ.

## 4. Kết luận (5 điểm)

KG đáng dùng khi dữ liệu có quan hệ ổn định giữa nhiều nguồn và câu hỏi cần join hoặc multi-hop: trong thí nghiệm này, GraphRAG thắng Q3–Q6, nâng recall trung bình 0,40 điểm và judge 0,50 điểm. Đổi lại, indexing tốn ×8,42 USD và truy vấn tốn ×4,85 USD, chậm ×1,50 và dùng ×5,69 input token.

Flat RAG là đủ cho câu hỏi single-hop khi đáp án nằm trong một chunk, thể hiện ở Q1–Q2 khi cả hai pipeline đều đạt recall 1,00 và judge 2. Với hệ thống chủ yếu hỏi định nghĩa hoặc tra một bài báo, chi phí ontology và extraction không đáng. KG phù hợp hơn khi tỷ lệ câu cross-KB/multi-hop/aggregation đủ lớn và giá trị của mức tăng chính xác cao hơn khoảng 0,00050 USD chi phí thêm mỗi câu; vẫn cần kiểm soát entity linking, chọn khoản luật và deduplicate thực thể.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.07s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 148 node / 293 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 22 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00078. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j:

- `report/img/kg_count.png`: Q-A, đủ 7 label.
- `report/img/kg_cross_kb.png`: truy vấn `shortestPath` xuyên hai KB và Results overview.
- `report/img/kg_my_case.png`: Q-D với người tự chọn **Trịnh Vũ Kiên**.

## Vấn đề gặp phải (không tính điểm)

Ban đầu API key được đặt nhầm ở `tests/.env`, nên `bench_kg.py` không nạp được; đã chuyển về `.env` ở thư mục gốc. Lần chạy đầu của KG-2 không tạo node tin tức vì lời gọi extraction chưa bật JSON mode; đã sửa thành `llm_fn(prompt, json_mode=True)`. Sau sửa, `--check`, `--judge` và toàn bộ test đều hoàn tất.
