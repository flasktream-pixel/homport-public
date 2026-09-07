# OAuth Cloudflare cho app desktop: kinh nghiệm thực chiến và cách làm

> Tài liệu này **tự đứng một mình** để mang sang dự án khác. Mọi sự thật trong đây đều đã chạy thật trên tài khoản Cloudflare thật (03–04/09/2026) với một app WPF trên Windows, không phải suy đoán từ tài liệu Cloudflare. Chỗ nào chưa chạy thật được đánh dấu **[CHƯA KIỂM]**. Không có token, không có secret, không có id đầy đủ trong file này.
>
> Điểm khác với luồng OAuth Cloudflare cho **web app chạy trên máy chủ**, vốn là thứ hầu hết tài liệu đang nói tới: ở đó client giữ được `client_secret`. Còn ở đây client là **public** chạy trên máy người dùng, nên **không có secret**, redirect về **loopback**, và **không có refresh token**. Ba khác biệt đó kéo theo gần như mọi điều còn lại trong tài liệu này.

---

## 0. Tóm tắt một phút

| Câu hỏi | Trả lời đã kiểm |
|---|---|
| Cloudflare có cho app desktop đăng nhập OAuth không? | **Có.** Self-managed OAuth client, public client, PKCE S256, không secret. |
| Redirect về đâu? | `http://localhost:<CỔNG CỐ ĐỊNH>/callback`. Cổng **không thể ngẫu nhiên** vì Cloudflare khớp nguyên văn chuỗi đã đăng ký. |
| Có refresh token không? | **Không.** Access token sống đúng **1 giờ** (`expires_in` 3599–3600). Client không được xin `offline_access`. |
| Hết giờ thì sao? | Người dùng phải đăng nhập lại qua trình duyệt, **và phải bấm Authorize lại** (Cloudflare không nhớ consent lần trước với client Private). |
| Phải gửi `scope` không? | **Bắt buộc.** Bỏ trống thì màn consent hiện "0 total permissions" và không cho Authorize. |
| Định danh scope lấy ở đâu? | Từ popup "scopes" của client trên dashboard, hoặc `GET /client/v4/oauth/scopes`. **Không suy ra được từ nhãn hiển thị.** |
| Token dùng được với API thật không? | **Có**, Bearer trên `api.cloudflare.com`, đúng scope đã cấp. |
| Bẫy lớn nhất? | (1) scope quyền Access có hai định danh gần giống mà ngược vai; (2) `Private` client chỉ cho thành viên account sở hữu bấm đồng ý, sai tài khoản trình duyệt thì báo "private application"; (3) lỗi chỉ lộ khi chạy thật, unit test không thấy. |

---

## 1. Đăng ký client trên dashboard

Đường đi: `dash.cloudflare.com` → chọn account → **Manage Account** → **OAuth clients** → **Create**.

| Trường | Điền gì | Vì sao |
|---|---|---|
| Name | Tên app | Hiện trên màn consent "X wants to access your account" |
| Client URL | Domain thật của sản phẩm (ví dụ `https://tenapp.dev`) | Điền `http://localhost:...` thì client vẫn dùng được ở chế độ **Private**, nhưng dashboard báo `Client URL required` và **không chuyển sang Public được** |
| Redirect URI | `http://localhost:47823/callback` (cổng tuỳ chọn, nhưng **cố định**) | Khớp **nguyên văn** lúc authorize và lúc đổi token. Xem mục 4 |
| Response type / Grant | Code / Authorization Code | |
| Token Authentication Method | Dashboard hiển thị `Client Secret POST`, ô secret bị mờ | **Bỏ qua.** Đổi token bằng PKCE **không cần secret** vẫn trả 200. Đừng nhúng secret vào app desktop |
| Scopes | Tick đúng những quyền app cần, tất cả **Required** | Client **chỉ được xin đúng scope đã đăng ký**; xin khác là `invalid_scope` ngay ở authorize |
| Visibility | Private lúc phát triển | Private = **chỉ thành viên của account sở hữu client** mới bấm Authorize được. Muốn người lạ dùng: Public + Verified, xem mục 9 |

