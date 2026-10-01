# Báo cáo thực hành PointPillars — Day 13

Kiểm tra formative; không ghi điểm của người khác. Output KITTI (JSON/PNG/CSV, `smoke.json`) giữ tại thư mục nhóm `ket-qua-nhom-01/`, không đưa vào Git; báo cáo chỉ dẫn tên file và số liệu.

## Nhóm và provenance

- Mã nhóm/phòng: nhom-01
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group` (một lệnh `student-bundle.py run`, `smoke.json` có `status: passed`).
- Người chạy; ngày/giờ; máy: nhóm chạy trên một máy, vai trò theo `TEAMMATES.md`; 2026-10-01 08:17–08:18 UTC (15:17 giờ VN); macOS Apple Silicon, container Linux `arm64`, giới hạn 4 CPU / 4 GB.
- Image tag / image ID / repo: `day13-pointpillars:lc-20261001-arm64` / `sha256:dd6999ad5dd67962fdca18eb526132c08ce105980c9d89475d66cb1a5193f8a1` / `0831856d921609312d42c7582c366e5a311bb7b1` (`working_tree_dirty: true` theo `smoke.json`).
- PCD / frame_id / fingerprint: `demo.pcd` (17 238 điểm, KITTI 000008 đã chuyển đổi, CC BY-NC-SA 3.0) / `demo` / `input_sha256: 3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI `epoch_160.pth`, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front ROI; score threshold 0.3 (giữ nguyên cả ba lượt).
- Kênh thứ tư/intensity và z_ground: reflectance thật bị bỏ khỏi PCD, RGB = 0 là placeholder (không phải benchmark có intensity thật); `z_ground` ước lượng từ dữ liệu = 0.075 m, không phải mặt đường đo chính xác.
- Quy ước z: `z_model = z_source − z_ground − delta` (thuận, trước inference); `z_source = z_model + delta + z_ground` (ngược, sau inference).

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-…png`, `summary.csv` | Chỉ 1 hộp `vehicles` tại x=13.15, y=−0.45, score 0.322 (sát ngưỡng 0.3). |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-…png`, `summary.csv` | 13 hộp: 10 `vehicles`, 1 `two-wheels`, 2 `pedestrian`; score từ 0.933 xuống 0.318. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-…png`, `summary.csv` | 6 hộp, **toàn bộ là `pedestrian`**; không còn hộp `vehicles` nào so với B. |

Mỗi lượt chỉ đổi một biến so với lượt trước (A→B: `delta`; B→C: pillar XY); frame, checkpoint, score 0.3 và ROI giữ nguyên.

- **A/B — dịch input trước model có giống dịch output cùng hằng số không?** Không. Dịch sau inference chỉ tịnh tiến z của cùng tập hộp. Dịch trước model làm đổi điểm nào lọt vào pillar/dải z, đặc trưng z và độ khớp với anchor mà checkpoint KITTI đã học (mặt đất ở độ cao cảm biến ~1.73 m). Bằng chứng: A (1 hộp, score 0.322) và B (13 hộp) không chỉ khác z mà khác cả số lượng và vị trí (hộp A ở x=13.15 không trùng hộp nào ở B). Vì vậy `delta` là tham số đầu vào của inference, không phải bước hậu xử lý.
- **B/C — đổi pillar thấy gì? Đủ nói cấu hình nào tốt hơn?** Pillar 0.16→0.32 làm 13 hộp còn 6 và đổi hẳn thành phần class: 11 hộp xe/xe hai bánh biến mất, xuất hiện 6 hộp `pedestrian` (nhiều hộp ở vị trí không có trong B, ví dụ x=19.43, y=−8.07, score 0.808). Chưa đủ bằng chứng nói cấu hình nào tốt hơn: không có nhãn chuẩn, và số hộp/score không chứng minh độ đúng. Chỉ có cơ sở nói 0.32 thay đổi kết quả mạnh, nên các hộp ở C cần kiểm lại bằng nhiều view trước khi tin.
- **ROI front và góc Side ảnh hưởng cách đọc miss/yaw?** ROI front cắt vật thể ngoài cửa sổ phía trước nên "không thấy hộp" có thể là ngoài ROI, chưa phải model bỏ sót. Ảnh Side chiếu theo một hướng nên đọc tốt z/chiều cao nhưng yaw và chồng lấn theo hướng nhìn dễ gây nhầm; không kết luận yaw sai/đúng chỉ từ một góc Side.
- **JSON nào chưa đủ cơ sở để import? Cần kiểm gì tiếp?** Cả ba JSON A/B/C là prediction của model chưa có nhãn chuẩn: A (1 hộp, score thấp) và C (6 hộp, lệch hẳn so với B) chưa đủ cơ sở; B là baseline nhưng vẫn cần rà hộp score thấp (0.385 `two-wheels`, 0.339 và 0.318 `pedestrian`). Cần kiểm class, tâm, kích thước, yaw, đáy cục bộ qua nhiều view, và không nhập prediction demo KITTI vào frame Robotaxi.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không | Không có lỗi; vẫn cần rà hộp như prediction thường | `qc-cases/case-correct.json`: bản sao B, z khớp B ở cả 13 hộp |
| case-batch-z | 13 / 13 | −1.805 m mỗi hộp | Không (chỉ z đổi) | Dừng batch, kiểm transform/pipeline | `case-batch-z.json` so với `case-correct.json`: cả 13 hộp cùng giảm z đúng 1.805 = delta 1.73 + z_ground 0.075 |
| case-one-box-z | 1 / 13 | −1.805 m (hộp đầu, `vehicles` score 0.933) | Không (chỉ z đổi) | Kiểm từng hộp, không dừng cả batch | `case-one-box-z.json`: 12 hộp còn lại trùng `case-correct.json` |

Các ca QC do helper tạo biến đổi có chủ đích từ prediction B (`qc-cases/manifest.json`, `training_only: true`), không phải kết quả inference riêng và không phải nhãn đúng. Không import `case-*.json` vào CVAT.

## Nhận xét cá nhân

**1. Trần Đăng Khang (2A202602189)**
- Vai trò: Người vận hành (A), Người ghi log (B), Người xem hình học (C).
- Quan sát: A cho 1 hộp (`run-A/summary.csv`), B cho 13 hộp (`run-B/summary.csv`) khi chỉ đổi `delta` 0 → 1.73.
- Phép z: thuận `z_model = z_source − z_ground − delta` đưa mặt đất về hệ KITTI trước inference; ngược `z_source = z_model + delta + z_ground` đưa hộp về hệ nguồn sau inference.
- Quyết định lỗi batch: với `case-batch-z`, cả 13 hộp đều lệch z −1.805 → dừng, kiểm bước chuyển ngược z, không import.
- Điều chưa chắc: B→C làm hộp xe biến mất, chưa rõ do pillar thô làm mất đặc trưng vật thể nhỏ hay do cách model chấm score.

**2. Lê Thanh Tùng (2A202602211)**
- Vai trò: Người kiểm JSON (A), Người vận hành (B), Người ghi log (C).
- Quan sát: đối chiếu `case-one-box-z.json` với `case-correct.json`, chỉ hộp đầu (`vehicles`, score 0.933) lệch z −1.805; 12 hộp còn lại trùng hoàn toàn, class/x/y/yaw không đổi.
- Phép z: quên phép ngược sẽ làm toàn bộ hộp thấp hơn đúng `delta + z_ground`, nên độ lệch cố định này là dấu hiệu lỗi transform.
- Quyết định lỗi batch: lỗi chỉ ở một hộp → không dừng cả batch, kiểm riêng hộp đó trên nhiều view rồi sửa/loại.
- Điều chưa chắc: ngưỡng 0.3 giữ lại hộp score 0.318 (`pedestrian`) ở B; chưa biết các hộp sát ngưỡng đó có đáng tin không.

**3. Nguyễn Công Thành (2A202602219)**
- Vai trò: Người xem hình học (A), Người kiểm JSON (B), Người vận hành (C).
- Quan sát: ở `run-C` (pillar 0.32) cả 6 hộp đều là `pedestrian` (score 0.808 → 0.301), không còn `vehicles`, trong khi `run-B` có 10 `vehicles`.
- Phép z: với `delta = 0` (lượt A) mặt đất của scan không được hạ về hệ KITTI nên phân bố z lệch so với checkpoint, khớp với việc A chỉ còn 1 hộp.
- Quyết định lỗi batch: nếu toàn bộ hộp lệch cùng một lượng z thì coi là lỗi pipeline, dừng và kiểm transform trước khi sửa từng cuboid.
- Điều chưa chắc: không biết 6 hộp `pedestrian` ở C là vật thể thật hay nhiễu do pillar thô.

**4. Đoàn Vĩnh Nguyên (2A202602201)**
- Vai trò: Người ghi log (A), Người xem hình học (B), Người kiểm JSON (C).
- Quan sát: `side-demo-delta-1.73-voxel-0.16.png` (B) với 13 hộp, mean_z 1.034 (`run-B/summary.csv`); `smoke.json` ghi A/B/C lần lượt 1/13/6 hộp, chạy lần lượt 13.4 s / 10.8 s / 12.3 s gồm cả khởi động.
- Phép z: ảnh Side chỉ đọc được cao độ; sai z toàn batch (−1.805) thấy được như cả cụm hộp chìm dưới mặt đất, còn một hộp lệch thì phải so với hộp lân cận.
- Quyết định lỗi batch: `case-batch-z` → dừng pipeline, không import CVAT; `case-one-box-z` → kiểm hộp đó. Mọi ca `case-*` chỉ để tập QC.
- Điều chưa chắc: reflectance thật bị bỏ (RGB=0) ảnh hưởng thế nào đến score; ROI front có thể che mất hộp ngoài phạm vi mà ảnh Side không cho thấy.
