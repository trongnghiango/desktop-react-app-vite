# NOTES


Thư mục `packages` trong boilerplate `vite-electron-builder` (hay `electron-vite`) **không phải** là nơi bạn đặt các package của riêng mình như trong một monorepo. Thay vào đó, nó chứa các **gói nội bộ (internal packages)**, là những đoạn script và cấu hình cốt lõi để boilerplate có thể hoạt động.

Đây là vai trò của từng thành phần và lời khuyên về việc chỉnh sửa chúng.

---

### Vai trò của từng thư mục và file

Đây là những thư mục mà bạn **sẽ làm việc chủ yếu**:

*   ✅ **`main`**:
    *   **Vai trò:** Đây là nơi chứa code cho **Tiến trình chính (Main Process)** của Electron. Nó giống như "backend" của ứng dụng desktop của bạn. Mọi logic liên quan đến việc tạo/quản lý cửa sổ, xử lý các sự kiện hệ thống (như đóng ứng dụng), tương tác với hệ điều hành (như mở file dialog), và lắng nghe các sự kiện IPC từ `renderer` đều được viết ở đây.
    *   **Lời khuyên:** Bạn sẽ dành phần lớn thời gian làm việc với Node.js và API của Electron trong thư mục này.

*   ✅ **`preload`**:
    *   **Vai trò:** Đây là "cây cầu" an toàn kết nối giữa thế giới của `main` (có toàn quyền truy cập Node.js) và thế giới của `renderer` (chạy trong môi trường web an toàn). Script trong `preload` có thể truy cập cả API của Node.js và DOM của trang web. Nó được dùng để tạo ra một API an toàn (`contextBridge`) mà giao diện React của bạn có thể gọi để yêu cầu tiến trình chính thực hiện các tác vụ.
    *   **Lời khuyên:** Bạn sẽ chỉnh sửa file trong này mỗi khi cần tạo một kênh giao tiếp mới giữa frontend và backend.

*   ✅ **`renderer`**:
    *   **Vai trò:** Đây là nơi chứa toàn bộ code **giao diện người dùng (Frontend)** của bạn. Toàn bộ ứng dụng React, bao gồm components, state, styling,... đều nằm ở đây. Đây là những gì người dùng nhìn thấy và tương tác.
    *   **Lời khuyên:** Đây là nơi bạn sẽ xây dựng giao diện ứng dụng của mình.

---

### Những phần bạn nên TRÁNH thay đổi

Đây là những gói nội bộ, là "bộ máy" của boilerplate. Việc thay đổi chúng có thể làm hỏng toàn bộ quy trình phát triển và build ứng dụng nếu bạn không hiểu rõ mình đang làm gì.

*   ❌ **`electron-versions`**:
    *   **Vai trò:** Chứa các script và cache để quản lý các phiên bản Electron khác nhau. Khi bạn cài đặt dự án, nó giúp tải về đúng phiên bản Electron cho hệ điều hành của bạn.
    *   **Lời khuyên:** **Tuyệt đối không chạm vào.** Đây là cơ sở hạ tầng cốt lõi của công cụ.

*   ❌ **`integrate-renderer`**:
    *   **Vai trò:** Chứa các script "ma thuật" để tích hợp Vite Dev Server (chạy cho `renderer`) với cửa sổ Electron trong chế độ phát triển. Nó chịu trách nhiệm khởi động server Vite và cung cấp URL cho tiến trình `main` để tải giao diện, cũng như kích hoạt hot-reload.
    *   **Lời khuyên:** **Tuyệt đối không chạm vào.** Thay đổi thư mục này sẽ làm hỏng trải nghiệm phát triển (`npm run dev`).

*   ❌ **`dev-mode.js` & `entry-point.mjs`**:
    *   **Vai trò:** Đây là các điểm vào (entry points) cho các kịch bản khác nhau. `dev-mode.js` có thể là điểm bắt đầu khi chạy `npm run dev`. `entry-point.mjs` có thể là điểm vào chính cho ứng dụng sau khi đã được build ra sản phẩm cuối cùng. Chúng điều phối việc khởi chạy các tiến trình và thiết lập môi trường.
    *   **Lời khuyên:** **Tuyệt đối không chạm vào.** Đây là phần lõi của logic khởi động ứng dụng.

### Bảng tóm tắt

| Tên | Vai trò | Có nên thay đổi? |
| :--- | :--- | :--- |
| **`main`** | Backend, quản lý cửa sổ (Node.js & Electron API) | ✅ **Có, đây là nơi làm việc chính** |
| **`preload`** | Cầu nối an toàn giữa `main` và `renderer` | ✅ **Có, để tạo kênh giao tiếp** |
| **`renderer`** | Frontend, giao diện người dùng (React, CSS,...) | ✅ **Có, đây là nơi làm việc chính** |
| `electron-versions` | Quản lý phiên bản Electron | ❌ **Không, đây là phần lõi của công cụ** |
| `integrate-renderer` | Tích hợp Vite Dev Server vào Electron | ❌ **Không, đây là phần lõi của công cụ** |
| `dev-mode.js` | Kịch bản khởi động chế độ phát triển | ❌ **Không, đây là phần lõi của công cụ** |
| `entry-point.mjs` | Kịch bản khởi động cho bản build sản phẩm | ❌ **Không, đây là phần lõi của công cụ** |

**Tóm lại:** Hãy coi `main`, `preload`, và `renderer` là "khu vực làm việc" của bạn. Các thư mục và file còn lại là "phòng máy" của boilerplate, hãy để nó tự vận hành.
