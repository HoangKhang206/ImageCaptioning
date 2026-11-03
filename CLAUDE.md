# CLAUDE.md — Đồ án Image Captioning (BLIP-2 & LLaVA)

> File context cho CLI. Đọc trước khi làm việc. Trả lời bằng tiếng Việt, ngắn gọn, trực tiếp.

## 1. Tổng quan dự án

- **Tiêu đề chính thức (giảng viên giao):** Khảo sát sinh câu chú thích từ ảnh (Image2Text) dựa vào mô hình ngôn ngữ trực quan (VLM) BLIP-2 hay mô hình ngôn ngữ lớn đa phương thức (Multimodal LLM) LLaVA.
- **Chủ đề:** Đồ án R&D về **Image Captioning** (sinh chú thích ngôn ngữ tự nhiên từ ảnh).
- **Hai hướng tiếp cận chính:** BLIP-2 (Vision-Language Model, Q-Former) và LLaVA (Multimodal LLM, visual instruction tuning).
- **Loại đồ án:** Khảo sát + cài đặt thử nghiệm (không huấn luyện mô hình mới từ đầu). Gọi mô hình qua thư viện Transformers với trọng số công bố; phần tự viết là dữ liệu, vòng chạy, cải tiến, đánh giá, ứng dụng.
- **Sinh viên:** Khang — MSSV 24120334 — HCMUS. Bối cảnh học thuật tiếng Việt.

## 2. Phát biểu bài toán

- **Input:** một ảnh tĩnh.
- **Output:** một câu/đoạn mô tả ngôn ngữ tự nhiên nội dung ảnh.
- Phân biệt rõ với VQA: captioning dùng prompt mô tả **cố định** ("hãy mô tả ảnh này"), không phải hỏi-đáp theo input người dùng.
- **Framework chung:** tiền xử lý ảnh → trích xuất đặc trưng thị giác (vision encoder) → cầu nối thị giác-ngôn ngữ → sinh chú thích (LLM/decoder) → đánh giá (CIDEr, SPICE, CLIPScore).

## 3. Hướng phát triển đã chốt (Khang chốt 03/11/2026)

Tổ hợp **1A + 2A + 2B**, giữ phần đọc biểu đồ:

- **1A — Ứng dụng:** công cụ tự chú thích hình trong file Word: sinh caption, đánh số "Hình n.", tạo mục lục hình.
- **2A — Cải tiến bằng ngữ cảnh:** đưa đoạn văn nhắc đến hình vào prompt. Kế thừa SciCap+ [40], Summaries as Captions [41]; điểm khác: mô hình đa phương thức tổng quát, không huấn luyện lại.
- **2B — Cải tiến bằng tự kiểm tra:** sinh caption → hỏi lại chính mô hình về từng đối tượng/số liệu → bỏ ý sai. Kế thừa Woodpecker [42]; điểm khác: không dùng detector ngoài, mở rộng sang dữ kiện biểu đồ.
- **Khoảng trống:** ảnh học thuật (biểu đồ, figure) hay bị mô tả chung chung hoặc sai dữ kiện [27, 28, 34, 35]. Word đã có alt-text AI [43] nhưng là mô tả ẩn, không phải caption hiển thị có đánh số + mục lục hình. (Nhận định cũ "Word alt-text yếu với ảnh học thuật" CHƯA kiểm chứng, không dùng.)
- **Đã CÂN NHẮC VÀ LOẠI:** (a) camera phát hiện hành động khả nghi — action recognition; (b) phân loại ảnh vào thư mục — classification/CLIP; (c) fine-tune QLoRA — T4 quá chật; (d) tiền xử lý dò biên cạnh — dự đoán làm giảm chất lượng (có thể làm thí nghiệm phụ).
- Các ý tưởng đều đã có người làm; đóng góp của đồ án là áp dụng + kết hợp + kiểm chứng trên mô hình mới, phần cứng sinh viên, dữ liệu biểu đồ khó.

## 4. Quyết định kỹ thuật đã chốt

