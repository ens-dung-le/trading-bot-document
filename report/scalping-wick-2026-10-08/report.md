# Cấu hình scalping BTC / ETH / SOL — 08/10/2026

Không tìm được cấu hình có lợi nhuận dương trong tập nghiên cứu. Đề xuất phân bổ thực thi: BTC 0%, ETH 0%, SOL 0%; giữ 5.000 USDT vốn dự phòng không mở vị thế. Đây là tối đa hóa lợi nhuận ước lượng trong tập ứng viên đã khảo sát khi cho phép không giao dịch, không phải kết luận mọi chiến lược scalping đều thất bại. Giữ vốn dự phòng có rủi ro lưu ký/USDT; mô hình không ghi nhận lãi từ khoản này.

## Phạm vi và tính tương thích

Giữ nguyên tín hiệu scalping-lower-wick-reclaim-v1: ATR(14)% lớn hơn P75 của 1.440 phút trước, yêu cầu 95% dữ liệu tham chiếu hợp lệ, sweep/reclaim đáy 20 nến, wick >= 2×max(body,tick), wick/range >= 50%, close-position >= 75%. Long sau nến tín hiệu đã đóng, tại giá mở cửa phút kế tiếp có spread/slippage, giá khớp còn trên đáy được reclaim. Một vị thế/mã, không DCA, thoát tối đa 60 phút. Không chọn các mức giá BTC/ETH/SOL cố định vì mức vào thay đổi theo tín hiệu.

V1 khóa schema: 10x isolated, 50 USDT margin/lệnh, 60 phút. TP=0,2% giá thực khớp; không SL giá. TP/SL là hardcode trong profile. Đòn bẩy 2/3/5x và các TP/SL mới là giả thuyết nghiên cứu, không phải JSON hợp lệ để nhập trực tiếp vào V1. TP là return trên giá, không phải margin: ở 10x, TP giá 0,8% tương đương margin +8% gross. Không sửa hoặc kích hoạt cấu hình bot đang chạy.

## Dữ liệu và thiết kế

Nến futures KuCoin 1m trong [08/10/2025 00:00 UTC, 08/10/2026 00:00 UTC), 365 ngày. Cộng 1.455 phút warm-up trước kỳ. Khung ngày được giữ UTC; không dùng dữ liệu trong ngày 08/10/2026. Có dữ liệu ở cả 365 ngày mỗi mã; không nội suy khoảng trống.

| Mã | Nến 1m trong kỳ | Thiếu / 525.600 | Tín hiệu gồm warm-up | Funding thực trong kỳ |
|---|---:|---:|---:|---:|
| BTC | 520,777 | 4,823 (0.918%) | 672 | 37 |
| ETH | 521,948 | 3,652 (0.695%) | 909 | 1067 |
| SOL | 521,897 | 3,703 (0.705%) | 804 | 1067 |

Chia theo thời gian, không xáo trộn: huấn luyện 219 ngày [08/10/2025,15/05/2026), validation 73 ngày [15/05/2026,27/07/2026), kiểm tra cuối kỳ 73 ngày [27/07/2026,08/10/2026), tất cả UTC. Không mở lệnh trong giờ cuối mỗi phần để tránh giao dịch xuyên ranh giới.

16 ứng viên đã ấn định: TP giá 0,2% / 0,3% / 0,5% / 0,8% × SL không có / 0,2% / 0,3% / 0,5%; 10x; margin 50 USDT/lệnh, notional tối đa 500 USDT, dự phòng giá 0,5% và làm tròn số hợp đồng xuống theo từng mã. Chọn lợi nhuận ròng huấn luyện cao nhất trước khi đọc validation/test, sau đó giữ nguyên; không đổi lựa chọn để làm đẹp kết quả kiểm tra. Số tiền margin thực có thể dưới 50 do quy cách hợp đồng.

Chi phí theo mã nguồn: taker 0,06% mỗi chiều; half-spread 0,01% mỗi chiều; slippage 1 tick mỗi chiều. TP mô phỏng market exit, không tự giả định maker fee. Stress: half-spread 0,02% và slippage 3 ticks mỗi chiều, phí giữ nguyên. Không mô phỏng latency riêng vì bộ fill hiện tại không áp dụng nó, dù costModel có metadata 80ms. Funding dùng dữ liệu thực; ở các mốc 8h thiếu dữ liệu giả định trả +0,01% notional, không coi funding thiếu là bằng 0. Đặc biệt BTC chỉ có funding từ 24/09/2026; đây là một giới hạn đáng kể.

Ưu tiên liquidation → SL → TP; cùng nến chạm SL và TP dùng SL; gap dưới SL dùng giá mở cửa bất lợi. Đến hạn 60 phút ưu tiên thoát tại giá mở cửa đầu tiên có thể thực thi, không giả định TP trong nến đó đến trước hạn. Trường hợp gap trong thời gian giữ lệnh được đếm riêng, nên stop/60 phút không đảm bảo giới hạn lỗ khi dữ liệu/giá bị gián đoạn. Liquidation dùng ước lượng repository từ giá giao dịch, chưa có chuỗi mark price thực.

## V1 nguyên bản

| Mã | Lệnh train / validation / test | Net train | Net validation | Net test | PF test |
|---|---:|---:|---:|---:|---:|
| BTC | 342 / 105 / 69 | -253.74 | -66.06 | -33.24 | 0.324 |
| ETH | 455 / 167 / 93 | -356.99 | -128.16 | -65.60 | 0.261 |
| SOL | 436 / 145 / 77 | -435.79 | -113.98 | -9.62 | 0.699 |

P&L tính bằng USDT, mỗi mã tham chiếu margin tối đa 50 USDT/lệnh trên tài khoản 5.000 USDT. Đây là P&L giao dịch sau phí/funding, không phải lợi suất buy-and-hold.

## Ứng viên ít lỗ nhất trên huấn luyện — không khuyến nghị triển khai

| Mã | TP giá | SL giá | Net train | Net validation | Net test | Lệnh test | PF test | Drawdown test, USDT | Net stress test |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| BTC | 0.8% | 0.5% | -196.56 | -61.82 | -36.44 | 69 | 0.578 | 37.70 | -46.87 |
| ETH | 0.5% | Không có | -274.44 | -104.34 | -59.70 | 87 | 0.518 | 72.97 | -75.61 |
| SOL | 0.8% | 0.3% | -314.97 | -141.12 | -81.82 | 84 | 0.370 | 88.42 | -87.71 |

PF = tổng lãi ròng / trị tuyệt đối tổng lỗ ròng. PF < 1 biểu thị tổng lỗ vượt tổng lãi. Drawdown dựa trên equity đánh dấu theo nến, gồm chi phí đóng vị thế giả định, không chỉ các giao dịch đã chốt. Các ứng viên này đã thất bại ngay trên train; test được trình bày để đánh giá tiếp, không phải giấy phép triển khai. ETH không SL là kết quả ít lỗ trong tập train, không phải khuyến nghị bỏ SL.

## Độ nhạy đòn bẩy

Cùng margin 50 USDT/lệnh và cùng TP/SL đã chọn; đòn bẩy cao tăng notional và P&L tuyệt đối, không tự tạo lợi thế. Quy cách hợp đồng có thể khiến quan hệ không tuyến tính.

| Mã | Net test 2x | 3x | 5x | 10x |
|---|---:|---:|---:|---:|
| BTC | -6.10 | -7.60 | -17.09 | -36.44 |
| ETH | -9.54 | -15.79 | -28.78 | -59.70 |
| SOL | -16.00 | -24.08 | -40.50 | -81.82 |

Không chọn 10x chỉ vì margin return gross lớn hơn: tất cả các mức thử đều âm. Với cấu hình nghiên cứu có SL, rủi ro gần đúng = margin × leverage × (SL giá + chi phí), cộng gap; phải chuẩn hóa hợp đồng và kiểm tra rủi ro thanh lý. Không SL thì không thể dùng SL để tính loss budget.

## Phân bổ theo phân tích định lượng

Không dùng Beta và lợi suất ngày buy-and-hold của báo cáo Markowitz trước để phân bổ cho scalping: nguồn lợi nhuận cần tối ưu là P&L của chính tín hiệu/đường thoát lệnh sau chi phí. Dùng số P&L ngày của cấu hình đã chọn trên train, trung bình BTC −0,898, ETH −1,253, SOL −1,438 USDT/ngày ở margin 50 USDT/lệnh. Với hệ số exposure không âm a_i và quyền giữ vốn không giao dịch, lợi nhuận kỳ vọng mẫu Σ a_i μ_i được tối đa tại a_i=0 cho cả ba. Với mức ngại rủi ro không âm trong mục tiêu mean−variance, việc trừ thêm aᵀΣa cũng không làm một danh mục μ âm tốt hơn cash. Không cần ép nghiệm Markowitz fully-invested vào ba hệ thống thua lỗ. Đây là kết luận dựa trên mẫu và chi phí giả định; μ tương lai chưa biết.

