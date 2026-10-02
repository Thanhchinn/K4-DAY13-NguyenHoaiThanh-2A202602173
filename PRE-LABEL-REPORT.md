# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: H210 (Thực hiện cá nhân: Nguyễn Hoài Thanh - 2A202602173)
- Thành viên: xem `TEAMMATES.md` (Thực hiện cá nhân: Nguyễn Hoài Thanh - MSSV: 2A202602173; tự đảm nhiệm toàn bộ vai trò A/B/C: Vận hành runner & kiểm tra Docker container, Kiểm tra cấu hình tham số & trích xuất JSON/CSV, Đọc hình học Side-view & đối chiếu ca QC).
- Trạng thái: `executed-by-group` (thực hiện độc lập/cá nhân trên máy riêng)
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Hoài Thanh (cá nhân); 2026-10-02 19:37:34 (UTC 12:37:34); Windows 11 64-bit / WSL2 Docker Desktop 29.8.0 / Architecture amd64 (x86_64), 12 CPUs, 7.4 GB RAM Docker VM.
- Image tag và image ID; phiên bản repo: Image tag: `day13-pointpillars:lc-20261001-amd64`; Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; Phiên bản repo: `e226b934c656f23c0da70b1e12cbf365fb78dd82` (clean tree).
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: PCD: `input/demo.pcd` / frame_id: `demo` (mẫu KITTI 000008, 17238 điểm, giấy phép CC BY-NC-SA 3.0); Chạy local offline trên container Docker (network none); PCD SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: `/opt/PointPillars/pretrained/epoch_160.pth`; SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window (`x ∈ [0, 70.4], y ∈ [-40, 40], z ∈ [-3, 1]`); score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ tư/reflectance thật bị loại bỏ trong bản PCD demo, thay bằng placeholder hằng số (RGB=0); `z_ground` được ước lượng từ đám mây điểm (`z_ground = 0.075 m`) và cộng thêm độ dịch sensor `delta = 1.73 m` (tổng bù cao độ = 1.805 m).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-demo-delta-0-voxel-0.16.png`<br>`run-A/summary.csv` | Chỉ có đúng 1 hộp `vehicles` tại x=13.15m, score thấp (0.322). Do delta=0, đám mây điểm không được bù độ cao sensor 1.73m của KITTI, các cụm điểm xe nằm lệch khỏi phân phối z của mạng khiến model bỏ sót hầu hết xe (false negatives nặng). |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-demo-delta-1.73-voxel-0.16.png`<br>`run-B/summary.csv` | Phát hiện 13 hộp (10 vehicles, 1 two-wheels, 2 pedestrian). Các xe ở vùng gần và trung (x từ 3.7m đến 40.9m) có score cao (0.64 - 0.93), bao bọc tốt các cụm điểm trên side-view. Cao độ mean_z = 1.034m phản ánh đúng tâm thân xe so với mặt đường. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-demo-delta-1.73-voxel-0.32.png`<br>`run-C/summary.csv` | Phát hiện 6 hộp và 100% bị gán nhãn `pedestrian` (0 vehicles, 0 two-wheels). Cỡ pillar 0.32m làm giảm một nửa độ phân giải không gian XY, khiến các đặc trưng thân xe bị gộp thô, làm model pretrained trên grid 0.16m phân loại sai toàn bộ sang người đi bộ hoặc bỏ sót. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh/file/vùng `side-demo-delta-0-voxel-0.16.png` so với `side-demo-delta-1.73-voxel-0.16.png` khác ở dải x từ 3m đến 56m: ở A không xuất hiện hộp nào dọc đường đi ngoại trừ một hộp yếu ở x≈13.15m (score 0.322); sang B xuất hiện đầy đủ 10 hộp xe (score lên tới 0.93) bám khít các vệt điểm của thân xe. Đây là chạy lại model trên input khác (được bù cao độ sensor), không chỉ dịch hộp cũ; điều em còn chưa chắc là hộp duy nhất ở A có phải là sự trùng hợp ngẫu nhiên của 1 cụm điểm lọt vào ngưỡng kích hoạt hay tương ứng với thân xe ở x=14.77m của B.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh/file/vùng `side-demo-delta-1.73-voxel-0.32.png` so với B khác ở chỗ toàn bộ các hộp kích thước lớn (chiều dài 3.1m - 4.2m) của lớp vehicles đều biến mất hoàn toàn, chỉ còn các hộp nhỏ (chiều dài 0.5m - 1.0m) mang nhãn `pedestrian`. Số lượng/lớp/vị trí thay đổi như sau: tổng số hộp giảm hơn 50% (từ 13 xuống 6), class vehicles giảm từ 10 xuống 0, class pedestrian tăng từ 2 lên 6. Có đủ bằng chứng để kết luận tốt hơn không? Hoàn toàn KHÔNG tốt hơn; thực tế kết quả C bị suy giảm chất lượng nghiêm trọng do mismatch biểu diễn đầu vào với checkpoint pretrained.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Góc Side là hình chiếu trực giao 2D lên mặt phẳng x-z của toàn bộ không gian 3D, do đó các vật thể nằm cùng khoảng cách x nhưng khác tọa độ y (khác làn đường) sẽ bị chiếu đè lên nhau, gây nhầm lẫn về mật độ điểm và không thể đọc được kích thước bề ngang (width/dy) cũng như góc xoay (yaw). Hơn nữa, model chỉ chạy trên ROI cửa sổ phía trước (front-window: x ∈ [0, 70.4], y ∈ [-40, 40], z ∈ [-3, 1]), những điểm hay vật thể phía sau xe hoặc ngoài biên ROI sẽ không có bounding box, đây là giới hạn phạm vi quét chứ không phải model bỏ sót. Muốn đánh giá đúng miss và yaw, bắt buộc phải đối chiếu trên Top-view (BEV), Front-view và ảnh camera 2D đồng bộ.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Cả ba file JSON (A, B, C) đều CHƯA đủ cơ sở để import vào CVAT Robotaxi: (1) Đây là suy luận từ mô hình KITTI trên PCD demo chuẩn hóa, khác biệt về hệ tọa độ và đặc tính cảm biến với Robotaxi; (2) Ngay cả bản B (tốt nhất trong 3 lượt) vẫn có các hộp score thấp (pedestrian score 0.31-0.34, two-wheels score 0.38) cần xác minh xem có phải false positive; (3) Hộp dự đoán chỉ là pre-label gợi ý, cần kiểm tra thủ công trên công cụ gán nhãn 3D, đối chiếu cả 3 hình chiếu và ảnh RGB, căn chỉnh lại 7 tham số hình học (tâm x, y, z; kích thước l, w, h; góc yaw) và gán lại đúng class theo schema 5 lớp của bài học.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm từng hộp | `qc-cases/case-correct.json` và `qc-cases/side-correct.png`. Tọa độ và thuộc tính 13 hộp giống hệt Run B, z_center trung bình 1.034m, các hộp nằm sát cụm điểm mặt đường. |
| case-batch-z | 13 / 13 | -1.805 m | Không đổi | Dừng batch! | `qc-cases/case-batch-z.json` và `qc-cases/side-batch-z.png`. Tất cả 13 hộp đều bị kéo tụt xuống dưới mặt đất (lệch đúng `delta + z_ground = 1.73 + 0.075 = 1.805 m`). Đây là lỗi pipeline hệ thống (quên bước biến đổi ngược coordinate transform về hệ sensor), cần dừng ngay toàn bộ batch và báo kỹ sư sửa pipeline. |
| case-one-box-z | 1 / 13 | -1.805 m (tại hộp ID 0) | Không đổi | Kiểm từng hộp | `qc-cases/case-one-box-z.json` và `qc-cases/side-one-box-z.png`. Duy nhất hộp xe đầu tiên tại x=8.09m bị chìm xuống z=-0.88m, 12 hộp còn lại giữ nguyên cao độ chuẩn. Đây là lỗi cục bộ trên 1 đối tượng, không dừng batch mà kiểm tra và sửa riêng hộp đó trên các view 3D. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### Nguyễn Hoài Thanh (MSSV: 2A202602173)

- **Vai trò đã làm**: Trực tiếp giải nén gói Student, khởi chạy môi trường Docker container và thực thi runner tự động `student-bundle.py` trên máy tính cá nhân. Kiểm tra tính toàn vẹn của manifest, image hash và checkpoint hash. Theo dõi tiến trình 3 lượt suy luận (A, B, C) và quá trình sinh 3 ca QC có kiểm soát. Trích xuất, phân tích số liệu từ các tệp `summary.csv`, `boxes-*.json` và đối chiếu ảnh trực quan `side-*.png`.
- **Quan sát A/B/C**:
  - So sánh A và B: Khi thay đổi tham số `delta` từ 0m lên 1.73m (giữ nguyên pillar 0.16m), số lượng hộp phát hiện tăng đột biến từ 1 hộp lên 13 hộp. Ở lượt A, chỉ có 1 xe được nhận diện tại x=13.15m với score thấp (0.322); sang lượt B, mô hình nhận diện chuẩn xác 10 xe (score từ 0.64 đến 0.93), 1 xe hai bánh và 2 người đi bộ, với các hộp bọc khít cụm điểm mặt đất. Điều này chứng minh `delta` không phải là phép tịnh tiến hình học sau suy luận mà là phép chuẩn hóa cao độ đầu vào trước khi đưa vào mạng nơ-ron.
  - So sánh B và C: Khi tăng kích thước pillar XY từ 0.16m lên 0.32m (giữ nguyên delta 1.73m), số lượng hộp giảm từ 13 xuống 6 hộp, và toàn bộ 6 hộp này đều bị phân loại nhầm thành `pedestrian` (0 xe ô tô nào được nhận diện). Do checkpoint được huấn luyện trên lưới pillar 0.16m, việc đổi kích thước cột điểm làm méo dạng feature map trích xuất, dẫn đến suy giảm hiệu năng nghiêm trọng.
- **Diễn giải phép z thuận/ngược**:
  - Phép biến đổi thuận (chuẩn hóa dữ liệu đầu vào cho model):
    $$z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$$
    Trong đó $z_{\text{ground}} = 0.075\,\text{m}$ (ước lượng mặt đường cục bộ) và $\text{delta} = 1.73\,\text{m}$ (cao độ lắp đặt LiDAR theo chuẩn KITTI). Phép biến đổi này đưa đám mây điểm về hệ tọa độ mà mạng PointPillars pretrained đã học.
  - Phép biến đổi ngược (đưa hộp dự đoán trở lại hệ tọa độ cảm biến nguồn):
    $$z_{\text{source}} = z_{\text{model}} + z_{\text{ground}} + \text{delta}$$
    Nhờ bước này, bounding box xuất ra có tọa độ tâm và đáy khớp chính xác với đám mây điểm gốc của sensor.
- **Quyết định lỗi batch và hành động**:
  - Trong ca `case-batch-z`: Toàn bộ 13/13 hộp đều bị sụt lún đều đặn một khoảng đúng bằng $1.805\,\text{m}$ ($z_{\text{ground}} + \text{delta}$). Đây là bằng chứng rõ ràng của lỗi hệ thống (systematic pipeline bug - script quên thực hiện bước cộng z ngược). Quyết định: **DỪNG BATCH NGAY LẬP TỨC**, không tiến hành sửa thủ công trong CVAT vì sẽ lãng phí công sức sửa hàng trăm hộp bị lỗi hệ thống, báo cáo ngay cho LC/kỹ sư phụ trách pipeline để khắc phục script chuyển đổi tọa độ.
  - Trong ca `case-one-box-z`: Chỉ có duy nhất 1 hộp tại x=8.09m bị chìm xuống dưới đất trong khi 12 hộp còn lại nằm đúng vị trí. Quyết định: **KHÔNG DỪNG BATCH, TIẾP TỤC KIỂM TRA TỪNG HỘP**. Đây là lỗi ngoại lai/cục bộ trên đối tượng cụ thể; annotator sẽ dùng các góc chiếu Top, Side, Front và camera đối chiếu để nâng đáy hộp lên bám đúng cụm điểm mặt đất.
- **Điều chưa chắc**:
  - Chưa thể khẳng định độ chính xác của 2 hộp `pedestrian` (score 0.31-0.34) và 1 hộp `two-wheels` (score 0.38) ở lượt B nếu chỉ quan sát qua ảnh Side 2D. Cần phải kiểm tra trên giao diện 3D trực quan kết hợp ảnh chụp camera RGB đồng thời.
  - Tác động của việc lược bỏ kênh reflectance thật và thay bằng hằng số RGB=0 đối với độ nhạy của detector ở các khoảng cách xa (>40m).
  - Sự khác biệt về hình học và sensor montage giữa xe KITTI demo và cấu hình Robotaxi thực tế trong 30 job cá nhân tiếp theo.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:

