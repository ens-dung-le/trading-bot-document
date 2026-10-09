# Phân tích Markowitz — 08/10/2026

Database chỉ có lịch sử BTC, ETH, SOL futures KuCoin. Theo lựa chọn của người dùng, phân tích 10 tài sản sử dụng hoàn toàn giá spot tổng hợp CoinGecko bằng USD, không trộn với giá futures và không ghi vào database.

Top 10 được chọn theo khối lượng giao dịch 24 giờ tại snapshot 08/10/2026 khoảng 10:46–10:50 (UTC+7), không phải tổng khối lượng cả năm. Có USDT, USDC, USD1; không loại stablecoin.

API công khai days=365 trả 364 điểm giá tại 00:00 UTC từ 09/10/2025 đến 07/10/2026 và một điểm trong ngày 08/10/2026. Đã loại điểm trong ngày, còn 363 lợi suất một ngày liên tiếp. Đây là cửa sổ gần một năm, chưa đủ 365 lợi suất. API từ chối days=366. Giá daily CoinGecko là giá tham chiếu tổng hợp tại 00:00 UTC, không phải OHLC close chính thức của một sàn.

## Công thức và giả định

- r(i,t) = P(i,t)/P(i,t-1) − 1.
- Lợi suất kỳ vọng ước lượng μ(i) = 365 × trung bình lợi suất ngày. Đây là trung bình số học thường niên hóa, không phải CAGR hoặc dự báo chắc chắn.
- Σ = 365 × ma trận hiệp phương sai mẫu (ddof=1); σ(i) = sqrt(Σ(i,i)).
- Beta(i) = Σ(i,BTC)/Σ(BTC,BTC). BTC là đại diện thị trường, không phải chỉ số crypto tổng hợp.
- Danh mục chỉ mua: 0 ≤ w(i) ≤ 1, tổng w = 1. Không đòn bẩy; không phí, staking, funding hay lãi stablecoin; rf=0.
- Tối thiểu phương sai: min wᵀΣw. Tối đa Sharpe: max (wᵀμ − rf)/sqrt(wᵀΣw). Beta dùng để diễn giải; Markowitz sử dụng toàn bộ Σ.

## Kết quả

| Tài sản | Khối lượng 24h, USD | μ năm | σ năm | Beta/BTC | Lợi suất thực tế kỳ |
|---|---:|---:|---:|---:|---:|
| USDT | 70,276,246,589 | -0.034% | 0.390% | 0.0020 | -0.035% |
| BTC | 35,914,679,347 | -26.816% | 44.643% | 1.0000 | -30.649% |
| USDC | 22,780,656,735 | 0.022% | 0.608% | 0.0006 | 0.020% |
| ETH | 16,458,000,473 | -32.072% | 63.336% | 1.2788 | -40.408% |
| SOL | 2,742,677,329 | -41.928% | 66.967% | 1.2982 | -47.322% |
| XRP | 2,220,075,506 | -43.329% | 67.191% | 1.2903 | -48.021% |
| NEAR | 1,091,392,141 | 108.934% | 106.739% | 1.2388 | 70.840% |
| USD1 | 980,951,811 | -0.025% | 0.613% | 0.0024 | -0.026% |
| DOGE | 861,459,561 | -75.180% | 71.500% | 1.2674 | -63.353% |
| BNB | 782,800,716 | -39.453% | 50.142% | 0.8877 | -40.417% |

### Tối thiểu phương sai

USDT: 59.6821%, USDC: 23.2619%, NEAR: 0.0036%, USD1: 17.0524%
μ=-0.0154%; σ=0.3352%; Beta=0.001762; Sharpe=-0.045894.

### Tối đa Sharpe

USDC: 94.3432%, NEAR: 5.6568%
μ=6.1833%; σ=6.0313%; Beta=0.070632; Sharpe=1.025189.

### Chia đều

USDT: 10.0000%, BTC: 10.0000%, USDC: 10.0000%, ETH: 10.0000%, SOL: 10.0000%, XRP: 10.0000%, NEAR: 10.0000%, USD1: 10.0000%, DOGE: 10.0000%, BNB: 10.0000%
μ=-14.9879%; σ=40.9832%; Beta=0.826618; Sharpe=-0.365709.

## Kiểm tra và giới hạn

Đã kiểm tra giá dương, hữu hạn; lịch ngày chung không thiếu ngày; ma trận hiệp phương sai xác định dương; tổng trọng số và giới hạn; chạy tối ưu từ 31 điểm khởi tạo và kiểm tra điều kiện KKT cho nghiệm phương sai tối thiểu. Tối đa Sharpe là kết quả tối ưu số trong mẫu; chưa kiểm chứng ngoài mẫu. Không thể suy ra hiệu quả đầu tư tương lai từ nghiệm trong mẫu.

Sai số μ có thể lớn: với NEAR, sai số chuẩn thường niên hóa xấp xỉ σ × sqrt(365/363) ≈107 điểm phần trăm nếu giả định lợi suất độc lập. Kết quả Sharpe tập trung vào NEAR rất nhạy với ước lượng này. Stablecoin có rủi ro mất neo, thanh khoản, tổ chức phát hành không được phản ánh đủ trong σ lịch sử. Chọn top 10 hiện tại tạo thiên lệch lựa chọn nếu dùng kết quả như backtest.

## Nguồn

- Markowitz (1952), Portfolio Selection, Journal of Finance: https://doi.org/10.1111/j.1540-6261.1952.tb01525.x
- Sharpe (1964), Capital Asset Prices: https://doi.org/10.1111/j.1540-6261.1964.tb02865.x
- CoinGecko markets: https://docs.coingecko.com/reference/coins-markets
- CoinGecko market chart: https://docs.coingecko.com/reference/coins-id-market-chart

analysis.json lưu snapshot xếp hạng, giá đầu vào, lợi suất ngày, ma trận Σ và kết quả chưa làm tròn.
