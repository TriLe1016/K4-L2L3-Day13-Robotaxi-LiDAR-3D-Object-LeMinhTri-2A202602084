# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Người làm và provenance

- Mã phòng/ca: H210, ca sáng — **làm cá nhân** (không theo nhóm 3–4 người)
- Thành viên: 1 người, xem `TEAMMATES.md` (họ tên/MSSV; một người làm mọi vai ở cả ba lượt).
- Trạng thái: **`executed-by-group`** — chạy thật bằng runner của gói Student trên máy cá nhân (chỉ một người thực hiện; mã trạng thái giữ nguyên theo mẫu). Không dùng `provided-results`.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lê Minh Trí (2A202602084); 2026-10-02, 09:51–09:52 (+07) (UTC 02:51:36–02:52:02 theo `output/smoke.json`); Linux x86_64 (`amd64`), Docker Engine 29.8.0, 16 CPU / 15 GB RAM, Python 3.12.3. Container bị giới hạn 4 CPU / 4 GB.
- Image tag và image ID; phiên bản repo:
  - Gói: `student-prelabel-amd64.zip` (release `student-prelabel-v1`). SHA256 `f58ca337…37aa9`, khớp `SHA256SUMS.txt`.
  - Image `day13-pointpillars:lc-20261001-amd64`, ID `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`, linux/amd64.
  - Code trong image: `repo_revision 0831856d921609312d42c7582c366e5a311bb7b1` (`working_tree_dirty: true` theo smoke.json); `preannotate.py` sha256 `65edf6ac…eb5ca`.
  - Repo Student đã clone: commit `e226b93`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - `input/demo.pcd` (KITTI 000008 đã chuyển đổi, CC BY-NC-SA 3.0), frame_id `demo`, 17 238 điểm.
  - sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
  - Chạy trên laptop cá nhân theo gói Student. Không có dữ liệu Robotaxi.
- Checkpoint: PointPillars KITTI có sẵn trong image, `/opt/PointPillars/pretrained/epoch_160.pth`, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window của checkpoint (không dùng `--full-scene`); score threshold 0.3; `--from KITTI`.
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - PCD chỉ có `x y z rgb`. Reflectance gốc đã bị bỏ, RGB = 0 là placeholder. Model dùng kênh hằng số của adapter, **không phải intensity thật**.
  - `z_ground = 0.075 m`, do script ước lượng từ PCD; giống nhau ở cả ba lượt.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh (`python3 student-bundle.py run --bundle . --out ../ket-qua-nhom-01`), `smoke.json` có `status: passed`.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ có 1 `vehicles` (x≈13.15, y≈−0.45), score 0.32, sát ngưỡng 0.3. Đáy hộp z≈−0.40 nằm **dưới** đường z=0 và dưới mặt đường cục bộ (≈0.06) trên ảnh Side. Ở x≈3–25 m có nhiều cụm điểm dạng xe nhưng không có hộp. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | 10 `vehicles`, 1 `two-wheels`, 2 `pedestrian`; score 0.32–0.93. Kích thước xe 3.1–4.2 × 1.5–1.7 × 1.5–1.8 m. Đáy phần lớn hộp gần mặt đường cục bộ (lệch −0.15…+0.21 m). Hộp xa (x≈34, 41, 56) chỉ có 13–40 điểm bên trong. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | **Cả 6 hộp đều là `pedestrian`, không còn `vehicles`.** Nhiều hộp nằm đúng vùng B gán xe, ví dụ C4 (13.24, −0.95) trùng vùng B1, C2 (9.11, 0.40) trùng vùng B0, nhưng kích thước chỉ ~1.0 × 0.7 × 1.7 m. |

Số điểm trong hộp và cao độ đáy so với mặt đường cục bộ do người làm tự tính thêm từ `input/demo.pcd` và các JSON. Mặt đường cục bộ lấy là percentile 5 của z trong vòng 2 m quanh footprint, nên chỉ là ước lượng thô, dùng để định hướng xem lại chứ không phải phép đo chính xác.

