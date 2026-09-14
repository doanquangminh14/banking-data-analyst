# Bộ dữ liệu Ngân hàng (Dirty Dataset) — Luyện tập DA / DS / DE

Dữ liệu **giả lập nhưng có cấu trúc quan hệ thực tế** của một ngân hàng, gồm 10 bảng,
tổng ~1.15 triệu dòng, được cố tình cài lỗi để bạn thực hành toàn bộ quy trình:
khám phá (EDA) → làm sạch (cleaning) → kiểm tra ràng buộc (data quality/DE) → phân tích/mô hình (DA/DS).

## 1. Sơ đồ quan hệ (ERD dạng text)

```
branches (1) ───< employees (N)
branches (1) ───< customers (N)
branches (1) ───< accounts (N)
customers (1) ───< accounts (N)
customers (1) ───< loans (N)
accounts (1) ───< cards (N)
accounts (1) ───< transactions (N)   (account_id = "từ tài khoản")
transactions (1) ─< fraud_alerts (N)
loans (1) ───< loan_payments (N)
exchange_rates: bảng tham chiếu tỷ giá theo ngày, không có FK trực tiếp
```

## 2. Danh sách bảng

| Bảng | Số dòng (xấp xỉ) | Khóa chính | Khóa ngoại |
|---|---|---|---|
| `branches.csv` | 203 | branch_id (có trùng cố ý) | — |
| `employees.csv` | 3,000 | employee_id | branch_id → branches |
| `customers.csv` | 50,750 | customer_id | branch_id → branches |
| `accounts.csv` | 70,280 | account_id (có trùng) | customer_id → customers, branch_id → branches |
| `cards.csv` | 45,000 | card_id | account_id → accounts |
| `loans.csv` | 15,000 | loan_id | customer_id → customers |
| `loan_payments.csv` | 150,450 | payment_id | loan_id → loans |
| `transactions.csv` | 802,400 | transaction_id (có trùng) | account_id → accounts, counterparty_account → accounts |
| `fraud_alerts.csv` | 8,000 | alert_id | transaction_id → transactions |
| `exchange_rates.csv` | 6,000 | (rate_date, currency) | — |

## 3. Các nhóm lỗi dữ liệu đã cài (để bạn luyện dò và xử lý)

**Định dạng không nhất quán**
- Ngày tháng: trộn lẫn `YYYY-MM-DD`, `DD/MM/YYYY`, `MM-DD-YYYY`, `DD-Mon-YYYY`, có timestamp, chuỗi rỗng, `0000-00-00`, ngày không hợp lệ (`31/02/...`).
- Số điện thoại: `0xxxxxxxxx`, `+84...`, `84...`, có dấu chấm/gạch ngang, sai số chữ số, rỗng.
- Email: thiếu `@`, có `@@`, có khoảng trắng thừa, rỗng.
- Giới tính: `Nam/Nữ/nam/NAM/Male/M/1/0/Khac/N/A...` — nhiều biến thể cho cùng 1 giá trị.
- Tên thành phố: `Hà Nội/Ha Noi/HN/hanoi...`.
- Tiền tệ: `VND/vnd/VNĐ`, `USD/usd`.
- Khoảng trắng thừa đầu/cuối chuỗi (`"  Tên  "`), chữ hoa/thường lộn xộn.

**Giá trị thiếu (NULL/blank)** rải rác ở hầu hết các cột "optional" (email, phone, balance, amount, salary, alert_type…).

**Trùng lặp**
- `branches`: vài `branch_id` bị lặp.
- `customers`: ~1.5% khách hàng bị đăng ký trùng với tên gõ sai (typo) — bài toán fuzzy matching / dedup.
- `accounts`: một số `account_id` bị load trùng (double-load).
- `transactions`: một số `transaction_id` trùng khóa chính, và một số dòng trùng hoàn toàn (double-ingestion).
- `loan_payments`: một số khoản thanh toán bị ghi nhận 2 lần.

**Vi phạm toàn vẹn tham chiếu (referential integrity)**
- Một số `accounts.customer_id`, `cards.account_id`, `transactions.account_id`,
  `loan_payments.loan_id`, `fraud_alerts.transaction_id` trỏ tới ID **không tồn tại** ở bảng cha (orphan FK).

**Lỗi nghiệp vụ / outlier**
- `accounts.balance`: một số âm bất thường (không hợp lệ với tài khoản thường).
- `customers.income_monthly`: một số âm, một số bị nhân nhầm 1000 lần (fat-finger).
- `transactions.amount`: giao dịch rút tiền (`Withdrawal`) có lúc để dương thay vì âm (lỗi dấu),
  có outlier do dư số 0, có giao dịch amount = 0.
- `loans.interest_rate`: một số âm, một số = 999 (giá trị canh gác/sentinel value).
- `cards`: một số `expiry_date` < `issue_date` (vô lý về logic).
- `exchange_rates.rate_to_vnd`: một số = 0, âm, hoặc NULL.

## 4. Gợi ý các việc có thể làm

- **DE**: viết pipeline chuẩn hoá định dạng (ngày, phone, email, category), validate FK,
  phát hiện & xử lý duplicate/orphan record, thiết kế star schema (fact `transactions`,
  dim `customers`/`accounts`/`branches`/`time`).
- **DA**: báo cáo số dư theo chi nhánh, phân khúc khách hàng, tỷ lệ nợ xấu, xu hướng giao dịch theo kênh.
- **DS**: mô hình phát hiện gian lận (dựa trên `fraud_alerts` + `transactions`), dự đoán khách hàng
  vỡ nợ (`loans` + `loan_payments`), phân cụm khách hàng theo hành vi giao dịch.

Toàn bộ file CSV dùng encoding `utf-8-sig` (mở tốt bằng Excel/pandas: `pd.read_csv(..., encoding='utf-8-sig')`).
