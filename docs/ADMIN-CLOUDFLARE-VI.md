# MEC Marine — Hướng dẫn quản trị và triển khai Cloudflare

Bản tiếng Việt của `ADMIN-CLOUDFLARE.md`, dành cho người triển khai hệ thống và nhân viên quản trị nội dung.

**Trạng thái bàn giao:** mã nguồn đã được triển khai và kiểm thử trên môi trường cục bộ. Hệ thống quản trị và gửi email chưa được kích hoạt trên tài khoản Cloudflare của bạn; việc đồng bộ đầy đủ lên GitHub cũng chưa hoàn tất. Các bước dưới đây là hướng dẫn cấu hình, không phải xác nhận hệ thống đã chạy trực tuyến.

## 1. Chức năng hiện có

| Nhóm | Chức năng |
|---|---|
| Sản phẩm, dịch vụ | Thêm, sửa, tải ảnh/PDF; tick để đăng, bỏ tick để ẩn |
| Thông báo, tuyển dụng | Soạn nội dung, lưu nháp, đăng và đặt ngày tự ẩn |
| Danh mục 3D | Gắn sản phẩm vào nhóm thiết bị cabin/boong có sẵn |
| Yêu cầu khách hàng | Lưu thông tin liên hệ, nội dung, nguồn trang và thời điểm gửi |
| Theo dõi bán hàng | Tiếp nhận, phân công, ghi chú và cập nhật trạng thái |
| Thông báo email | Gửi đến cả `contact@vnmec.com.vn` và `sales@vnmec.com.vn` sau khi cấu hình dịch vụ gửi |

Dữ liệu khởi tạo gồm **113 sản phẩm và 8 dịch vụ**, được nhập một lần vào D1. Mô tả, ảnh sản phẩm, liên kết PDF và nhóm thiết bị 3D được giữ lại. Nội dung tạo mới mặc định là bản nháp. Việc đăng/ẩn có hiệu lực trên danh sách, trang chi tiết, sitemap và dữ liệu 3D mà không cần triển khai lại mã nguồn.

Biểu mẫu lưu yêu cầu vào cơ sở dữ liệu trước khi thử gửi email. Hệ thống chống lưu trùng khi gửi lại cùng một yêu cầu. Nếu gửi email thất bại, tác vụ chạy mỗi 5 phút sẽ kiểm tra các thông báo đến hạn để thử lại; thời gian chờ tăng dần. Danh sách quản trị hiển thị trạng thái gửi và có thao tác thử lại khi thông báo đã thất bại. Nhà cung cấp nhận email để gửi không đồng nghĩa email chắc chắn đã vào hộp thư người nhận.

Danh sách yêu cầu tự làm mới mỗi 15 giây khi đang hiển thị và không mở hộp chỉnh sửa. Đăng nhập quản trị dùng Cloudflare Access, kiểm tra chữ ký JWT và danh sách email được cấp quyền; không có mật khẩu mặc định cho môi trường thật.

## 2. Chạy thử trên Windows

1. Cài **Node.js bản LTS**.
2. Giải nén gói dự án hoặc tải dự án bằng GitHub Desktop khi mã nguồn đã được đồng bộ đầy đủ.
3. Nhấp đúp tệp **`preview.cmd`** trong thư mục dự án.
4. Chờ cài thư viện và khởi tạo cơ sở dữ liệu cục bộ.
5. Mở `http://localhost:8787/admin` để dùng trang quản trị; mở `http://localhost:8787/` để xem website.
6. Nhấn **Ctrl+C** trong cửa sổ lệnh để dừng.

Bản chạy thử chỉ bỏ qua đăng nhập khi truy cập trên máy cục bộ. D1 và R2 được mô phỏng trên máy, chưa có thông tin xác thực gửi email. **Nội dung sửa trên máy không tự đồng bộ lên website thật.**

Có hai luồng làm việc riêng:

- **Người phát triển:** sửa mã nguồn trên Windows → Commit và Push bằng GitHub Desktop → Cloudflare triển khai.
- **Nhân viên nội dung/bán hàng:** đăng nhập `/admin` trên website thật → sửa nội dung hoặc xử lý yêu cầu → lưu trực tiếp vào dữ liệu của website.