- **A/B — chỉ đổi delta:**
  - A có 1 hộp, B có 13 hộp. Trên `side-*-delta-0-*.png` chỉ có một hộp ở x≈11–15 m và hộp này cắm xuống dưới z=0. Trên `side-*-delta-1.73-*.png`, các hộp phủ hầu hết cụm điểm ở x≈2–27 m và vài cụm xa ở x≈32–58 m.
  - Lý do: checkpoint KITTI giả định sensor cao ~1.73 m, tức mặt đường nằm ở z≈−1.73 trong hệ model. Pipeline là `z_model = z_source − z_ground − delta`. Với delta=0, mặt đường rơi vào z_model≈0, cao hơn chỗ model "chờ" khoảng 1.73 m, nên model gần như không nhận ra vật.
  - Đây là **chạy lại model trên input khác**, không phải dịch hộp cũ. mean_z đổi từ 0.330 lên 1.034, chênh 0.70 m chứ không phải 1.73 m, và số hộp đổi từ 1 lên 13.
  - Hộp A0 (13.15, −0.45) gần hộp B1 (14.77, −1.08) nhưng không trùng: tâm lệch ~1.7 m theo x, yaw 2.67 so với −0.30, nên không coi A0 là "B1 dịch xuống".
  - Điều em còn chưa chắc: B nhiều hộp hơn và score cao hơn **không tự chứng minh B đúng**. Chưa có camera/ground truth để xác nhận từng đối tượng.
- **B/C — chỉ đổi pillar:**
  - B có 13 hộp, C có 6 hộp. Class chuyển từ 10 vehicles + 1 two-wheels + 2 pedestrian thành 6 pedestrian. Kích thước hộp nhỏ hẳn (L 0.6–1.1 m).
  - Ảnh `side-*-voxel-0.32.png` chỉ còn các hộp hẹp, cao. Vùng xe B0/B1/B4 ở x≈2–16 m không còn hộp xe. C4 và C2 nằm trong footprint xe của B1 và B0.
  - C0 (19.43, −8.07) có score cao nhất cả lượt (0.81) nhưng không có hộp tương ứng ở B. Ví dụ này cho thấy score cao không có nghĩa là đúng.
  - Diễn giải: checkpoint được train với pillar 0.16 m. Đổi sang 0.32 m làm thay đổi pseudo-image (mỗi ô gom gấp 4 lần diện tích), tức là đổi biểu diễn đầu vào mà model chưa từng học. Đây không phải lỗi quên cộng z ngược: mean_z của C (1.091) vẫn cùng thang với B (1.034).
  - Có đủ bằng chứng để kết luận C tốt hơn không? **Không.** Ngược lại, C mất toàn bộ class `vehicles` ở vùng có cụm điểm dạng xe rõ, nên không nên dùng C làm pre-label.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  - Chỉ chạy front-window của checkpoint, nên vật nằm ngoài ROI không phải bằng chứng model bỏ sót.
  - Ảnh Side là hình chiếu x-z toàn scene, các vật ở y khác nhau chồng lên nhau (ví dụ B8, B10 ở y≈4–5 m chồng lên B0 ở y≈1.2 m trong khoảng x≈7–11). Vì vậy Side không đủ để kết luận tâm, chiều rộng hay **yaw**, và không phân biệt được hướng 180°.
  - Đường z=0 trên Side chỉ là đường tham chiếu của plot. Mặt đường cục bộ ở x≈30–48 m cao lên ~0.25–0.45 m.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  - Không import JSON nào vào Robotaxi: đây là KITTI demo, khác frame.
  - Riêng trong B, các hộp cần xem lại bằng Top/Front và camera:
    - B8 `vehicles` (9.38, 4.23): đáy ở z≈0.65, cả vùng quanh đó cũng cao ~0.6–0.7, có thể là vỉa hè/bậc hoặc tường.
    - B10 `two-wheels` (10.32, 5.25): score 0.38, sát B8.
    - B11, B12 `pedestrian`: score 0.32–0.34, sát ngưỡng; B11 rộng 0.90 m nhưng dài chỉ 0.54 m.
    - B3, B6, B9: xe ở x≥33 m, chỉ có 13–40 điểm, kích thước và yaw khó xác nhận.
  - Toàn bộ A và C chưa đủ cơ sở (lý do ở trên).

## Ca QC có kiểm soát — không import CVAT

