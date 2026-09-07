# Ghi chép công khai từ dự án Homport

Repo này chỉ chứa phần kiến thức tách rời được, viết để dùng lại ở dự án khác. Mã nguồn sản phẩm nằm ở repo riêng.

## Nội dung

- [`docs/oauth-cloudflare-desktop.md`](docs/oauth-cloudflare-desktop.md) — luồng OAuth của Cloudflare cho ứng dụng desktop: public client, PKCE, không có refresh token, cách bắt callback trên loopback, sáu quyền cần xin và ý nghĩa thật của từng quyền, những chỗ dễ sai và cách phát hiện. Mọi điều trong tài liệu đã chạy thật trên tài khoản Cloudflare thật; chỗ nào chưa kiểm chứng đều được đánh dấu rõ.

## Về Homport

Homport là ứng dụng Windows biến một web app đang chạy trên máy bạn thành một địa chỉ web riêng tư, dùng Cloudflare Tunnel và Cloudflare Access. Ứng dụng chạy trên tài khoản Cloudflare của chính người dùng, không có máy chủ của nhà phát triển, không thu thập dữ liệu.

Đây là ứng dụng độc lập, không liên kết với Cloudflare, Inc.

## Giấy phép

Xem [LICENSE](LICENSE).
