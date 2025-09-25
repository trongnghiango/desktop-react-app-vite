Bạn đã chỉ ra một điểm cực kỳ chính xác và quan trọng, là mấu chốt để hiểu rõ kiến trúc state trong Electron. Bạn nói đúng:

> **Store trong `main` process KHÔNG phải là Redux/Context.** Nó không tự động "kết nối" và làm cho giao diện React re-render. Nó chỉ là nơi lưu trữ dữ liệu thuần túy (plain data) ở "backend".

Vậy câu hỏi là: Làm thế nào để sự thay đổi trạng thái ở `main` process (ví dụ: người dùng đăng nhập thành công, thông tin được lưu vào `authStore`) có thể được **phản ánh lên giao diện** và làm cho component React hiển thị "Xin chào, [Tên người dùng]"?

Câu trả lời nằm ở một mô hình giao tiếp hai chiều, trong đó `main` process phải chủ động **thông báo** cho `renderer` process về sự thay đổi.

### Mô hình Đồng bộ hóa Trạng thái (State Synchronization Model)

Chúng ta có hai loại trạng thái riêng biệt:

1.  **Trạng thái Ứng dụng (Application State):** Nguồn chân lý duy nhất (Single Source of Truth). Nó nằm trong `main` process (ví dụ: `authStore`).
2.  **Trạng thái Giao diện (UI State):** Một bản sao, một "view" của Trạng thái Ứng dụng, được cache lại ở `renderer` process. Nó nằm trong React (Context, Redux, `useState`...). **Giao diện React CHỈ re-render khi trạng thái này thay đổi.**

Nhiệm vụ của chúng ta là xây dựng một cây cầu để đồng bộ hóa chúng.

#### Luồng hoạt động

1.  **Renderer -> Main (Yêu cầu thay đổi):** Người dùng thực hiện một hành động (nhấn nút Login). Giao diện gọi một hàm IPC (`invoke`) để yêu cầu `main` process xử lý.
2.  **Main (Xử lý và cập nhật "Nguồn chân lý"):** `main` process nhận yêu cầu, gọi service, và cập nhật `authStore` của nó.
3.  **Main -> Renderer (Thông báo sự thay đổi):** Đây là bước quan trọng nhất. Ngay sau khi `authStore` thay đổi, `main` process phải gửi một sự kiện IPC (`webContents.send`) đến tất cả các cửa sổ liên quan để nói rằng: "Này, trạng thái xác thực đã thay đổi rồi đấy!".
4.  **Renderer (Lắng nghe và cập nhật UI State):** Giao diện React (thông qua `preload`) luôn lắng nghe sự kiện này. Khi nhận được thông báo, nó sẽ lấy dữ liệu mới và cập nhật state của chính nó (ví dụ: gọi hàm `setUser` của Context). **Chính hành động này sẽ kích hoạt re-render.**

### Triển khai chi tiết

Hãy nâng cấp ví dụ về `authStore` để hiện thực hóa mô hình này.

#### Bước 1: Nâng cấp `auth.store.js` để có khả năng "phát" sự kiện

Store bây giờ không chỉ lưu dữ liệu mà còn phải biết cách thông báo cho các cửa sổ.

**`main/store/auth.store.js`**

```javascript
import { BrowserWindow } from 'electron';

class AuthStore {
  // ... (constructor như cũ)

  setCurrentUser(user) {
    this.currentUser = user;
    // SAU KHI THAY ĐỔI STATE, GỌI HÀM NÀY
    this.notifyWindowsOfChange();
  }

  getCurrentUser() {
    return this.currentUser;
  }
  
  // HÀM MỚI: Gửi sự kiện đến tất cả các cửa sổ
  notifyWindowsOfChange() {
    // Lấy tất cả các cửa sổ đang mở
    const windows = BrowserWindow.getAllWindows();
    // Gửi dữ liệu user hiện tại qua kênh 'auth-state-changed'
    windows.forEach(win => {
      win.webContents.send('auth-state-changed', this.currentUser);
    });
  }
}

export const authStore = new AuthStore();
```

