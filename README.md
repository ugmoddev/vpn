# PyVPN — VPN tunnel điểm-điểm bằng Python (lấy cảm hứng từ SoftEtherVPN)

Đây là bản triển khai **gốc bằng Python** cho một VPN đơn giản kiểu SSL-VPN:
đóng gói các gói IP qua thiết bị mạng ảo **TUN**, mã hoá và xác thực bằng
**TLS 2 chiều (mutual TLS)**. Đây KHÔNG phải là bản sao/port trực tiếp source
code của SoftEtherVPN — chỉ mô phỏng lại nguyên lý cốt lõi (SSL-VPN qua TUN)
ở quy mô nhỏ, dễ đọc để học tập.

## Kiến trúc

```
[App trên máy A]        [App trên máy B]
      |                        |
   tun0 (10.8.0.2)      tun0 (10.8.0.1)
      |                        |
  vpn_client.py  <== TLS ==>  vpn_server.py
```

- `tun.py` — tạo/điều khiển thiết bị TUN (Linux, cần quyền root)
- `vpn_common.py` — đóng khung gói tin qua TCP + forward 2 chiều
- `vpn_server.py` — lắng nghe, xác thực client bằng chứng chỉ, forward gói tin
- `vpn_client.py` — kết nối tới server, forward gói tin
- `gen_certs.sh` — sinh CA + chứng chỉ server/client tự ký

## Yêu cầu

- Linux (dùng `/dev/net/tun` và lệnh `ip` từ iproute2)
- Python 3.10+ (dùng cú pháp `X | None`)
- `openssl` để sinh chứng chỉ
- Quyền root để tạo interface mạng

## Cài đặt & chạy

### 1. Sinh chứng chỉ

```bash
bash gen_certs.sh
```

Copy `certs/ca.crt`, `certs/client.crt`, `certs/client.key` sang máy client.
Giữ `certs/ca.crt`, `certs/server.crt`, `certs/server.key` trên máy server.

### 2. Chạy server (máy A)

```bash
sudo python3 vpn_server.py
```

### 3. Chạy client (máy B)

```bash
sudo python3 vpn_client.py <IP_cong_khai_cua_server>
```

Sau khi kết nối, hai máy có thể ping nhau qua địa chỉ nội bộ:

```bash
# Từ client
ping 10.8.0.1

# Từ server
ping 10.8.0.2
```

## Giới hạn của bản demo này (so với SoftEtherVPN thật)

| Tính năng | PyVPN (demo) | SoftEtherVPN |
|---|---|---|
| Số client đồng thời | 1 | Nhiều |
| Giao thức | Chỉ 1 kiểu (TLS + TUN) | SSL-VPN, WireGuard, OpenVPN, IPsec, L2TP... |
| NAT traversal, DDNS | Không | Có |
| Web admin console | Không | Có |
| Nén, chống DPI | Không | Có |
| Hiệu năng | Thấp (Python, single-thread I/O) | Cao (C, tối ưu) |

## Mở rộng gợi ý

- Hỗ trợ nhiều client: server giữ dict `{client_ip: socket}`, dùng bảng
  định tuyến nội bộ để forward gói theo IP đích thay vì 1-1.
- Thêm `iptables`/`nftables` NAT để client đi Internet qua server (full-tunnel).
- Thay TCP bằng UDP (giảm hiện tượng "TCP over TCP" làm chậm khi mạng lag).
- Nén dữ liệu trước khi mã hoá (zlib) để giảm băng thông.

## Cảnh báo

Chỉ dùng để kết nối các máy/mạng mà bạn được phép quản trị. Đừng dùng để
truy cập trái phép vào hệ thống của người khác.