Sau khi tạo, copy **client_id** (32 hex). Nó **không phải bí mật**: nó nằm nguyên trong URL authorize mà người dùng nhìn thấy trên thanh địa chỉ. Nhúng thẳng vào mã nguồn app là đúng cách; đừng đọc từ file cấu hình người dùng sửa được.

**Lấy định danh scope**: trên bảng OAuth clients, bấm vào số scope của client để mở popup, định danh nằm cạnh nhãn. Hoặc gọi `GET https://api.cloudflare.com/client/v4/oauth/scopes` bằng một API token bất kỳ (trả ~383 scope). Ví dụ đã dùng:

| Nhãn trên dashboard | Định danh gửi trong `scope` |
|---|---|
| DNS Write | `dns.write` |
| Zone Read | `zone.read` |
| Access: Apps and Policies Write | `zone-access.write` |
| Access: Organizations, Identity Providers, and Groups Write | `access-acct.write` |
| Cloudflare Tunnel Write | `argotunnel.write` |
| Account Settings Read | `account-settings.read` |

Ba cái không đoán được từ nhãn: Tunnel mang tên cũ `argotunnel`; hai scope Access tên có `zone` lại làm việc với app/policy, tên có `acct` lại làm việc với tổ chức/identity provider. Đã có một lần ánh xạ nhầm và mất nửa ngày (mục 10).

---

## 2. Endpoint (xác nhận qua OIDC discovery `https://dash.cloudflare.com/.well-known/openid-configuration`)

| Việc | URL |
|---|---|
| Authorize | `https://dash.cloudflare.com/oauth2/auth` |
| Token | `https://dash.cloudflare.com/oauth2/token` |
| Revoke | `https://dash.cloudflare.com/oauth2/revoke` **[CHƯA KIỂM]** |
| Device authorization | `https://dash.cloudflare.com/oauth2/device/auth` (có, chưa dùng) |
| Userinfo / JWKS | `/oauth2/userinfo`, `/.well-known/jwks.json` (có, chưa dùng) |
| API | `https://api.cloudflare.com/client/v4/...` với header `Authorization: Bearer <access_token>` |

Discovery khai `token_endpoint_auth_methods_supported` gồm `none` (public client hợp lệ), `code_challenge_methods_supported` gồm `S256`, và có `refresh_token` trong `grant_types_supported` — nhưng xem mục 5: client không xin được `offline_access` nên refresh không bao giờ tới tay.

---

## 3. Trình tự HTTP đầy đủ (không phụ thuộc ngôn ngữ)

```
[1] Sinh ngẫu nhiên:
    state          = base64url(32 byte)
    code_verifier  = base64url(64 byte)
    code_challenge = base64url(SHA256(code_verifier))

[2] Mở listener loopback TRƯỚC khi mở trình duyệt (mục 4), rồi mở trình duyệt hệ thống tới:
    https://dash.cloudflare.com/oauth2/auth
      ?client_id=<client_id>
      &redirect_uri=http://localhost:47823/callback      (nguyên văn như đã đăng ký)
      &response_type=code
      &state=<state>
      &code_challenge=<code_challenge>
      &code_challenge_method=S256
      &scope=dns.write zone.read ...                        (BẮT BUỘC, ngăn cách bằng dấu cách)

[3] Người dùng bấm Authorize. Trình duyệt về:
    GET http://localhost:47823/callback?code=cfoac_...&state=<state>
    Lỗi thì:  ?error=access_denied            (người dùng bấm Cancel)
              ?error=invalid_scope&error_description=...   (xin scope chưa đăng ký)
    Listener: so state, chỉ nhận đúng MỘT request, trả trang "You can close this tab", đóng listener.

[4] Đổi code lấy token (POST form-urlencoded, KHÔNG client_secret):
    POST https://dash.cloudflare.com/oauth2/token
      grant_type=authorization_code
      code=<code>
      client_id=<client_id>
      redirect_uri=http://localhost:47823/callback
      code_verifier=<code_verifier>
    Không được retry request này: code dùng một lần.

[5] Response 200:
    { "access_token": "cfoa...", "token_type": "bearer", "expires_in": 3599, "scope": "dns.write zone.read ..." }
    KHÔNG có refresh_token. Trường scope là danh sách THẬT SỰ được cấp, dùng để phát hiện thiếu quyền.

[6] Dùng: Authorization: Bearer <access_token> trên api.cloudflare.com. Hết hạn sau 1 giờ.
```

