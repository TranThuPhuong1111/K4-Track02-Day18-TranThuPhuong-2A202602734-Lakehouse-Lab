# Reflection — Anti-pattern: Small files

Hệ thống mình quan tâm là pipeline log gọi LLM và trajectory của agent. Mỗi request sinh
một bản ghi nhỏ cần xuất hiện gần như ngay, nên writer dễ commit từng micro-batch thành
file vài chục KB. Đó là anti-pattern **small files**.

Ở NB6, 200 commit tạo 200 file trung bình 51.5 KB; point query phải mở mọi file.
Ở NB5, metadata bằng 284% dữ liệu: trả giá hai lần, cả data lẫn planning.
Sau compaction còn 11 file (18×), clustering bỏ qua 90% file.

Bất ngờ là dọn dẹp cũng có bẫy: compaction làm dung lượng tạm tăng (10.1 lên 16.1 MB) đến khi
VACUUM; VACUUM không thấy file do writer crash; expire snapshot Iceberg không tự xóa file.

Cách phòng tránh: gom batch ở writer với target 128–512 MB; lên lịch compaction và clustering
theo cột hay lọc; expiry luôn đi cặp orphan sweep có age guard; checkpoint định kỳ; theo dõi
số file và kích thước file trung bình như một SLO.

*Sử dụng AI: Claude Code hỗ trợ dựng môi trường, chẩn đoán lỗi, render screenshots và soạn
nháp RESULTS/reflection; mọi số liệu lấy từ lần chạy thật.*