## 3. Dữ liệu nằm ở đâu?

| Nơi lưu | Nội dung |
|---|---|
| GitHub | Mã nguồn, giao diện, ảnh/tài liệu có sẵn trong dự án và dữ liệu khởi tạo |
| Cloudflare Workers | Chạy website, trang quản trị và API |
| Cloudflare D1 | Nội dung được chỉnh sửa, yêu cầu khách hàng, phân công, ghi chú và hàng đợi thông báo |
| Cloudflare R2 | Ảnh/PDF tải lên từ trang quản trị |

Dữ liệu D1 và tệp tải lên R2 **không tự được sao chép về GitHub**. Cần sao lưu riêng.

Bản này dùng **Cloudflare Workers kèm tài nguyên tĩnh**. Không sử dụng cấu hình Pages chỉ phục vụ tệp tĩnh của bản trước. Giao diện website và trình tạo nội dung hiện tại được giữ lại; **chưa chuyển sang Astro**.

## 4. Thiết lập lần đầu trên Cloudflare

Thực hiện các lệnh trong cửa sổ Terminal tại thư mục gốc của dự án, nơi có `package.json` và `wrangler.jsonc`.

### Bước 1 — Cài thư viện và đăng nhập

Nếu chưa chạy `preview.cmd`, cài thư viện trước:

```sh
npm ci
```

Đăng nhập Cloudflare bằng trình duyệt:

```sh
npx wrangler login
```

### Bước 2 — Tạo cơ sở dữ liệu D1

```sh
npx wrangler d1 create mec-marine
```

Sao chép **database ID** được trả về vào trường `database_id` của mục `d1_databases` ở cấp ngoài cùng trong `wrangler.jsonc`, thay cho `REPLACE_WITH_D1_ID`.

Không sửa nhầm cấu hình nằm trong `env.local`; đó là môi trường chạy thử.

### Bước 3 — Tạo kho ảnh và PDF

```sh
npx wrangler r2 bucket create mec-marine-media
```

Kích hoạt R2 trong tài khoản nếu Cloudflare yêu cầu. Tệp đã có trong dự án vẫn nằm trong Git; tệp nhân viên tải lên qua trình duyệt sẽ nằm trong bucket này.

### Bước 4 — Triển khai lần đầu

```sh
npm run deploy
```

Lệnh này áp dụng các thay đổi cơ sở dữ liệu rồi triển khai Worker. Dùng địa chỉ thực tế do Cloudflare trả về để kiểm tra website.

Lúc này danh mục công khai có thể xem được, nhưng trang quản trị và biểu mẫu gửi yêu cầu sẽ từ chối hoạt động cho đến khi cấu hình đủ các bước bảo vệ bên dưới.

### Bước 5 — Gắn tên miền

Thêm tên miền hoặc tên miền phụ bạn chọn vào Worker trong Cloudflare.

Chỉ thay DNS của `vnmec.com.vn` khi bạn chủ động quyết định thay thế website đang hoạt động. Tài liệu này không thực hiện thay đổi DNS.

### Bước 6 — Cấu hình đăng nhập cho nhân viên

Trong **Cloudflare Zero Trust → Access → Applications**, tạo ứng dụng **Self-hosted** bảo vệ cả hai đường dẫn:

```text
<domain>/admin*
<domain>/api/admin/*
```

Thay `<domain>` bằng tên miền thật. Cho phép các email nhân viên được chọn đăng nhập bằng mã PIN qua email hoặc nhà cung cấp danh tính của bạn. Cả hai đường dẫn phải dùng cùng ứng dụng Access và cùng audience.

Không đặt biểu mẫu công khai `/api/leads` sau lớp đăng nhập Access, vì khách truy cập cần gửi yêu cầu mà không đăng nhập.

Điền các biến trong `wrangler.jsonc`:

| Biến | Giá trị cần điền |
|---|---|
| `ACCESS_TEAM_DOMAIN` | Tên miền Access của nhóm, ví dụ `ten-nhom.cloudflareaccess.com` |
| `ACCESS_AUD` | Audience tag của ứng dụng Access vừa tạo |
| `ADMIN_EMAILS` | Các email nhân viên được quyền quản trị, phân cách bằng dấu phẩy |