Ghi chú thực nghiệm: mã `code` có tiền tố `cfoac_`, access token có tiền tố `cfoa`, dài ~93 ký tự. Đừng dựa vào tiền tố để xử lý logic, chỉ dùng để nhận diện khi cần redact log.

---

## 4. Listener loopback: những điều bắt buộc

- **Cổng cố định**, đúng cổng trong redirect URI đã đăng ký. Khuyến nghị "cổng bất kỳ" của RFC 8252 **không áp dụng** với Cloudflare vì so khớp nguyên văn. Hệ quả: cổng có thể bị chương trình khác chiếm → cần thông điệp riêng ("app khác đang dùng cổng X"), không phải lỗi chung chung.
- **Bind hai listener riêng**: `127.0.0.1:<port>` và `[::1]:<port>`. Trình duyệt có thể resolve `localhost` ra IPv6 trước. **Không** bind `IPv6Any` với `IPv6Only=false`: cách đó nhận đủ hai họ nhưng mở cổng ra mọi card mạng — lỗi bảo mật thật đã mắc ở prototype.
- Chỉ nhận **một** request có `state` đúng, rồi đóng ngay. Request khác `state` → trang "Sign-in could not be verified", không dùng code.
- Thời gian chờ callback: **5 phút**, hết giờ đóng listener và báo timeout (khác với người dùng bấm Cancel).
- Trang callback nên tự đủ, không tải tài nguyên ngoài: `<h2>You can close this tab</h2><p>Go back to the app to continue.</p>`.
- Mở URL bằng **trình duyệt hệ thống** (`ShellExecute`), **không** nhúng webview. Người dùng cần thấy thanh địa chỉ `dash.cloudflare.com` thật để phân biệt phishing. Chỉ cho phép scheme `http`/`https` khi mở.

---

## 5. Không có refresh token — và cách sống chung

Sự thật đo được (hai lần, giống hệt nhau):

- `offline_access` **không nằm trong danh mục 383 scope** mà dashboard cho phép gán cho client. Xin thêm vào tham số `scope` → `error=invalid_scope: The OAuth 2.0 Client is not allowed to request scope 'offline_access'`.
- Token response **không có `refresh_token`**. `expires_in` = 3599 hoặc 3600.
- Đăng nhập lần hai, cùng trình duyệt, cùng phiên Cloudflare: **vẫn hiện màn consent, vẫn phải bấm Authorize**. Không có chuyển hướng im lặng.

Thiết kế đã chốt và đã chạy thật:

1. **Tách quyền chạy khỏi quyền quản lý.** Thứ cần chạy liên tục (ở đây là connector cloudflared) dùng credential riêng dài hạn (tunnel token) lưu DPAPI, **không cần OAuth**. Access token OAuth chỉ cho thao tác quản lý. Hết giờ thì mọi thứ đang chạy vẫn chạy.
2. **Không làm mới ngầm, không có timer.** Lưu access token cùng `expires_at`; lúc mở app đọc lại; còn hạn thì `SignedIn`, hết hạn thì `SignInRequired`. **Tuyệt đối không tự bật trình duyệt lúc khởi động.**
3. **Đăng nhập theo nhu cầu, nối liền thao tác.** Nút ghi vẫn bấm được ở trạng thái `SignInRequired`; bấm vào thì hiện một dialog báo trước một câu ("Cloudflare will ask you to allow X again. Click Allow, then come back here."), chạy đăng nhập, rồi **tự tiếp tục đúng thao tác đang dở**. Đóng gói bằng một hàm dạng `RunWriteOperation(closure)`: closure giữ dữ liệu người dùng đã nhập, service tự lo hỏi → đăng nhập → kiểm scope → kiểm đúng account → chạy tiếp. Giao diện không phải nhớ gì.
4. **Xem thì không cần token.** Dashboard, trạng thái, chẩn đoán đọc từ dữ liệu cục bộ. Người dùng mở app xem tình hình không bao giờ bị hỏi đăng nhập.

Chi phí thật: hai cú bấm mỗi lần token hết hạn mà người dùng muốn *thay đổi* gì đó. Với app "cài một lần rồi để đó" là chấp nhận được.

**[CHƯA KIỂM]**: client **Public + Verified** có được bỏ qua màn consent không (nhiều nhà cung cấp làm vậy). Tạo client qua API `POST /oauth/clients` có nhận `offline_access` không.

---

## 6. Lưu token và khôi phục phiên

- Lưu bằng **DPAPI `CurrentUser`** kèm entropy riêng của app, mỗi bí mật một file (`secrets/oauth.access.dpapi`, `secrets/tunnel.token.dpapi`). Không Registry, không file cấu hình chung.
- Nội dung blob gồm: token, `expires_at` (UTC), danh sách scope đã cấp. **Phải lưu cả scope**: sau khi khôi phục, preflight cần biết đã có quyền gì; quên chỗ này thì mở lại app sẽ báo "thiếu quyền" dù token tốt (lỗi thật đã mắc).
- Lúc mở app: đọc blob → còn hạn → trạng thái `SignedIn` và **đi thẳng qua màn đăng nhập** (lỗi thật đã mắc: Core khôi phục đúng nhưng giao diện vẫn mở màn Sign in).
- Blob hết hạn: giữ trên đĩa (vô hại) nhưng không giữ chuỗi token trong bộ nhớ.
- Đăng xuất: gọi revoke (**[CHƯA KIỂM]** phía Cloudflare) rồi xoá blob. Đăng xuất **không** được xoá credential dài hạn của thứ đang chạy nếu người dùng chỉ muốn thoát phiên quản lý.

---

## 7. Sau khi có token: preflight và những lỗi API dễ hiểu nhầm

Ngay sau khi đổi token, chạy một preflight rẻ tiền để biết token thuộc account nào và có đủ quyền không:

- Dùng `GET /accounts` (200 với scope `account-settings.read`). **Đừng dùng `/memberships`**: nó đòi thêm scope `memberships.read`, trả 403 `code 10000` nếu thiếu.
- `GET /zones?account.id=<id>` để liệt kê domain; `status` = `active` mới dùng được.
- So account của token với account đã lưu từ lần trước; lệch thì báo "You signed in to a different Cloudflare account" thay vì ghi lung tung.

Bảng lỗi đã gặp thật và ý nghĩa đúng của chúng:

| Response | Nghĩa thật | Hay bị hiểu nhầm thành |
|---|---|---|
| 403 `code 9999`, message chứa `access.api.error.not_enabled` | Account **chưa bật Zero Trust**; mọi `/access/*` trả vậy | Thiếu scope |
| 403 `code 1010`, `error: auth.forbidden` (chú ý: nằm ở khoá `error`, không phải `message`) | Gọi endpoint **mức account** bằng quyền **mức zone** (ví dụ `POST /accounts/{id}/access/apps` với `zone-access.write`) | Token hỏng |
| 403 `code 10000 Authentication error` | Thiếu hẳn quyền cho endpoint đó (ví dụ `/memberships`, `/access/tags`) | Zero Trust chưa bật |
| 401 | Token hết hạn hoặc đã revoke | |
| `error=invalid_scope` ở authorize | Xin scope client chưa đăng ký, hoặc `offline_access` | Lỗi mạng |

