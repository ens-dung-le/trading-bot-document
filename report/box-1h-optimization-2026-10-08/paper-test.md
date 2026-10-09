# Box 1h v2 — paper test đã kích hoạt

Tài khoản paper hiện tại: 1. Cấu hình: box-strategy v2, trạng thái running. Mã chính trong menu là ETH; BTC và SOL chạy cùng tài khoản qua hai luồng riêng, dùng chung sổ lệnh, số dư và bộ giới hạn rủi ro.

| Mã | Tỷ trọng vốn tham chiếu | Vốn tham chiếu USDT | Rủi ro dự kiến/lệnh USDT | Trần margin USDT | SL × | TP × |
|---|---:|---:|---:|---:|---:|---:|
| BTC | 36,6144% | 1.830,7192 | 9,1536 | 549,2158 | 1 | 1 |
| ETH | 28,3962% | 1.419,8081 | 14,1981 | 425,9424 | 1 | 1 |
| SOL | 34,9895% | 1.749,4727 | 17,4947 | 524,8418 | 2 | 1,25 |

Tổng vốn tham chiếu 5.000 USDT; trần margin danh mục 1.500 USDT. Đòn bẩy 5x isolated. Mỗi mã tối đa một vị thế hoặc một lệnh chờ vào. Các trần riêng cộng thành trần danh mục; hệ thống còn áp dụng giới hạn số dư, đòn bẩy, thanh lý và rủi ro hiện có. Không tự tăng vốn theo lãi. Số dư đã khôi phục lúc kích hoạt là 5.000,188429987922 USDT, gồm lợi nhuận lịch sử 0,188429987922 USDT; lịch sử được giữ nguyên.

Với entry E, stop cấu trúc S0 và target cấu trúc T0: BTC/ETH dùng S0/T0; SOL dùng SL=E−2(E−S0), TP=E+1,25(T0−E). Hệ số nhân khoảng cách, không phải phần trăm giá. Khối lượng được làm tròn xuống theo hợp đồng, tính sau khi điều chỉnh cả SL và TP, yêu cầu net RR tối thiểu 1,5 trước funding. Trần notional lần lượt 3.661,4384 / 2.839,6162 / 3.498,9454 USDT; trần margin tại 5x có thể chặt hơn.

Tín hiệu chỉ dùng nến 1h đã đóng. Lệnh chờ và SL/TP được mô phỏng trên nến 15 phút đã đóng. SL được ưu tiên nếu cùng nến có cả SL và TP. Lệnh không khớp nếu nến mở vượt mức invalidation; lệnh chưa chạm entry bị huỷ nếu mất stop. Giá khớp làm vượt trần margin cũng bị huỷ. Lệnh chờ tối đa ba nến 1h tính từ lúc tín hiệu có thể giao dịch, cắt tại 00:00 UTC. Long-only; không DCA, trailing, break-even hoặc giới hạn thời gian giữ vị thế.

Paper ghi nhận phí và mô hình giá khớp của bộ thực thi hiện có. Sổ paper này chưa tự hạch toán funding thực tế; cần tính funding bổ sung khi đối chiếu P&L với báo cáo nghiên cứu. Rủi ro dự kiến không bảo đảm giới hạn tổn thất khi gap hoặc giá khớp bất lợi.

Đã xác nhận 192 kiểm thử qua, build thành công, ba mã hoàn tất nạp lịch sử, trạng thái running, số dư được khôi phục và vị thế của luồng khác không thay đổi. Không ép mở lệnh khi kích hoạt; danh mục chờ tín hiệu mới.

Cấu hình chính xác: paper-config.json. Bản trước kích hoạt: paper-before-activation.json. Kết quả xác nhận: paper-activation.json. Các thay đổi sau này cần lưu vào cấu hình chiến lược trong database và áp dụng lại runtime; chỉ sửa bản JSON báo cáo không làm thay đổi bot đang chạy.