- **Tiêu chí chọn mô hình:** mô hình chính phải thuộc dòng BLIP-2 hoặc LLaVA theo tiêu đề đề tài; trong đó chọn bản hiệu năng tốt nhất **vừa chạy được, vừa đã công bố, kiểm chứng**. Mô hình ngoài dòng chỉ dùng để so sánh.
- **Mô hình chính (Ch.3, Ch.4):** **LLaVA-OneVision-7B** [25] (TMLR 2025), `llava-hf/llava-onevision-qwen2-7b-ov-hf`. Bài gốc không có CIDEr trên COCO.
- **Mô hình đối chứng:** BLIP-2 OPT 2.7B [11] (ICML 2023, hướng VLM của đề tài); Qwen3-VL-8B [32] (preprint, đối chứng thế hệ mới).
- **Mốc so sánh, không cài đặt:** LLaVA-1.5 [13], Qwen3.8-27B [33], Gemini 4 Argon (đóng).
- **Chạy thử Colab T4, 4-bit nf4 (02/11/2026):** LLaVA-OV 5,7 GB nạp / 12,1 GB cao nhất, 7,0 s/ảnh; BLIP-2 2,8 GB, 0,6 s/ảnh; Qwen3-VL-8B 6,3 / 6,8 GB, 15,3 s/ảnh. Kết quả: `notebooks/ketqua/01_chay_thu.json`. Quan sát: LLaVA-OV mô tả biểu đồ chung chung không nêu số; BLIP-2 đọc sai chữ ("captain"); Qwen3-VL ghi B = 83% (thật 85%).
- **Dữ liệu:** COCO **val2017** (không dùng train2017 vì LLaVA-OV đã huấn luyện trên mô tả COCO118K); SciCap+ [40] trên Hugging Face `anselyang/SciCapPlus` (CC BY-NC 4.0, img.zip 18 GB → chỉ lấy vài chục hình qua đọc zip từ xa); biểu đồ tự vẽ có đáp án.
- **Metric:** CIDEr, CLIPScore, số lỗi sai dữ kiện (đếm tay). Không chọn caption bằng CLIP rồi lại chấm bằng CLIPScore.
- **Môi trường chạy:** **Google Colab** (có GPU). KHÔNG chạy mô hình trên MacBook (không có GPU NVIDIA/CUDA) — MacBook chỉ để đọc tài liệu + viết báo cáo.

## 5. Cấu trúc thư mục

```
ImageCaptioning/
├── README.txt          # hướng dẫn dùng + quy ước đặt tên
├── LINKS.txt           # link PDF + link code của 43 tài liệu
├── BaoCao_ImageCaptioning.docx / .pdf
├── notebooks/          # notebook Colab tự viết + ketqua/ (JSON kết quả)
├── PDF/                # PDF bài báo: [Mã]_[Năm]_[Tên].pdf
└── Code/               # repo của tác giả, mỗi tài liệu 1 folder (git clone --depth 1) — chỉ để đọc, không cần để chạy
```

Quy ước: mã `[n]` của file = số thứ tự tài liệu `[n]` trong danh mục (nhất quán giữa LINKS.txt, báo cáo, bảng so sánh). [1]–[28] là danh mục gốc, [29]–[43] bổ sung 11/2026.

Khi thêm tài liệu mới: thêm vào LINKS.txt, tải PDF vào `PDF/` đúng tên `[n]_[Năm]_[Tên].pdf`, clone code vào `Code/`.

## 6. Deliverable đã có

- **Doc Tổng quan tài liệu** (Claude Docs): 28 mục gốc có chú thích, chia 6 nhóm; chưa có các mục [29]–[43].
- **BaoCao_ImageCaptioning.docx:** Bìa (đúng tiêu đề giảng viên) + Ch.1 + Ch.2 (Bảng 2.1 sự thay đổi qua các thế hệ, Bảng 2.2 kỹ thuật, Bảng 2.3 điểm phù hợp) + Ch.3 (Hình 3.1, Bảng 3.1–3.2, công thức 3.1–3.2) + Tài liệu tham khảo. Trích dẫn dạng [n]. PDF cần xuất lại từ Word.
- **notebooks/01_chay_thu.ipynb:** chạy thử 3 mô hình ở 4-bit, đo bộ nhớ/thời gian.