Bài học về **mức phạm vi**: scope tên `zone-access.write` chỉ mở được `/zones/{zone_id}/access/apps...`; muốn gọi `/accounts/{id}/access/apps` cần quyền mức account mà scope này không có. Endpoint mức zone chạy được với **cả hai** loại quyền, nên chọn zone-level là an toàn. Hệ quả kéo theo: `tags` của Access app phải tạo trước qua `POST /accounts/{id}/access/tags` — endpoint không có bản zone-level — nên với scope zone, **không dùng tag được**, marker sở hữu phải nằm trong `name`.

---

## 8. Ánh xạ lỗi cho giao diện

| Tình huống | Nhận diện | Thông điệp nên có |
|---|---|---|
| Người dùng bấm Cancel / đóng tab | callback `error=access_denied`, hoặc huỷ trong app | "Sign-in was cancelled." — cho bấm lại |
| Hết 5 phút không callback | timeout listener | "We didn't hear back from your browser." — **khác** với Cancel |
| `state` sai / callback thiếu code | so khớp | "Sign-in could not be verified. Try again." |
| Thiếu scope sau đăng nhập | so `scope` trong token response với danh sách bắt buộc | "X didn't get the permissions it needs. Sign in again and allow access to your <nhóm thân thiện>." — tên quyền thật chỉ để trong Details |
| Cổng loopback bị chiếm | bind thất bại | "Another app is using the port X needs to finish signing in. Close that app, then try again." |
| Không mở được trình duyệt | ShellExecute ném | "X couldn't open your browser. Set a default browser in Windows Settings." |
| Sai tài khoản trình duyệt với client Private | Cloudflare hiện "This is a private application…" **trong trình duyệt**, app chỉ thấy timeout/cancel | Ghi trong hướng dẫn: đăng nhập dash.cloudflare.com đúng tài khoản trước |
| Token hết hạn giữa chừng | 401 khi gọi API | Chuyển `SignInRequired`, không mất việc đang làm |
| Token của account khác | preflight so account | "You signed in to a different Cloudflare account." |

---

## 9. Đường lên Public (để người lạ dùng được)

Từ trạng thái client hiện tại và một lần đã làm trọn quy trình này cho một dự án web trước đó:

1. Client URL phải là domain thật (`http://localhost` bị `Client URL required`).
2. `logo_uri` không rỗng (thiếu thì chặn với lỗi `non-empty logo_uri`).
3. Xác minh publisher domain bằng bản ghi DNS TXT `cloudflare_oauth_client_publisher=<token>`.
4. Visibility Private → Public. Kết quả mong đợi: Visibility Public, Verification Verified.

Trước khi Public, mọi người thử app đều phải là thành viên account sở hữu client. Đây là lý do dev nên tự tạo client trên chính account của mình ngay từ đầu.

---

## 10. Những lỗi chỉ lộ khi chạy thật (để đừng lặp lại)

Mười phút chạy thật đầu tiên làm app sập ba lần dù **1376 unit test xanh**. Ghi lại để lần sau chạy thật **ngay khi nối dây xong**, không đợi cuối:

| Lỗi | Vì sao test không thấy | Cách chữa |
|---|---|---|
| Ánh xạ nhầm hai scope Access → API token gate thiếu quyền tổ chức | Tên định danh không suy ra được từ nhãn | Bảng đối chiếu ở một chỗ; test khoá **định danh**, không khoá nhãn |
| Trình duyệt mặc định đăng nhập account khác → "private application" ×3 | Không phải lỗi code | Kiểm ngữ cảnh (account nào đang đăng nhập) trước khi đổ lỗi cấu hình |
| Bỏ `scope` khỏi URL vì tưởng Cloudflare dùng scope đã cấu hình | Không có test với Cloudflare thật | `scope` bắt buộc, luôn gửi |
| Spinner trong template ném "name cannot be found in the name scope" ngay sau khi có token → app chết | Không test nào mở cửa sổ thật để `Loaded` bắn | Animation đặt trên chính phần tử, không `TargetName`; test mở cửa sổ thật |
| Không có hàng rào lỗi toàn cục → một lỗi vẽ giết cả tiến trình | Thiếu hẳn | `DispatcherUnhandledException` + `UnobservedTaskException`: ghi báo cáo đã redact, hỏi Continue/Quit |
| Token còn hạn mà mở lại vẫn đòi Sign in | Chưa từng có token thật trên đĩa lúc test | Luồng đầu đọc trạng thái phiên khôi phục; nạp lại scope |
| Callback từ thread nền sửa `ObservableCollection` đang bind → bảng rỗng không báo lỗi | Unit test không có CollectionView | Adapter phía giao diện bắt `SynchronizationContext` lúc dựng và `Post` mọi callback về đó; bỏ `ConfigureAwait(false)` trong code giao diện |

Cách chạy thật khi người chủ bận: điều khiển app bằng UI Automation (`System.Windows.Automation` từ PowerShell: tìm theo Name + ControlType, `SelectionItemPattern`/`InvokePattern`/`ValuePattern`/`TogglePattern`), chụp cửa sổ bằng `Graphics.CopyFromScreen`, đọc Event Log `Application` khi app thoát bất ngờ. Chỉ bước **Authorize trên trình duyệt** là cần người thật.

---

## 11. Checklist bê sang dự án mới

- [ ] Tạo OAuth client trên account của dev; Client URL là domain thật nếu định Public; redirect `http://localhost:<cổng cố định>/callback`; tick scope Required.
- [ ] Lấy **định danh** scope từ popup hoặc `/oauth/scopes`; ghi bảng nhãn ↔ định danh vào một chỗ; test khoá định danh.
- [ ] Prototype 100 dòng chạy trọn vòng trước khi viết code sản phẩm: authorize → callback → đổi token → `GET /accounts`. Ghi lại response thật (đã redact).
- [ ] Public client + PKCE S256 + `state`; **không** secret; `scope` bắt buộc; không xin `offline_access`.
- [ ] Listener: cổng cố định, hai listener loopback riêng, một request, 5 phút, trang callback tự đủ; mã lỗi riêng cho "cổng bị chiếm" và "không mở được trình duyệt".
- [ ] Đổi code: không retry; kiểm `token_type` là `bearer`, có `access_token` và `scope`; thiếu `expires_in` thì mặc định 3600.
- [ ] Lưu DPAPI kèm `expires_at` và scope; khôi phục lúc mở app; **không** tự bật trình duyệt; đăng nhập theo nhu cầu nối liền thao tác.
- [ ] Preflight bằng `GET /accounts`; nhận diện 9999/1010/10000 đúng nghĩa; so account với lần trước.
- [ ] Thứ cần chạy liên tục dùng credential riêng, không phụ thuộc access token.
- [ ] Hàng rào lỗi toàn cục trước lần chạy thật đầu tiên.
- [ ] Chạy thật ngay khi nối dây xong; mỗi lỗi lộ ra viết test hồi quy **đúng tầng** (cửa sổ thật, binary thật), không chỉ unit test.
- [ ] Trước khi Public: Client URL domain thật, logo, TXT xác minh; thử xem consent có còn hiện lại không.

---

## 12. Còn mở

- Revoke (`/oauth2/revoke`) chưa gọi thật lần nào.
- Client Public + Verified có bỏ qua consent hay không.
- `POST /oauth/clients` có cho `offline_access` hay không.
- Hành vi khi người dùng bỏ tick một scope **tuỳ chọn** (client thử nghiệm chỉ có scope Required).
- Device Authorization Grant có sẵn, chưa thử; đáng cân nhắc nếu trình duyệt mặc định hay đăng nhập sai tài khoản.
