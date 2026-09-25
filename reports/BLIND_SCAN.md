# Quét độc lập trước khi xem pre-label

Frame: ĐIỀN tên một ảnh trong `to_label/round1/images/train/frame_0099`

Số xe nhìn thấy bằng mắt: 24

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe: Vị trí 1 là 2 xe chạy xuôi hướng nhìn phía trái khung hình (chỉ nhìn thấy đèn xe sau, hai xe chồng lấn nhau 80%). Vị trí 2 là xe bị lóa đèn chỉ nhìn thấy đèn xe trước phía trái khung hình

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
