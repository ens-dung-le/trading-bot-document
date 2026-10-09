# Box Strategy 1h — cấu hình và danh mục nghiên cứu, 08/10/2026

Box 1h có kết quả tốt hơn scalping trong tập khảo sát. Đề xuất một cấu hình thử nghiệm có SL cho cả ba mã và đòn bẩy 5x; chưa có bằng chứng để gọi là tối ưu toàn cục hoặc phê duyệt chạy thật. Vốn tham chiếu 5.000 USDT; ngưỡng drawdown chọn cấu hình 10% là giả định nghiên cứu khi chưa nhận lựa chọn khác. Không thay đổi cấu hình database hoặc khởi động/deploy bot.

## Xác minh ảnh Job 102

Đã đọc snapshot Job 102 và run 395, rồi tái lập bằng BacktestRunnerService thật với cùng nến, warm-up, funding thực ghi nhận và execution configuration: 39 lệnh, net +406,668758 USDT (+8,1334%), PF 1,651861, max DD 3,464493%, Sharpe 1,339765. Kỳ 08/10/2025–08/10/2026 theo giờ UTC trong snapshot; cấu hình thực chạy 10x dù strategy_parameters.defaultLeverage ghi 2. execution_configuration.leverage.defaultLeverage=10 quyết định lần chạy.

Gate thất bại vì 39<100 lệnh và 10x ngoài dải 2–5x. Gate không xóa kết quả lãi; số lệnh ít vẫn hạn chế độ tin cậy. Không tự sửa gate để hợp thức hóa một cấu hình.

## Dữ liệu và cách chọn

Tín hiệu 1h, khớp lệnh/thoát lệnh bằng nến 15m, Box từ các ngày UTC hoàn chỉnh trước tín hiệu. SOL thiếu dữ liệu 15m ngày cuối, nên kỳ nghiên cứu chung kết thúc trước 07/10/2026 UTC. Bắt đầu 28/11/2024 để có đủ nguồn nến và 480 giờ warm-up cho cả ba. Không nội suy nến thiếu.

- Huấn luyện: [28/11/2024,15/05/2026), 533 ngày.
- Xác nhận: [15/05/2026,27/07/2026), 73 ngày.
- Kiểm tra cuối: [27/07/2026,07/10/2026), 72 ngày.
- Bảng gần một năm: [08/10/2025,07/10/2026), 364 ngày; không cùng mốc cuối với ảnh Job 102.

| Mã | Nến 1h / kỳ vọng | Nến 15m / kỳ vọng | Funding thực | Funding 8h giả định thêm |
|---|---:|---:|---:|---:|
| BTC | 16,272 / 16,272 | 65,087 / 65,088 | 34 | 2000 |
| ETH | 16,272 / 16,272 | 65,087 / 65,088 | 1130 | 904 |
| SOL | 16,272 / 16,272 | 65,087 / 65,088 | 1130 | 904 |

Phí: taker 0,06%/chiều, half-spread 0,01%/chiều, slippage 1 tick/chiều; cả TP và SL trả taker. Funding thực nơi có dữ liệu; mốc 8h thiếu giả định long trả 0,01% notional. Không xem BTC thiếu funding lịch sử là miễn funding. Stress: half-spread 0,02%, slippage 3 ticks, fee giữ nguyên. Latency metadata 80ms chưa được mô phỏng riêng trong fill engine.

64 cấu hình tín hiệu/đường thoát mỗi mã: 21 cấu hình có cấu trúc gồm baseline, các cặp SL/TP, long/short/both và TP-only không SL; 43 cấu hình lấy mẫu bằng seed 20261008. Cho phép thay ATR, Box lookback ngày, pivot trái/phải/lookback, tolerance, stop buffer, số tick buffer, độ dài wick/body và wick/range, vị trí close, số nến màu trước, stop cluster, entry buffer, hạn breakout/retest, hạn pending, TP/SL và hạn giữ. Không điều chỉnh phí/quy cách sàn như thể chúng là tham số tạo lợi nhuận.