`output/qc-cases/` do helper `pipeline-qc-cases.py` tạo ra từ **prediction thật của B** (`source_prediction_sha256 16f30b08…f61`, khớp hash của run-B trong smoke.json). Đây là các biến đổi có chủ đích, có cờ `training_only: true`. **Không phải kết quả inference riêng, không phải nhãn đúng, không import CVAT.**

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không đổi trường nào so với B | Không phải lỗi; chỉ là bản sao prediction B, giữ phép chuyển z ngược. Không phải "cuboid đúng". | `case-correct.json` so trường-từng-trường với `run-B/boxes-…json`: không khác; `side-correct.png` giống ảnh Side của B |
| case-batch-z | 13 / 13 | −1.805 m (= delta 1.73 + z_ground 0.075) | Không đổi class/x/y/yaw/kích thước/score, chỉ đổi z | **Dừng batch.** Không sửa tay từng hộp; báo LC kiểm phép chuyển ngược (thiếu `+ z_ground + delta`) và tạo lại prediction từ pipeline đúng. | `side-batch-z.png`: mọi hộp tụt xuống, đáy trong khoảng −1.86…−1.16 m (B là −0.05…+0.65), trong khi điểm mặt đường ở z≈0. Cả 13 hộp lệch đúng một lượng, đúng bằng tổng hai hằng số của pipeline. |
| case-one-box-z | 1 / 13 (hộp 0, `vehicles` x≈8.09, y≈1.21) | −1.805 m (z 0.92 → −0.88) | Không đổi, chỉ z của hộp 0 | **Kiểm từng hộp.** 12 hộp còn lại không đổi nên pipeline không lệch toàn cục. Xem hộp 0 qua Top/Side/Front và camera rồi chỉnh đáy theo mặt đường cục bộ (~0.05). | `side-one-box-z.png`: chỉ một hộp ở x≈6–10 m chìm xuống −1.65…−0.11, các hộp khác giữ như B |

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục gồm:
- vai trò đã làm;
- một quan sát A/B/C có dẫn file hoặc hộp/vùng;
- diễn giải phép z thuận/ngược;
- một quyết định lỗi batch và hành động;
- điều chưa chắc.

Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

### Lê Minh Trí — 2A202602084 (làm cá nhân, một mình đảm nhận mọi vai)

- **Vai trò:** làm một mình cả 4 vai ở cả ba lượt A/B/C:
  - chạy runner;
  - kiểm cấu hình/JSON (`delta`, `voxel_size`, `z_ground`, hash trong `smoke.json`);
  - xem hình học trên ảnh Side;
  - ghi log và báo cáo.
- **Quan sát có dẫn file:**
  - `run-C/boxes-demo-delta-1.73-voxel-0.32.json` có 6 hộp, đều là `pedestrian`.
  - Hộp C4 (13.24, −0.95) và C2 (9.11, 0.40) nằm trong footprint của xe B1 và B0 ở `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, nhưng chỉ dài ~1.07 m.
  - Như vậy chỉ đổi pillar 0.16 → 0.32 mà checkpoint (train ở 0.16) đã đổi cả class lẫn kích thước. C không đáng tin hơn B dù có hộp score 0.81.
- **Phép z thuận/ngược:**
  - Thuận: `z_model = z_source − z_ground − delta`. Ngược khi xuất hộp: `z_source = z_model + z_ground + delta`. Ở đây z_ground = 0.075 m.
  - Ở A (delta=0), mặt đường nằm ở z_model ≈ 0 thay vì ≈ −1.73 như KITTI. Model nhận input khác nên chỉ ra 1 hộp, score 0.32, đáy cắm dưới đường.
  - Đổi delta là đổi input **trước** inference: model chạy lại, số hộp đổi từ 1 lên 13, mean_z chênh 0.70 m chứ không phải 1.73 m. Việc này khác với cộng hoặc trừ một lượng cho mọi hộp **sau** inference.
- **Quyết định lỗi batch và hành động:**
  - `case-batch-z`: cả 13/13 hộp lệch đúng −1.805 m (= delta + z_ground), class/x/y/yaw giữ nguyên. Đây là dấu hiệu quên phép chuyển ngược, nên **dừng, không sửa tay**, báo LC kiểm pipeline và tạo lại prediction.
  - `case-one-box-z`: chỉ hộp 0 lệch, nên **kiểm riêng hộp đó** bằng Top/Side/Front và camera.
- **Điều chưa chắc:**
  - Không có camera hay ground truth nên không xác nhận được hộp nào của B là đúng.
  - Nghi B8 (đáy z≈0.65, vùng quanh cao ~0.6–0.7 m) là vỉa hè/tường hơn là xe.
  - B10, B11, B12 có score sát ngưỡng 0.3. Xe ở x≥33 m chỉ có 13–40 điểm.
  - Ảnh Side không đủ để kết luận yaw (chồng vật theo y, không phân biệt được hướng 180°).

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