#### Bước 2: Nâng cấp `preload/index.js` để có thể "nghe"

`preload` cần cung cấp một cách để giao diện đăng ký một hàm callback để lắng nghe sự kiện từ `main`.

**`preload/index.js`**
```javascript
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('api', {
  // ... (các hàm invoke như cũ)
  
  // HÀM MỚI: Cho phép renderer đăng ký một hàm để lắng nghe
  // sự kiện 'auth-state-changed'
  onAuthStateChange: (callback) => {
    // Khi nhận được sự kiện, gọi hàm callback với dữ liệu (user)
    const listener = (event, user) => callback(user);
    ipcRenderer.on('auth-state-changed', listener);
    
    // Trả về một hàm để "hủy đăng ký" khi component bị unmount
    return () => {
      ipcRenderer.removeListener('auth-state-changed', listener);
    };
  }
});
```

#### Bước 3: Quản lý State trong React (ví dụ với Context)

Đây là nơi chúng ta kết nối mọi thứ lại với nhau. Chúng ta sẽ tạo một `AuthContext` để lưu trữ bản sao của trạng thái xác thực.

**`renderer/src/contexts/AuthContext.jsx`**
```jsx
import React, { createContext, useState, useEffect, useContext } from 'react';

const AuthContext = createContext();

export function useAuth() {
  return useContext(AuthContext);
}

export function AuthProvider({ children }) {
  const [currentUser, setCurrentUser] = useState(null);

  useEffect(() => {
    // KHI PROVIDER ĐƯỢC MOUNT, NÓ SẼ ĐĂNG KÝ LẮNG NGHE
    // SỰ THAY ĐỔI TỪ MAIN PROCESS
    const unsubscribe = window.api.onAuthStateChange((user) => {
      console.log('Received auth state change from main process:', user);
      // CẬP NHẬT UI STATE, KÍCH HOẠT RE-RENDER!
      setCurrentUser(user);
    });

    // Khi component unmount, hủy đăng ký lắng nghe để tránh memory leak
    return () => {
      unsubscribe();
    };
  }, []); // Mảng rỗng đảm bảo useEffect chỉ chạy một lần

  const value = {
    currentUser,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

**Bọc ứng dụng của bạn trong `AuthProvider`:**

**`renderer/dashboard/src/main.jsx`**
```jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import { AuthProvider } from '../../contexts/AuthContext'; // Import provider

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <AuthProvider>
      <App />
    </AuthProvider>
  </React.StrictMode>
);
```

**Bây giờ, bất kỳ component nào cũng có thể truy cập trạng thái này:**
```jsx
import { useAuth } from '../../contexts/AuthContext';

function UserProfile() {
  const { currentUser } = useAuth();

  if (!currentUser) {
    return <div>Please log in.</div>;
  }

  return <h1>Welcome, {currentUser.name}!</h1>;
}
```

### Tổng kết luồng hoàn chỉnh

1.  Người dùng nhấn Login trong `LoginWindow`.
2.  Giao diện gọi `window.api.auth.login(username, password)`.
3.  `main` process xử lý, xác thực thành công.
4.  `main` process gọi `authStore.setCurrentUser({ id: 1, name: 'John Doe' })`.
5.  Trong `setCurrentUser`, hàm `notifyWindowsOfChange` được gọi.
6.  `main` process gửi sự kiện `auth-state-changed` với payload là `{ id: 1, name: 'John Doe' }` đến `DashboardWindow`.
7.  Trong `AuthProvider` của `DashboardWindow`, `useEffect` đang lắng nghe. Nó nhận được sự kiện.
8.  Hàm callback `(user) => setCurrentUser(user)` được thực thi.
9.  State `currentUser` trong `AuthProvider` thay đổi.
10. Tất cả các component con đang dùng `useAuth()` (như `UserProfile`) sẽ tự động re-render với dữ liệu mới.

Kiến trúc này giữ cho code của bạn cực kỳ sạch sẽ: `main` process quản lý trạng thái toàn cục, `renderer` process chỉ là một "view" phản ứng lại các thay đổi đó.
