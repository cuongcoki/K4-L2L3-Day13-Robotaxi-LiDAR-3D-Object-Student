# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Không có — làm cá nhân (1 thành viên)
- Thành viên: xem `TEAMMATES.md` (1 người, đảm nhận cả vai A/B/C).
- Trạng thái: `executed-by-group` (tự chạy trên laptop cá nhân bằng gói Student; không dùng máy LC, không dùng `provided-results`).
- Người thực sự chạy: Trần Đình Cương; ngày/giờ: 2026-10-03, 03:35–03:37 UTC (10:35–10:37 giờ VN); máy: Windows 11 Pro, Intel Core i5-10300H, **amd64**, Docker Desktop 29.7.2 (Linux containers, linux/x86_64), Python 3.12.10.
- Image: `day13-pointpillars:lc-20261001-amd64`, image ID `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82` (nạp từ `image.tar.gz` trong ZIP `student-prelabel-amd64.zip`); repo revision trong image `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD: `input/demo.pcd` (KITTI 000008 đã chuyển đổi, CC BY-NC-SA 3.0), frame_id `demo`, 17 238 điểm; input sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`. Chạy offline trong container (`--network none`, 4 CPU / 4 GB).
- Checkpoint: PointPillars KITTI có sẵn trong image, `/opt/PointPillars/pretrained/epoch_160.pth`, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window (cùng ROI cho cả 3 lượt); score threshold 0.3.
- Giả định kênh thứ tư/intensity: reflectance thật đã bị bỏ trong PCD, adapter dùng hằng số (RGB=0 placeholder) → không phải benchmark KITTI với intensity thật. z_ground ước lượng từ scan: **0.075 m**.