Sàng lọc trên huấn luyện bằng score = net − 5000×maxDD, ngưỡng DD≤10%, không thanh lý, ít nhất 20 quan sát đóng. Chọn 5 cấu hình tốt, xem validation rồi chọn theo score_train + 2×score_validation. Sau đó thử 30 cặp sizing: leverage 2/3/5/10/20 × risk 12,5/25/50 USDT × margin cap 10%/30%. Chọn net_train + net_validation cao nhất với hai kỳ net dương, DD≤10%, không thanh lý, validation ít nhất 5 quan sát. Không dùng kỳ test để đổi cấu hình hoặc tỷ trọng. Đây là tìm kiếm hữu hạn, không chứng minh cực đại mọi giá trị có thể có.

Các hằng số thuật toán được mở thành optional overrides trong helper hiện hữu, mặc định giữ nguyên và Job 102 tái lập đúng; không viết bộ phát hiện Box thứ hai. Các overrides nghiên cứu chưa được thêm vào menu/schema cấu hình vận hành.

## Cấu hình được chọn trước khi đọc test

| Thông số | BTC | ETH | SOL |
|---|---:|---:|---:|
| Tín hiệu / thực thi | 1h / 15m | 1h / 15m | 1h / 15m |
| Hướng | Long | Long | Long |
| Đòn bẩy isolated | 5x | 5x | 5x |
| Risk trên tài khoản độc lập 5.000 USDT | 25 USDT | 50 USDT | 50 USDT |
| SL so với khoảng SL gốc | 1× | 1× | 2× |
| TP so với khoảng TP gốc | 1× | 1× | 1,25× |
| Minimum net RR trước funding | 1,5 | 1,5 | 1,5 |
| Margin cap | 30% | 30% | 30% |
| Notional cap tham chiếu | 10.000 USDT | 10.000 USDT | 10.000 USDT |
| Hạn giữ | Không giới hạn | Không giới hạn | Không giới hạn |

Tín hiệu chung của cả ba trở lại các giá trị gốc: ATR14; Box ngày trước; pivot lookback 288 nến, trái/phải 2/2; tolerance=max(2tick,0,2ATR); stop buffer=max(2tick,0,1ATR); wick≥body và wick/range≥40%; close ở nửa trên; màu đối nghịch trong 3 nến trước; stop cluster 3 nến; entry trên high tín hiệu 1 tick; pending tối đa 3 nến 1h, cắt ở 00:00 UTC; breakout/retest tối đa 12 nến 1h. Không trailing, break-even hay DCA.

Định nghĩa giá cho long: E=high nến tín hiệu+1tick; S0=min(low 3 nến)−stop buffer. T0=biên trên Box−tolerance với reversal; với retest là pivot high đã xác nhận gần nhất−tolerance. BTC/ETH: S=S0, T=T0. SOL: S=E−2(E−S0), T=E+1,25(T0−E). Làm tròn theo tick bằng helper hiện hữu. Đây là khoảng giá cấu trúc, không phải SL −2% hoặc TP +1,25% cố định. Giá khớp có thể vượt trigger vì gap/spread/slippage, do đó rủi ro thực có thể vượt ngân sách.

## Kết quả riêng từng mã

Mỗi dòng là tài khoản tham chiếu độc lập 5.000 USDT; không cộng ba dòng thành lợi nhuận một tài khoản. Net bao gồm market-close giả định ở cuối mẫu để định giá các vị thế chưa chạm TP/SL.

| Mã | Net 364 ngày | Return | Max DD | Net test chỉ lệnh đã đóng | P&L close giả định cuối test | Tổng test | Net stress test |
|---|---:|---:|---:|---:|---:|---:|---:|
| BTC | 578.66 | 11.57% | 2.88% | -32.49 | 0.00 | -32.49 | -34.60 |
| ETH | 945.49 | 18.91% | 6.95% | 73.22 | 70.41 | 143.62 | 135.96 |
| SOL | 1779.26 | 35.59% | 3.67% | 431.04 | 24.03 | 455.07 | 448.09 |

BTC lỗ ở kỳ test, dù lãi trên năm gần nhất. ETH và SOL dương; mỗi mã chỉ có 5 lệnh TP/SL đã đóng ở kỳ test. ETH và SOL mỗi mã còn một vị thế ở cuối kỳ được ghi riêng, không gọi là TP hoặc lệnh thắng đã thực hiện. Không lấy số ít này để đảm bảo lợi thế lâu dài.

## Phân bổ Markowitz từ P&L của chiến lược

