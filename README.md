# Phát hiện gian lận Bitcoin trên đồ thị dị thể (Elliptic++)

Dự án thử nghiệm phát hiện giao dịch và địa chỉ ví gian lận (illicit) trên bộ dữ liệu Elliptic++, kết hợp đặc trưng lan truyền kiểu Active Diffusion (lấy cảm hứng từ ADGNN) với Random Forest.

## Dữ liệu

- Bộ dữ liệu: [Elliptic++](https://github.com/git-disl/EllipticPlusPlus) (203K giao dịch, 49 mốc thời gian, ví địa chỉ, nhãn illicit / licit / unknown).
- Repo này **không** chứa dữ liệu. Tải từ link Google Drive trong README của Elliptic++ và đặt vào thư mục `data/`.

## Cách làm

1. **Chia tập theo thời gian:** train = time step 1-29, val = 30-34, test = 35-49 (tránh rò rỉ dữ liệu).
2. **Đồ thị:** mỗi time step là một đồ thị dị thể riêng (node giao dịch và ví; cạnh tx→tx, addr→tx, tx→addr).
3. **Nhãn thiếu:** node unknown vẫn nằm trong đồ thị, chỉ bị loại khỏi loss và metric bằng mask.
4. **Mất cân bằng lớp:** trọng số lớp `balanced`, tính riêng cho giao dịch và ví, chỉ trên tập train.
5. **Chuẩn hoá:** chỉ fit scaler trên train; giữ nguyên 165 đặc trưng gốc đã chuẩn hoá sẵn.
6. **Mô hình:** Logistic Regression, Random Forest, GCN, GNN dị thể kiểu RGCN (HeteroConv + SAGEConv), Temporal GNN (GRU).
7. **Đặc trưng Active Diffusion:** tính nghiệm dạng đóng qua chuỗi Neumann, không có tham số học, ghép với đặc trưng gốc rồi đưa vào Random Forest.
8. **Concept drift:** phân tích bằng rolling validation và biểu đồ đặc trưng theo time step.

## Kết quả (test, lớp illicit)

| Mô hình | Giao dịch P / R / F1 | Ví P / R / F1 |
|---|---|---|
| Logistic Regression | 0.149 / 0.922 / 0.256 | 0.071 / 0.903 / 0.132 |
| Random Forest | 0.981 / 0.661 / 0.790 | 0.121 / 0.094 / 0.106 (lỗi, xem dưới) |
| GCN (gộp tx + addr) | 0.664 / 0.715 / 0.688 | 0.512 / 0.544 / 0.527 |
| GNN dị thể (early stopping) | 0.403 / 0.748 / 0.524 | 0.248 / 0.716 / 0.368 |
| Temporal GNN (GRU) | 0.217 / 0.801 / 0.342 | 0.217 / 0.811 / 0.342 |
| RF + đặc trưng Active Diffusion | 0.989 / 0.650 / 0.784 | 0.116 / 0.083 / 0.097 (lỗi, xem dưới) |

Tham khảo bài báo gốc (RF): giao dịch P 0.975 / R 0.719; ví P 0.911 / R 0.789.

## Nhận xét

- Trên giao dịch, Random Forest mạnh nhất và gần khớp bài báo; các GNN đều kém hơn. Đặc trưng Active Diffusion chưa cải thiện rõ rệt.
- Hiệu năng giảm mạnh từ val sang test do concept drift: đặc trưng trôi liên tục theo thời gian, không có ranh giới chia "sạch".

## Vấn đề đã biết

- **Random Forest trên ví:** kết quả thấp bất thường so với bài báo. Nghi do cấu hình `class_weight`; đã có hướng sửa (`balanced_subsample`, tăng số cây) nhưng **chưa chạy lại để xác nhận**.
- **Temporal GNN:** Precision và F1 của giao dịch và ví trùng khớp bất thường, **chưa xác minh** có lỗi kỹ thuật hay không.
- Đặc trưng Active Diffusion là bản lấy cảm hứng, không phải mô hình ADGNN đầy đủ (không có phần ego embedding học được).

## Cách chạy

```bash
pip install -r requirements.txt
```

Chạy các notebook theo thứ tự: tiền xử lý → dựng đồ thị → baseline → GNN → phân tích drift → đặc trưng Active Diffusion + RF.

## Tài liệu tham khảo

- Elmougy, Y. & Liu, L. (2023). *Demystifying Fraudulent Transactions and Illicit Nodes in the Bitcoin Network for Financial Forensics.* KDD '23. arXiv:2306.06108
- Jiang, M. (2025). *An Active Diffusion Neural Network for Graphs.* arXiv:2510.19202