## 7. Cấu trúc báo cáo (theo hướng dẫn môn học)

- Ch.1 Giới thiệu: bối cảnh, động lực (ý nghĩa KH + thực tiễn), phát biểu bài toán, đóng góp. **[ĐÃ VIẾT]**
- Ch.2 Công trình liên quan: phát triển + bảng so sánh + "đã làm gì / muốn làm gì" (bốn việc: kiểm chứng, 2A, 2B, 1A). **[ĐÃ VIẾT]**
- Ch.3 Phương pháp: nguyên lý LLaVA-OneVision theo sơ đồ hệ thống. **[ĐÃ VIẾT — bản nháp 03/11/2026]**
- Ch.4 Cài đặt & thử nghiệm: (1) so sánh LLaVA-OV / BLIP-2 / Qwen3-VL-8B trên COCO val2017, SciCap+, biểu đồ; (2) cấu hình A gốc / B +ngữ cảnh / C +tự kiểm tra / D cả hai cho LLaVA-OV; (3) demo công cụ Word. **[CHƯA VIẾT — cần chạy Colab trước để có số liệu]**
- Ch.5 Kết luận & hướng phát triển. **[CHƯA VIẾT]**

## 8. Tài liệu cốt lõi (mã [n] + arXiv)

- [11] BLIP-2 — arXiv:2301.12597 (ICML 2023) ← đối chứng
- [13] LLaVA-1.5 — arXiv:2310.03744 (CVPR 2024)
- [25] LLaVA-OneVision — arXiv:2408.03326 (TMLR 2025) ← mô hình chính
- [32] Qwen3-VL — arXiv:2511.21631 (preprint) ← đối chứng thế hệ mới
- [38] SigLIP — arXiv:2303.15343 (ICCV 2023); [39] Qwen2 — arXiv:2407.10671 (preprint) ← thành phần của LLaVA-OV
- [40] SciCap+ — arXiv:2306.03491 (J. NLP 2024) ← cơ sở cho 2A, nguồn dữ liệu figure
- [41] Summaries as Captions — arXiv:2302.12324 (INLG 2023) ← cơ sở cho 2A
- [42] Woodpecker — arXiv:2310.16045 (SCIS 2024) ← cơ sở cho 2B
- [16] POPE — arXiv:2305.10355 (EMNLP 2023)
- [18] COCO Captions — arXiv:1504.00325; [19] CIDEr — arXiv:1411.5726; [21] CLIPScore — arXiv:2104.08718
- [27] SciCap — arXiv:2110.11624; [28] Chart-to-Text — arXiv:2203.06486; [34] ChartCap (ICCV 2025); [35] ChartFI (IEEE VIS 2026)

## 9. Việc tiếp theo (gợi ý ưu tiên)

1. Notebook 02 — dữ liệu: ~100 ảnh COCO val2017 + 5 câu tham chiếu; 30–50 hình SciCap+ kèm caption gốc và đoạn văn nhắc đến hình; biểu đồ tự vẽ có/không ghi số.
2. Notebook 04 — đánh giá: CIDEr (pycocoevalcap, cần Java), CLIPScore tự viết theo [21], đếm lỗi sai dữ kiện. Thử trên vài ảnh trước.
3. Notebook 03 — sinh caption: 3 mô hình (cấu hình A) + LLaVA-OV cấu hình B/C/D.
4. Notebook minh họa nguyên lý: in số ô AnyRes, số token ảnh, kích thước vector qua SigLIP/MLP.
5. Công cụ Word (1A), rồi viết Ch.4, Ch.5.

## 10. Quy tắc làm việc

- KHÔNG bịa tên bài, tác giả, kết quả, đường dẫn. Không chắc thì ghi rõ "chưa xác minh".
- Ưu tiên bản đã phản biện; phân biệt rõ preprint với bản hội nghị/tạp chí.
- Mọi phần rút trích phải ghi nguồn theo mã [n].
- KHÔNG commit dataset/checkpoint nặng lên Git (dùng .gitignore loại *.pth, *.bin, *.safetensors, datasets/, checkpoints/).