**Email nhận thông báo không tự động có quyền quản trị.** Bạn phải chủ động đưa email đăng nhập của nhân viên vào danh sách cho phép. Kiểm tra JWT tại Worker cũng từ chối truy cập quản trị chưa xác thực qua địa chỉ `workers.dev`.

### Bước 7 — Chống thư rác bằng Turnstile

Tạo widget Cloudflare Turnstile cho hostname của website thật. Điền:

| Biến | Nội dung |
|---|---|
| `TURNSTILE_SITE_KEY` | Site key của widget |
| `TURNSTILE_HOSTNAME` | Hostname chính xác, không có `https://` hoặc đường dẫn |

Lưu secret bằng lệnh:

```sh
npx wrangler secret put TURNSTILE_SECRET
```

Dán giá trị secret trực tiếp vào cửa sổ lệnh khi được yêu cầu.

### Bước 8 — Kích hoạt gửi email

1. Tạo tài khoản Resend.
2. Xác minh tên miền gửi theo hướng dẫn DNS của Resend.
3. Tạo API key.
4. Điền `EMAIL_FROM` bằng địa chỉ gửi thuộc tên miền đã xác minh. Ví dụ dưới đây chỉ dùng sau khi tên miền tương ứng đã được xác minh:

```text
MEC Marine <notifications@vnmec.com.vn>
```

5. Lưu API key:

```sh
npx wrangler secret put RESEND_API_KEY
```

Biến `NOTIFY_TO` đã đặt sẵn cả hai địa chỉ:

```text
contact@vnmec.com.vn,sales@vnmec.com.vn
```

Không ghi API key vào GitHub hoặc gửi trong hội thoại.

### Bước 9 — Triển khai lại và kiểm tra thực tế

```sh
npm run deploy
```

Sau đó:

1. Đăng nhập `<domain>/admin` bằng tài khoản được cấp quyền.
2. Gửi một yêu cầu thử qua website.
3. Xác nhận yêu cầu xuất hiện một lần trong danh sách quản trị.
4. Kiểm tra cả hộp thư Contact và Sales, kể cả thư mục spam.
5. Thử tiếp nhận yêu cầu, thêm ghi chú và đổi trạng thái.

Các kiểm thử khi bàn giao chỉ dùng môi trường cục bộ; chưa gửi email thật.

## 5. Tự triển khai từ GitHub sang Cloudflare

Sau khi toàn bộ dự án đã có trong `ngtrthanh/mec-marine`, vào **Cloudflare Workers & Pages → Create application → Import repository** và chọn repository đó.

| Thiết lập | Giá trị |
|---|---|
| Loại ứng dụng | **Workers** |
| Repository | `ngtrthanh/mec-marine` |
| Nhánh chính | `main` |
| Lệnh triển khai | `npm run deploy` |
| Build nội dung | Không cần chạy lại; dự án đã kèm thư mục `dist` |

Cấu hình cùng các liên kết tài nguyên (bindings), biến và secret đã nêu ở mục 4. Giữ nguyên database ID và tên bucket để không tách dữ liệu sang tài nguyên khác.

Mỗi lần cập nhật mã nguồn: **Commit → Push → kiểm tra kết quả triển khai trên Cloudflare**. Không dùng `--env local` khi triển khai môi trường thật.

## 6. Hướng dẫn nhanh cho nhân viên

### Thêm sản phẩm, dịch vụ, thông báo hoặc tuyển dụng

1. Chọn đúng mục ở thanh bên.
2. Bấm **Thêm mới**.
3. Nhập tên, danh mục, mô tả ngắn và nội dung chi tiết.
4. Chọn ảnh/PDF nếu cần.
5. Bấm **Lưu thay đổi**. Nếu chưa tick đăng, nội dung được lưu dưới dạng nháp.

### Đăng hoặc tạm ẩn

- Tick **Đăng trên website** rồi lưu trong cửa sổ chỉnh sửa để công khai nội dung.
- Bỏ tick để ẩn. Dữ liệu và lịch sử yêu cầu trước đó vẫn được giữ.
- Checkbox ở danh sách thực hiện thay đổi ngay khi tick/bỏ tick.
- Ngày tự ẩn có hiệu lực vào cuối ngày theo giờ Việt Nam. Nội dung đã hết hạn vẫn bị ẩn dù checkbox đang được tick.