| Hạng mục | Tỷ trọng vốn giao dịch khuyến nghị | Vốn, USDT |
|---|---:|---:|
| BTC | 0% | 0 |
| ETH | 0% | 0 |
| SOL | 0% | 0 |
| Dự phòng không mở vị thế | 100% | 5.000 |

Mục tiêu hiện tại là tránh triển khai cấu hình chưa có bằng chứng lợi thế. Chưa có bộ TP/SL/đòn bẩy để khuyến nghị chạy thật cả ba mã. Nếu mở lệnh nghiên cứu, cần một nhiệm vụ thay đổi schema/profile và một vòng xác nhận mới; không thể dán các TP/SL tùy ý vào V1 hiện tại.

## Bất định và kiểm chứng

Bootstrap khối tròn 7 ngày, 10.000 lần, seed 20261008, chuỗi P&L ngày test 73 ngày; khoảng 95% của tổng P&L 73 ngày (không phải khoảng dự báo bảo đảm):
- BTC: [-58.62, -14.12] USDT; 0 giao dịch gặp gap; 0 nến SL/TP mơ hồ; không có liquidation mô phỏng.
- ETH: [-108.80, -6.98] USDT; 1 giao dịch gặp gap; 0 nến SL/TP mơ hồ; không có liquidation mô phỏng.
- SOL: [-137.33, -34.91] USDT; 0 giao dịch gặp gap; 1 nến SL/TP mơ hồ; không có liquidation mô phỏng.

Bootstrap phụ thuộc giả định khối 7 ngày đại diện phụ thuộc thời gian; dữ liệu test 73 ngày còn hạn chế. Không phải bảo đảm thống kê lợi thế tương lai âm hoặc đủ an toàn. Kết quả chưa đo queue position, spread thực hay market impact; test đã được xem nên không được dùng lại làm holdout sau các lần chỉnh tiếp theo.

Đối chiếu V1 bằng BacktestRunnerService thật trên [08/09/2026,08/10/2026), cùng nến/quy cách/chi phí/funding giả định:
- BTC: 34 lệnh, thời điểm entry/exit khớp; sai lệch net tối đa 4.44e-16 USDT.
- ETH: 50 lệnh, thời điểm entry/exit khớp; sai lệch net tối đa 8.88e-16 USDT.
- SOL: 48 lệnh, thời điểm entry/exit khớp; sai lệch net tối đa 8.88e-16 USDT.

132 giao dịch đối chiếu tất cả khớp. 14 tests hiện hữu về tín hiệu/replay đã qua; TypeScript typecheck đã qua. Đối chiếu native trên 30 ngày kiểm chứng đường V1; các đường SL dùng chung decideExit/roundPriceUp và tests SL/TP/gap của V2/V3. Không thay đổi script chiến lược, tài khoản, lệnh hoặc database.

## Tái lập và tài liệu nền

- Công cụ: scripts/research-scalping-wick-configuration.ts; chạy `npx tsx scripts/research-scalping-wick-configuration.ts` với kết nối market-data trong .env. Công cụ chỉ SELECT database, lưu báo cáo local.
- results.json lưu ma trận ứng viên train, các giao dịch của cấu hình được chọn, P&L ngày, giả định, hash nến và kết quả đối chiếu. Không lưu credentials.
- Markowitz (1952), Portfolio Selection: https://doi.org/10.1111/j.1540-6261.1952.tb01525.x
- Bailey, Borwein, López de Prado & Zhu, The Probability of Backtest Overfitting: cảnh báo tối ưu nhiều cấu hình rồi báo điểm tốt nhất trên cùng dữ liệu.
- Efron & Tibshirani (1993), An Introduction to the Bootstrap; bootstrap khối dùng cho chuỗi có phụ thuộc thời gian.
- docs/scalping-lower-wick-reclaim-v1.md; src/strategies/scalping-lower-wick-reclaim-v1/{signal,parameters,profiles,strategy}.ts; src/risk/execution-cost.ts là nguồn logic và chi phí thực tế của nghiên cứu.