Dùng daily P&L / 5.000 từ train+validation, 606 ngày, không dùng daily return buy-and-hold của coin. Không lãi cash trong mô hình (rf=0). μ=365×mean(r); Σ=365×sampleCov(r). Chọn w≥0, Σw_i=1, tối đa Sharpe=(wᵀμ)/sqrt(wᵀΣw). Với nghiệm dương của mẫu này, có thể tái lập bằng w∝Σ⁻¹μ rồi chuẩn hóa; dữ liệu ma trận có trong portfolio.json. Chọn mục tiêu Sharpe để cân bằng lợi nhuận và rủi ro; đây không phải tối đa lợi nhuận tuyệt đối bằng cách dồn vốn vào một mã.

| Mã | Tỷ trọng ngân sách | Vốn 5.000 USDT | Risk/lệnh sau phân bổ | Margin cap ban đầu |
|---|---:|---:|---:|---:|
| BTC | 36.614% | 1830.72 | 9.15 | 549.22 |
| ETH | 28.396% | 1419.81 | 14.20 | 425.94 |
| SOL | 34.989% | 1749.47 | 17.49 | 524.84 |

Các ngân sách là ba sleeve, không phải dùng toàn bộ làm ký quỹ. Tổng initial margin cap ban đầu 1.500 USDT (30% tài khoản); ít nhất 3.500 USDT còn ngoài ký quỹ nếu chạm tất cả cap. Tổng risk dự kiến khi cả ba có một lệnh ≈40,85 USDT, tức 0,817% tổng vốn trước funding/gap. Risk đã gồm fee/spread/slippage dự kiến tới SL; actual fill và funding vẫn có thể vượt mức này. Notional cap và risk cũng nhân với w_i, rồi làm tròn hợp đồng; đã chạy lại bằng engine thật ở đúng số vốn, không chỉ nhân lợi nhuận cũ với tỷ trọng. Mô phỏng phân tách ngân sách theo mã, chưa xác minh bộ quản lý vốn chung trong bot khi triển khai đồng thời.

Fit Sharpe ≈2,044; lợi suất số học thường niên hóa mẫu ≈14,06%; volatility năm ≈6,88%. Đây là ước lượng trong mẫu đã chọn tham số, không phải dự báo lợi nhuận năm tới. Tương quan daily P&L BTC–ETH≈0,258, BTC–SOL≈0,258, ETH–SOL≈0,343; không đồng nghĩa coin có tương quan thấp.

## Kết quả danh mục sau làm tròn hợp đồng

| Kỳ | Net lệnh đã đóng | P&L đóng giả định cuối kỳ | Tổng net/MTM đã trừ closing cost | Return | Max DD 15m | Lệnh đóng tự nhiên | Vị thế cuối kỳ |
|---|---:|---:|---:|---:|---:|---:|---:|
| Huấn luyện | 937.48 | 0.00 | 937.48 | 18.75% | 6.25% | 150 | 0 |
| Xác nhận | 226.45 | 0.00 | 226.45 | 4.53% | 2.27% | 29 | 0 |
| Test 72 ngày | 159.88 | 28.31 | 188.18 | 3.76% | 1.56% | 15 | 2 |
| Test tăng chi phí | 154.53 | 27.28 | 181.81 | 3.64% | 1.50% | 15 | 2 |
| Gần năm 364 ngày | 1067.43 | 28.31 | 1095.74 | 21.91% | 2.00% | 106 | 2 |

Không có liquidation mô phỏng. Max DD trên train≈6,25%, validation≈2,27%, năm gần nhất≈2,00%; dưới ngưỡng mẫu 10%, không bảo đảm DD tương lai. Khi kết hợp equity các sleeve, dùng timestamp 15m hợp nhất và giữ mark mới nhất nếu một mã thiếu nến; không tự tạo giá khớp trong gap. Điểm equity cuối được điều chỉnh bằng settlement để tính closing fee, tránh bỏ qua chi phí của vị thế chưa đóng.

## Ví dụ mức giá — chỉ là giao dịch lịch sử

| Mã | Ngày entry UTC | Entry trigger | SL | TP |
|---|---|---:|---:|---:|
| BTC | 24/09/2026 | 83630.100 | 83055.200 | 87136.100 |
| ETH | 24/09/2026 | 2648.480 | 2624.500 | 2782.910 |
| SOL | 05/10/2026 | 119.607 | 117.969 | 122.710 |

Các mức này minh họa cách bot tính SL/TP theo tín hiệu; không phải lệnh vào hiện tại. Không lấy giá của giao dịch cũ làm giá vào ngày 08/10/2026.