### Thay ảnh hoặc catalogue

Mở nội dung → **Chọn ảnh / Chọn PDF** → chọn tệp → **Lưu thay đổi**.

Hỗ trợ JPG, PNG, WebP và PDF, tối đa **10 MB mỗi tệp**. Không nhận SVG hoặc HTML. Đường dẫn tệp tải lên có thể truy cập công khai; không tải tài liệu nội bộ hoặc thông tin khách hàng vào đây.

Trong mục **Ảnh & tài liệu**, có thể sao chép đường dẫn để dùng lại cho nội dung khác.

### Gắn sản phẩm vào cabin hoặc boong 3D

Trong phần chỉnh sửa sản phẩm, chọn nhóm tại **Vị trí trong cabin / boong 3D**. Các vị trí tương ứng của cùng hệ thống được cập nhật tự động.

Chọn **Không hiển thị trong 3D** nếu chỉ muốn đăng sản phẩm trong danh mục. Việc tạo thêm vị trí vật lý hoặc hình dáng thiết bị mới trong mô hình cần sửa mã nguồn.

### Xử lý yêu cầu khách hàng

1. Mở **Yêu cầu khách hàng**.
2. Bấm **Mở yêu cầu**.
3. Chọn **Tôi tiếp nhận yêu cầu này** hoặc nhập người phụ trách.
4. Thêm ghi chú nội bộ và cập nhật trạng thái.
5. Bấm **Lưu thay đổi**.

Các trạng thái: **Mới → Đã liên hệ → Đã báo giá → Thành công / Không thành công**.

Nếu đồng nghiệp vừa sửa cùng yêu cầu, hệ thống báo xung đột. Tải lại thông tin trước khi sửa tiếp để tránh ghi đè thay đổi của người khác.

## 7. Giới hạn hiện tại và sao lưu

- Một số PDF vẫn trỏ về `vnmec.com.vn`. Muốn lưu độc lập, tải PDF lên qua quản trị và thay đường dẫn trong nội dung.
- Email gửi trực tiếp vào hộp thư Contact/Sales chưa được nhập vào cơ sở dữ liệu. Hệ thống hiện thu nhận yêu cầu qua biểu mẫu website.
- Thao tác rê chuột hoặc bấm xem thiết bị ẩn danh không tự tạo thành một khách hàng tiềm năng.
- Tất cả nhân viên được cấp quyền trong bản đầu có cùng quyền quản trị. Chỉ cấp quyền cho người phù hợp.
- GitHub lưu mã nguồn và dữ liệu khởi tạo, không chứa bản sao tự động của dữ liệu khách hàng hoặc tệp tải lên sau này.

Sao lưu D1 vào tệp riêng, ví dụ:

```sh
npx wrangler d1 export mec-marine --remote --output=mec-backup-private.sql
```

Chạy lệnh từ thư mục sao lưu riêng hoặc chuyển tệp ra khỏi repository ngay sau khi xuất. Không Commit tệp sao lưu chứa thông tin khách hàng. Với R2, sử dụng công cụ sao lưu/đồng bộ tệp phù hợp.

Tại thời điểm bàn giao, chưa cấu hình tài nguyên Cloudflare, DNS, thông tin xác thực email hoặc tài khoản quản trị thật. Cần hoàn thành mục 4 trên tài khoản của bạn để kích hoạt.

## 8. Kiểm thử dành cho người triển khai

Kiểm tra cơ chế xác thực:

```sh
npm run test:admin
```

Khởi động bản chạy thử, sau đó mở cửa sổ Terminal thứ hai và chạy:

```sh
npm run test:integration
```

Dùng cơ sở dữ liệu cục bộ mới để kiểm tra đăng/ẩn, lọc danh mục 3D, hiển thị nội dung an toàn, chống ghi đè, chống yêu cầu giả mạo, chống lưu trùng, phân công và tải tệp.

Các bài kiểm thử tạo bản ghi thử trên máy. Không đổi đích kiểm thử sang website thật.
