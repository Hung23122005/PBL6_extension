# PhishGuard AI — Chrome / Edge extension

Project Manifest V3 độc lập, React + TypeScript + Vite + Tailwind + Lucide. Không sửa frontend hoặc backend.

## Cài đặt trên Windows PowerShell

Cần Node.js 22.12+ hoặc 24+. Dùng `npm.cmd` nếu PowerShell chặn `npm.ps1`.

```powershell
cd D:\HocKi7\PBL6\web\phishguard_extension
npm.cmd install
npm.cmd run build
```

Mở `chrome://extensions` (Edge: `edge://extensions`), bật **Developer mode**, chọn **Load unpacked** và thư mục `phishguard_extension\dist`. Ghim biểu tượng PhishGuard, mở website HTTP/HTTPS, nhấn biểu tượng rồi **Scan Current Website**.

Sau khi cập nhật source: build lại, bấm Reload extension tại trang quản lý extension. `npm.cmd run dev` tự build khi source đổi; không load source hoặc Vite dev server làm extension.

## Tích hợp backend

Mặc định `http://127.0.0.1:8000`. Extension đọc `/openapi.json` để chọn đúng API đã đối chiếu với source hiện có; không thử endpoint khác khi inference thất bại.

| Backend | Model information | Scan | Screenshot |
| --- | --- | --- | --- |
| U-PIG, thư mục `../be upig` | `GET /api/v1/model` | `POST /api/v1/scans` với `{url}`, sau đó poll `GET /api/v1/scans/{scan_id}` | `GET /api/v1/screenshots/{preview_id}` |
| CrossPhire, thư mục `../PhishGuard_Backend` | `GET /model-info` | `POST /scan` với `{url}` | `GET /scans/{scan_id}/screenshot` |

Cả hai dùng `GET /health`. **Settings → Test connection** kiểm tra health và metadata model. U-PIG trả một chế độ suy luận đã chọn; các chế độ khác hiển thị **Not returned**, không bịa xác suất. CrossPhire hiển thị từng chế độ có dữ liệu. Adapter kiểm label, xác suất, bằng chứng và mode trước khi đưa lên UI. Giữ nguyên nhãn backend vì U-PIG dùng threshold riêng từng mode, không tự áp threshold 50%.

Không train hoặc chạy PyTorch trong extension. Khi backend hoặc model chưa sẵn sàng, hiển thị lỗi và Retry, không dùng mock prediction. Backend đang chạy phải cung cấp OpenAPI contract tương ứng. URL website cần dùng port HTTP/HTTPS tiêu chuẩn; U-PIG giới hạn 2048 ký tự.

## Chức năng và quyền

- Popup 400px: website hiện tại, skeleton, prediction, xác suất, inference mode, timestamp, ảnh xem trước và cảnh báo phishing.
- Quét thủ công; offscreen document tạo Web Worker để tiếp tục quét khi đóng popup. Timeout toàn lượt 180 giây; watchdog xử lý lượt bị gián đoạn. Chỉ một lượt đồng thời; nhấn trùng không gửi thêm request.
- Badge riêng từng tab: `...`, `OK`, `WARN`, `!`. Đổi URL sẽ xóa trạng thái, không gắn kết quả cũ vào trang mới.
- Lịch sử `chrome.storage.local`: URL bỏ query/fragment, domain, prediction, probability, model, mode, timestamp, status; tìm kiếm, lọc, xóa từng bản ghi hoặc toàn bộ; giới hạn 10–500.
- URL đầy đủ và trạng thái quét chỉ nằm trong `storage.session`; không lưu HTML, ảnh hoặc API key vào lịch sử. Không tự gửi lại lượt quét khi khởi động lại Chrome/reload extension.
- Auto Scan tắt. Notifications mặc định tắt; chỉ xin optional permission khi người dùng bật và lưu. Không tự chặn/đóng/chuyển hướng website.
- Dashboard mở URL đã cấu hình (mặc định `http://localhost:5173`); lịch sử extension độc lập với frontend.

Manifest chỉ cấp host localhost/127.0.0.1, `activeTab`, storage, offscreen, alarms; không content script, không quyền `tabs` hoặc đọc mọi website. `activeTab` lấy URL sau thao tác bấm toolbar. Chrome host patterns không giới hạn port.

Đổi port localhost trong Settings. Với backend HTTPS mới, thêm đúng host vào `manifest.json`, ví dụ `"https://api.example.com/*"`, build và reload trước khi lưu địa chỉ mới. Không cấp `<all_urls>`. Remote backend phải dùng HTTPS.

Nếu backend bật `X-API-Key`, extension báo cần xác thực. Không nhúng/lưu shared secret; cần proxy giữ key ở server và có phiên đăng nhập riêng cho người dùng. Bản này chưa triển khai login/proxy.

## Kiểm thử

```powershell
npm.cmd run typecheck
npm.cmd run lint
npm.cmd run build
npm.cmd test
# Cần Chrome và backend thật đang chạy:
npm.cmd run test:browser
```

`tests/extension.test.ts` kiểm adapter cả hai backend (có response U-PIG thực đã thu), missing modalities, threshold theo backend, schema không hợp lệ, URL, HTTP 401/403/404/429/500/503, offline, timeout, polling job lỗi, quét trùng/đồng thời, đổi URL, toàn bộ badge, watchdog, giới hạn/xóa lịch sử và manifest build. Fixture chỉ dùng trong test, không được đóng gói vào extension.

`tests/browser.mjs` dùng Chrome DevTools Protocol và profile tạm, không thay đổi profile duyệt web của bạn. Có thể đặt `CHROME_PATH` và `PHISHGUARD_API_BASE` nếu khác mặc định. Trong môi trường sandbox không chạy được renderer, cờ `PHISHGUARD_NO_SANDBOX=1` chỉ dùng cho tiến trình kiểm thử với profile tạm; extension không yêu cầu cờ này khi sử dụng bình thường.

Đã kiểm tra Chrome 157 headless với U-PIG thật (epoch 8, CPU): kết nối model, nút Test connection, lưu/validate Settings, thao tác toolbar cấp activeTab, quét website, đóng giao diện khi quét, hiển thị verdict/probability/mode/ảnh, badge, đổi URL khi quét, chuyển tab, tìm kiếm/lọc/xóa lịch sử, trang không hỗ trợ, lỗi offline và kết nối lại. `example.com` trả **BENIGN**, phishing probability **0.01245097**, mode **full**. Đây là kết quả model, không phải nhãn ground truth hoặc bằng chứng đánh giá độ chính xác tổng thể.

Giới hạn kiểm thử: popup gốc của toolbar không render ổn định qua headless automation; suite kích hoạt toolbar thật để cấp activeTab rồi kiểm cùng tài liệu popup trong tab extension. Chưa xác nhận trực quan popup gốc bằng thao tác tay, thông báo hệ điều hành hoặc Edge. Các trường hợp PHISHING, lỗi HTTP và thiếu modality được kiểm bằng fixture/mô phỏng, không truy cập website phishing thật. Backend không bị dừng hoặc sửa trong kiểm thử offline: chỉ đổi API Base URL trong profile kiểm thử rồi khôi phục.

Chrome giới hạn thời gian chờ fetch ở service worker; request dài và polling chạy trong Web Worker thuộc offscreen document. Xem [Chrome service worker lifecycle](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle).

Source, cấu hình, test và output build là các file cần thiết; không tạo ZIP trùng lặp. `dist/` là thư mục load unpacked.
# PBL6_extension