## Độ tin cậy và sử dụng kết quả

Cấu hình SOL khá nhạy với cửa sổ huấn luyện: lượt chẩn đoán trước khi chỉnh kỳ dữ liệu chung đã chọn SL hẹp hơn; trên kỳ đủ warm-up/thực thi chung, lựa chọn là SL×2. Kết quả mới là kết quả chính thức; sự thay đổi này cho thấy chưa nên xem một điểm tối ưu là ổn định. Không chọn lại theo test để làm đẹp.

Ảnh Job 102 đã xem toàn bộ năm gần nhất; kết quả kiểm tra theo thời gian trong chính năm đó không phải holdout hoàn toàn chưa từng xem. 64 cấu hình × 3 mã và nhiều sizing tạo selection bias. Cần một kỳ paper tiến về phía trước, với cấu hình/tỷ trọng cố định trước kỳ, để xác nhận. Phiên bản Box trong bot có cơ chế censor vị thế cuối mẫu; nghiên cứu này dùng hypothetical market-close và tách riêng các vị thế đó để không bỏ qua unrealized P&L khi chọn thông số.

Bootstrap khối tròn 14 ngày trên P&L ngày của test danh mục, 10.000 lần, seed 20261008: khoảng 95% cho tổng P&L 72 ngày ≈[−7,18; +397,78] USDT, gồm terminal mark-close. Khoảng vẫn chạm vùng lỗ. Phụ thuộc giả định khối 14 ngày và mẫu ít; không phải bảo đảm hoặc dự báo. Đây là cơ sở để ưu tiên paper thay vì gọi cấu hình đã tối ưu đủ chắc cho tiền thật.

Funding dài hạn thiếu, nhất là BTC, được giả định trả +1bp/8h; mark-price, liquidation phí thật, tick/orderbook, latency và market impact chưa có đầy đủ. SL không đảm bảo loss cap khi gap. TP/SL cùng nến dùng SL trước; liquidation được ưu tiên. Mọi thống kê và giới hạn chỉ đúng với mô hình thực thi được ghi ở đây.

## Kiểm chứng và tái lập

- Job 102 tái lập chính xác net/trades/PF/DD/Sharpe; chưa chạy benchmark random-entry.
- 41 tests Box/model đã qua, gồm default parity, tùy biến ATR/pivot không đọc tương lai, yêu cầu đủ các ngày Box, và giới hạn risk/notional/net RR. TypeScript typecheck đã qua.
- Chạy `npx tsx scripts/research-box-strategy/optimize-1h.ts` để tái lập tìm kiếm; kết nối DB chỉ đọc. Có thể đổi mục tiêu DD bằng BOX_RESEARCH_DRAWDOWN_LIMIT.
- portfolio.json lưu weights, μ, Σ, covariance/correlation, giao dịch và equity 15m của replay theo vốn thật phân bổ. Dùng BOX_RESEARCH_PORTFOLIO_FILE trỏ file này khi chạy cùng công cụ để tái lập các replay phân bổ, không sàng lọc lại.
- Tổng hợp danh mục: sum equity từng sleeve theo timestamp; PF chỉ lấy lệnh đã đóng tự nhiên; terminal positions ghi riêng. Bootstrap theo P&L ngày trong portfolioSummary.test.dailyEquity.
- results.json lưu 64 ứng viên, shortlist và 30 sizing/mã, lựa chọn đã khóa trước test, funding assumptions qua coverage, hash dữ liệu và các giao dịch.
- Không ghi lệnh/configuration vào database, không deploy hoặc khởi động lại bot. Chỉ thêm tùy biến helper nghiên cứu, tests và công cụ/báo cáo.

## Tài liệu định lượng

- Markowitz (1952), Portfolio Selection: https://doi.org/10.1111/j.1540-6261.1952.tb01525.x
- Bailey, Borwein, López de Prado & Zhu, The Probability of Backtest Overfitting: nguy cơ chọn cấu hình tốt nhất sau nhiều thử nghiệm trên cùng dữ liệu.
- Efron & Tibshirani, An Introduction to the Bootstrap: bất định ước lượng; chuỗi thời gian dùng bootstrap khối.
- Logic Box và fill/risk dùng trực tiếp mã nguồn repository, không dựa vào một giả định rằng mô hình nến bảo đảm lợi nhuận.