Kiểm kết quả: `output/smoke.json` có `status: passed`; mọi bước docker-load / run-A / run-B / run-C / qc-cases đều `passed`.

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | Chỉ 1 `vehicles` tại x≈13.2 m, y≈-0.45, score 0.32 (sát ngưỡng). Ảnh Side: đáy hộp chìm dưới mặt đất (≈ -0.4 m), trong khi rất nhiều cụm điểm xe ở x 3–25 m không có hộp. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`. Ảnh Side: đáy hộp xe gần z≈0–0.2 m, khớp cụm điểm ở x 3–56 m. Score xe 0.50–0.93; pedestrian/two-wheels thấp (0.32–0.38). |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | Cả 6 hộp đều `pedestrian` (dài 0.6–1.07 m, cao 1.7–1.8 m), không còn hộp `vehicles` nào; nhiều hộp nằm đúng vùng mà B là xe (x≈9–13 m). |

- **A/B — chỉ đổi delta:** A có 1 hộp; B có 13 hộp. Ảnh `side-…delta-0…` cho thấy cả đám mây điểm nằm thấp hơn hệ mà checkpoint KITTI được train (cảm biến cao ~1.73 m, mặt đất ≈ -1.73 m trong hệ LiDAR KITTI), nên model gần như không nhận ra xe; hộp duy nhất còn bị chìm dưới mặt đất. Ở B, input được dịch xuống đúng 1.73 m trước inference rồi kết quả được dịch ngược +1.73 m (cộng z_ground) để về hệ PCD, nên hộp ngồi lên mặt đất. Đây là **chạy lại model trên input khác**, không phải dịch hộp cũ: số hộp, vị trí x/y, class đều thay đổi (hộp duy nhất của A ở x=13.15 không trùng hộp nào của B; B gần nhất là x=14.77). Điều em còn chưa chắc: không có nhãn tham chiếu nên không khẳng định 13 hộp của B đều đúng — đặc biệt 2 pedestrian và 1 two-wheels có score < 0.4, và hộp xe ở x≈55 m có rất ít điểm.
- **B/C — chỉ đổi pillar:** B có 13 hộp; C có 6 hộp. Ảnh `side-…voxel-0.32…` chỉ còn hộp hẹp, cao. Lớp thay đổi hoàn toàn: B = 10 xe + 2 người + 1 xe 2 bánh; C = 6 người, 0 xe. Vị trí: nhiều hộp C (x≈9.1, 13.2, 13.2, 10.5 m) nằm trong vùng B có xe (x≈8.1, 9.4, 14.8 m) → model đang gọi một phần thân xe là người. **Không đủ bằng chứng để nói C tốt hơn**; ngược lại, checkpoint được train với pillar 0.16 m, đổi sang 0.32 m mà không train lại làm feature map lệch phân phối → C nên coi là cấu hình sai, không dùng để pre-label. Score cao hơn (0.81 ở C so với 0.34 pedestrian ở B) không chứng minh đúng hơn.
- **ROI và góc Side:** chỉ chạy front-window nên vật phía sau/hai bên ngoài cửa sổ không thể có hộp (rear=0 ở cả 3 lượt) — không được tính là "miss". Góc Side chỉ cho thấy x–z: kiểm được độ cao đáy/đỉnh và chiều dài, nhưng các hộp chồng lên nhau theo y (ví dụ x≈8–11 m) và không đọc được yaw/chiều rộng → cần thêm góc Trên (BEV) để đánh giá miss và yaw.
- **JSON nào chưa đủ cơ sở để import?** Không JSON nào ở đây được import (KITTI demo, không phải frame Robotaxi). Nếu đây là pipeline thật: A bị loại (lỗi hệ z cả batch); C bị loại (pillar không khớp checkpoint, đổi class toàn bộ); B là ứng viên duy nhất nhưng vẫn cần kiểm BEV + ảnh camera cho các hộp score thấp (pedestrian x≈18.7 và x≈34.0, two-wheels x≈10.3 nằm sát xe x≈9.4) và hộp xa x≈55.6 m.

## Ca QC có kiểm soát — không import CVAT

Các ca do helper `practice/pipeline-qc-cases.py` tạo từ prediction thật của B (`source_prediction_sha256` = `c2a8db24…` khớp run-B), offset chiều cao = delta + z_ground = 1.73 + 0.075 = **1.805 m**.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không | Không phát hiện lỗi transform; vẫn kiểm từng hộp như bình thường (đây là output model, không phải đáp án) | `case-correct.json` trùng B từng trường; `side-correct.png` giống `side-…delta-1.73-voxel-0.16.png` |
| case-batch-z | 13 / 13 | -1.805 m (đều nhau) | Không — chỉ z đổi; class, x, y, l/w/h, yaw giữ nguyên | **Dừng batch, kiểm pipeline**: tất cả hộp lệch cùng một lượng đúng bằng delta+z_ground → quên phép đổi ngược z, không phải lỗi từng vật | `side-batch-z.png`: mọi hộp chìm dưới đường z=0 (tâm ≈ -0.9 m); so JSON với B: Δz = -1.805 cho cả 13 hộp |
| case-one-box-z | 1 / 13 (hộp index 0, `vehicles` x≈8.09, y≈1.21) | -1.805 m | Không | **Kiểm riêng hộp đó** (nhiều view + camera), không dừng batch: 12 hộp còn lại vẫn khớp B | `side-one-box-z.png`; so JSON với B: chỉ index 0 có Δz = -1.805 |

Ghi chú: helper tạo biến đổi có chủ đích từ prediction B, **không phải kết quả inference riêng hoặc nhãn đúng**. Không import `case-*.json` vào CVAT.

## Nhận xét cá nhân

### Trần Đình Cương — 2A202602150

- **Vai trò:** làm một mình nên đảm nhận cả vận hành (chạy `student-bundle.py`), đọc JSON/cấu hình, xem hình học Side và ghi log cho cả ba lượt.
- **Quan sát A/B/C:** ở `run-A/side-demo-delta-0-voxel-0.16.png` chỉ có 1 hộp xe tại x≈13 m và đáy hộp chìm dưới z=0; sang B cùng vùng x 3–25 m có 7 hộp xe đáy gần mặt đất. Ở C (`voxel 0.32`) cả 6 hộp thành `pedestrian`, mất toàn bộ `vehicles`.
- **Phép z thuận/ngược:** PCD đặt mặt đất ≈ 0 (z_pcd = z_kitti + 1.73). Trước inference trừ delta: z_model = z_pcd − 1.73 để về hệ KITTI mà checkpoint quen. Sau inference cộng ngược: z_box_pcd = z_box_model + 1.73 (+ z_ground ước lượng 0.075). Quên bước ngược thì mọi hộp lệch xuống 1.805 m như `case-batch-z`.
- **Quyết định lỗi batch:** nếu mọi hộp lệch z cùng một lượng bằng delta + z_ground → dừng import cả batch, kiểm lại transform/config; nếu chỉ một hộp lệch → giữ batch, kiểm riêng hộp đó qua góc Trên/Trước/Bên và camera.
- **Điều chưa chắc:** không có ground-truth nên không biết 13 hộp của B có đủ/đúng không; các hộp score < 0.4 và hộp xa x≈55 m cần kiểm thêm; ảnh Side không cho kiểm yaw/chiều rộng.

> Phần trên do trợ lý AI soạn nháp từ output thật; học viên cần tự đọc lại, sửa bằng lời của mình và chịu trách nhiệm giải thích được trước LC.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
