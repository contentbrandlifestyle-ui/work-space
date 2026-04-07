# Luồng Xác thực & Onboarding — iruKa App

> **Mục đích:** Tài liệu tham chiếu cho toàn bộ luồng đăng nhập, đăng ký và onboarding bé.
> Các tình huống được liệt kê đầy đủ để làm cơ sở triển khai và test.

---

## 📖 Giải Thích Thuật Ngữ

| Thuật ngữ | Nghĩa tiếng Việt |
|---|---|
| **Middleware** | Lớp bảo vệ chạy ở máy chủ — kiểm tra quyền vào trang TRƯỚC khi trang load |
| **Token / accessToken** | Thẻ đăng nhập — lưu trong trình duyệt, xác nhận user đã đăng nhập |
| **refreshToken** | Thẻ làm mới — dùng để lấy token mới khi token cũ hết hạn (không cần login lại) |
| **Cookie** | File nhỏ lưu trong trình duyệt — chứa token để gửi lên server mỗi request |
| **Redirect** | Chuyển hướng tự động sang trang khác |
| **Guard** | Bảo vệ trang — chỉ cho vào nếu đủ điều kiện (đã login, đã có hồ sơ...) |
| **Happy Path** | Luồng suôn sẻ — mọi thứ đúng, không có lỗi |
| **Edge Case** | Tình huống bất thường / biên — hiếm xảy ra nhưng cần xử lý |
| **OAuth** | Đăng nhập bên thứ 3 — qua Google/Facebook thay vì tạo tài khoản riêng |
| **OTP** | Mã xác thực 1 lần — gửi qua SMS/email, có thời hạn |
| **Onboarding** | Luồng khởi tạo — các bước sau đăng ký lần đầu (tên phụ huynh, tạo bé, chọn môn) |
| **profile_status** | Trạng thái hồ sơ bé — `incomplete` (chưa xong khảo sát) / `active` (đã xong) |
| **resolvePostLoginRoute** | Hàm tính toán sau khi login: đi về trang nào phù hợp nhất |
| **Zod** | Thư viện kiểm tra dữ liệu — validate form trước khi gửi lên server |
| **Toast** | Thông báo nhỏ popup tạm thời — xuất hiện góc màn hình vài giây rồi tự mất |
| **Spinner** | Vòng xoay loading — hiển thị khi đang chờ kết quả |
| **inline error** | Lỗi hiển thị ngay tại chỗ — thường xuất hiện dưới ô nhập bị lỗi |
| **on blur** | Khi user rời khỏi ô nhập — xảy ra khi click ra ngoài ô input |
| **Fallback** | Phương án dự phòng — nếu xử lý chính thất bại thì làm gì thay thế |
| **Zustand store** | Kho lưu trữ dữ liệu tạm trên trình duyệt — lưu trạng thái app (user, bé đang chọn...) |
| **Interceptor** | Đoạn code tự động chạy trước/sau mọi request — dùng để xử lý 401 tự động |
| **Multipart / form-data** | Định dạng gửi file lên server (ảnh, video...) |
| **Rate limit** | Giới hạn số lần gọi — VD: chỉ gửi OTP tối đa 5 lần, vượt thì bị khóa |
| **Snapshot** | Bản chụp năng lực bé tại thời điểm khảo sát — AI phân tích ra "Tọa Độ Vàng" |
| **Axis** | Trục năng lực — VD: foundation (nền tảng), support (hỗ trợ), approach (thiên hướng), interest (sở thích) |
| **Score percent** | Điểm phần trăm — mức độ phát triển của bé theo từng trục |
| **Level label** | Nhãn mức độ — VD: "Tự tin", "Đang phát triển", "Đang nảy mầm", "Cần thêm hỗ trợ" |
| **SSR / Server Component** | Render phía server — trang chạy trên máy chủ, không có quyền truy cập browser (window, localStorage) |
| **`use client`** | Chỉ thị Next.js — bắt buộc thêm khi component dùng state, event, browser API |
| **Suspense** | Cơ chế Next.js — bọc component dùng `useSearchParams()` để tránh lỗi build |
| **R2 Storage** | Dịch vụ lưu trữ file của Cloudflare — dùng để lưu ảnh avatar bé |

---

## 1. Bức Tranh Toàn Cảnh — Hành Trình Người Dùng

```
Người dùng mở app.irukaedu.vn/
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│                    MIDDLEWARE (server)                      │
│  Kiểm tra token trước khi hiển thị bất kỳ trang nào        │
└─────────────────────────────────────────────────────────────┘
         │
         ├── Đã login  ──────────────────────────────────────► /learn/today ✅
         │
         └── Chưa login ──────────────────────────────────────► /login
                  │
                  ├── [Đã có tài khoản] Đăng nhập ──────────► /learn/today ✅
                  │
                  ├── [Quên mật khẩu] Reset password ────────► /login → đăng nhập lại
                  │
                  └── [Chưa có tài khoản] Đăng ký
                           │
                           ▼
                    Onboarding (3 bước)
                    Bước 1: Tạo hồ sơ bé
                    Bước 2: Chọn môn học
                    Bước 3: Làm khảo sát
                           │
                           ▼
                    Xem kết quả "Tọa Độ Vàng"
                           │
                           ▼
                    /learn/today ✅
```

---

## 2. Entry Point & Middleware Guard

**File:** `src/middleware.ts` — Chạy server-side trước mọi trang load.

```
Mở bất kỳ URL nào
    │
    ├── pathname === "/"
    │       ├── Có token  → redirect /learn/today         ✅ Đã fix (P0)
    │       └── Không token → redirect /login             ✅ Đã fix (P0)
    │
    ├── pathname là AUTH_PAGE (/login, /register, /forgot-password)
    │       ├── Có token + ?redirect= an toàn → redirect về URL đó
    │       └── Có token, không có redirect → /learn/today
    │
    ├── pathname là PROTECTED_PAGE (/learn, /onboarding, /profile, /settings)
    │       ├── Có token  → cho vào
    │       └── Không token → /login?redirect={pathname}
    │
    ├── Token HẾT HẠN (có refresh_token — xử lý bởi client.ts)
    │       ├── Refresh OK  → tự retry request gốc, user không biết  ✅
    │       └── Refresh FAIL → xóa cookie → /login?redirect=...      ✅
    │
    └── Các trang khác → cho qua bình thường
```

### 4 Luồng Vào App

**Luồng A — User MỚI (chưa có tài khoản):**
```
Mở app → middleware → /login → click "Đăng ký ngay"
    → /register (4 bước) → auto-login
    → resolvePostLoginRoute:
        full_name rỗng?   → /register?step=complete
        không có bé?      → /onboarding/create-child
        bé incomplete?    → /onboarding/select-subject?child_id=...
        bé có môn rồi?    → /onboarding/survey?child_id=&subject_id=
        tất cả active?    → /learn/today ✅
```

**Luồng B — User ĐÃ CÓ Tài Khoản (login lại):**
```
Mở app → middleware → /login
    → Nhập SĐT/Email + Mật khẩu → doLogin()
    → resolvePostLoginRoute → /learn/today (hoặc onboarding nếu chưa xong) ✅
```

**Luồng C — User ĐÃ ĐĂNG NHẬP (mở lại app):**
```
Mở app.irukaedu.vn/
    → Middleware: có token + pathname="/" → redirect /learn/today ✅ (đã fix P0)
    (/learn/today có guard riêng: nếu chưa onboard → redirect đúng bước)
```

**Luồng D — Google OAuth:**
```
/login → click Google → apiGoogleStart() → redirect Google
    → Google callback → /auth/google/callback
    → resolvePostLoginRoute → /learn/today ✅
```

**Luồng E — Facebook OAuth:**
```
/login → click Facebook → apiFacebookStart() → redirect Facebook
    → Facebook callback → /auth/facebook/callback
    → resolvePostLoginRoute → /learn/today ✅

Lưu ý: User mới (chưa có bé) → /onboarding/create-child (BE tự điền full_name từ FB)
        User cũ (bé active) → /learn/today
```

### Edge Cases Entry Point

| # | Tình huống | Xử lý hiện tại | Trạng thái |
|---|---|---|---|
| EC1 | Đã login vào `/login` | Middleware redirect `/learn/today` | ✅ |
| EC2 | Đã login vào `/register` | Middleware redirect `/learn/today` | ✅ |
| EC3 | Chưa login vào `/learn` | Middleware redirect `/login?redirect=/learn` | ✅ |
| EC4 | Vào `/` chưa login | Middleware redirect `/login` | ✅ **Đã fix P0** |
| EC5 | Vào `/` đã login | Middleware redirect `/learn/today` | ✅ **Đã fix P0** |
| EC6 | Đăng ký Google, full_name rỗng | Redirect `/register?step=complete` | ✅ |
| EC7 | Bé `incomplete` + đã chọn môn | Bỏ qua chọn môn, vào thẳng survey | ✅ |
| EC8 | Bé `incomplete` + chưa chọn môn | Về `/onboarding/select-subject` | ✅ |
| EC9 | `redirect` param trỏ về auth page | Middleware lọc, dùng `/learn/today` | ✅ |
| EC10 | Nhiều bé, có bé incomplete + bé active | Ưu tiên xử lý bé incomplete trước | ✅ |
| EC11 | Token hết hạn, vào trang bảo vệ | `client.ts` tự refresh; nếu fail → `/login` | ✅ Đã có sẵn |
| EC12 | `resolvePostLoginRoute` lỗi 401 | `isAuthError()` → im lặng, interceptor đã redirect | ✅ **Đã fix P1** |
| EC13 | `resolvePostLoginRoute` lỗi mạng | toast + fallback `/learn/today` (có guard) | ✅ **Đã fix P1** |

---

## 3. Luồng Đăng Nhập

### 3.1 — Đăng Nhập SĐT/Email + Mật Khẩu

```
[/login]
    │
    ├── Nhập sdt/email + mật khẩu
    │
    ├── apiLogin() ──────► THÀNH CÔNG
    │                           │
    │                           ├── Lưu accessToken cookie (1 ngày)
    │                           ├── Lưu refreshToken cookie (7 ngày)
    │                           ├── Zustand store updated
    │                           │
    │                           └── Redirect:
    │                               ├── Có ?redirect= param → về URL đó ✅
    │                               └── Không có → /learn/today
    │
    └── apiLogin() ──────► THẤT BẠI
                                │
                                ├── Sai mật khẩu → toast lỗi "Sai mật khẩu"
                                ├── Tài khoản không tồn tại → toast lỗi
                                └── Ở lại /login
```

### 3.2 — Đăng Nhập Google OAuth

> **Social login:** Hỗ trợ **Facebook** và **Google** (đã bỏ Apple — không triển khai iOS).

**Luồng kỹ thuật:**
```
User click Google [login/page.tsx]
    → handleSocialLogin('google')
        → apiGoogleStart(returnTo)          GET /api/v1/auth/google/start?return_to=...
            ← { authUrl }
        → window.location.href = authUrl    (rời app → Google OAuth)
            → Google xác thực xong
                → Backend redirect về /auth/google/callback?access_token=...
                    → google/callback/page.tsx
                        → check params.get('error')  ← nếu có → hiện lỗi từ backend
                        → đọc access_token → store.setTokens()
                        → resolvePostLoginRoute() → router.replace(route)
```

**Bảng flows:**

| # | Flow | Logic | Hiển thị User | Trạng thái |
|---|------|-------|---------------|------------|
| 1 | Happy: Google thành công | start → redirect → callback → token → route | Spinner "Đang xác thực..." → vào app | ✅ OK |
| 2 | Error: apiGoogleStart lỗi | catch → toast.error | Toast "Không thể kết nối Google. Thử lại sau." | ✅ OK |
| 3 | Error: Callback thiếu access_token | setStatus('error') | Card đỏ "Đăng nhập Google thất bại" | ✅ OK |
| 4 | Error: resolvePostLoginRoute lỗi | catch → setStatus('error') | Card đỏ + nút "Thử lại" | ✅ Đã fix |
| 5 | Edge: User đã login, click Google lại | Guard accessToken → redirect `/learn/today` | Redirect ngay, không trigger OAuth | ✅ Đã fix |
| 6 | Edge: Backend trả error param | check params.get('error') | Card đỏ + message từ backend | ✅ Đã fix |
| 7 | Apple click | — | Đã xoá khỏi UI | ✅ Xoá |

**File liên quan:**
- `src/components/SocialIcons.tsx` — UI 2 icon Facebook + Google
- `src/app/(auth)/login/page.tsx` — handleSocialLogin() + apiGoogleStart()
- `src/app/(auth)/auth/google/callback/page.tsx` — xử lý callback + error
- `src/features/auth/auth.api.ts` — apiGoogleStart() → GET /api/v1/auth/google/start

---

### 3.3 — Đăng Nhập Facebook OAuth ✅

> **Trạng thái:** Đã triển khai (BE test pass ✅). ⚠️ App đang **Development Mode** — chỉ Testers mới login được.

**Luồng kỹ thuật:**
```
User click Facebook [login/page.tsx]
    → handleSocialLogin('facebook')
        → apiFacebookStart(returnTo)         GET /api/v1/auth/facebook/start?return_to=...
            ← { authUrl }
        → window.location.href = authUrl     (rời app → Facebook)
            → Facebook xác thực xong
                → BE redirect về:
                  /auth/facebook/callback?access_token=...&session_id=...&user_id=...&provider=facebook&name=...&email=...
                    → facebook/callback/page.tsx
                        → check ?error=facebook_denied ← user từ chối → hiện card đỏ
                        → đọc access_token (bắt buộc) / email (có thể null nếu FB dùng SĐT)
                        → window.history.replaceState() ← xóa token khỏi URL history
                        → store.setTokens(access_token, '', session_id)
                        → resolvePostLoginRoute() → router.replace(route)
```

**Post-Login Routing — Lần đầu vs Lần sau:**

| # | Trường hợp | Điều kiện | resolvePostLoginRoute() dẫn về |
|---|---|---|---|
| A | **FB lần đầu — user mới** | `full_name` tự điền từ FB, `children` = [] | `/onboarding/create-child` |
| B | **FB lần đầu — email trùng TK cũ, chưa có bé** | Account đã link, `children` = [] | `/onboarding/create-child` |
| C | **FB lần đầu — email trùng, bé incomplete** | `profile_status = 'incomplete'` | `/onboarding/survey` hoặc `/select-subject` |
| D | **FB lần đầu — email trùng, đã xong** | `profile_status = 'active'` | `/learn/today` ✅ |
| E | **FB lần sau** | Đã có TK + bé active | `/learn/today` ✅ |
| F | **FB — full_name rỗng** | BE không set đúng | `/register?step=complete` (safety net) |

> **Lưu ý:** Facebook khác email/OTP — BE **tự điền `full_name`** từ FB profile khi tạo tài khoản.
> User mới qua FB **không qua bước "hoàn thiện hồ sơ"**, đi thẳng vào tạo bé.
> Hàm `resolvePostLoginRoute()` đã xử lý đúng tất cả cases.

**Bảng flows đầy đủ:**

| # | Flow | Logic | Hiển thị User | Trạng thái |
|---|------|-------|---------------|------------|
| 1 | Happy: FB thành công — lần sau | Token → setTokens → route | Spinner "Đang xác thực Facebook..." | ✅ |
| 2 | Happy: FB thành công — lần đầu (user mới) | Token → setTokens → resolveRoute | → `/onboarding/create-child` | ✅ |
| 3 | Error: apiFacebookStart lỗi | catch → toast.error | Toast "Không thể kết nối Facebook" | ✅ |
| 4 | Error: User từ chối (ấn Cancel) | `?error=facebook_denied` | Card đỏ "Đã huỷ đăng nhập Facebook" + nút Thử lại | ✅ |
| 5 | Error: Thiếu access_token | setStatus('error') | Card đỏ "Đăng nhập Facebook thất bại" | ✅ |
| 6 | Error: resolvePostLoginRoute lỗi 401 | `isAuthError()` → im lặng | Interceptor đã redirect | ✅ |
| 7 | Error: resolvePostLoginRoute lỗi mạng | toast.error → router.replace('/login') | Toast đỏ + về login | ✅ |
| 8 | Edge: User đã login, click FB lại | Guard accessToken → redirect `/learn/today` | Redirect ngay | ✅ |
| 9 | Edge: Token lộ trong URL | `window.history.replaceState` | Không thấy trong history | ✅ |
| 10 | Edge: Email null (FB dùng SĐT) | Bỏ qua email, chỉ dùng access_token | Không hiển thị gì thêm | ✅ |

**⚠️ Dev Mode — Thêm Tester:**
1. Admin vào `developers.facebook.com` → **iruka edu** → **Vai trò** → **Người dùng thử** → **Thêm người**
2. Tester nhận thông báo → vào `developers.facebook.com` → Chấp nhận lời mời

**File liên quan:**
- `src/components/SocialIcons.tsx` — UI 2 icon (không đổi)
- `src/app/(auth)/login/page.tsx` — handleSocialLogin() + apiFacebookStart() **[UPDATED]**
- `src/app/(auth)/auth/facebook/callback/page.tsx` **[NEW]**
- `src/features/auth/auth.api.ts` — apiFacebookStart() **[NEW]**
- `src/features/auth/auth.types.ts` — FacebookStartRes, FacebookCallbackParams **[NEW]**

---

## 4. Luồng Đăng Ký (4 bước)

```
[/register]
     │
     Bước 1: Nhập SĐT hoặc Email → Gửi OTP
     Bước 2: Nhập mã OTP 6 số → nhận otp_verified_token
     Bước 3: Đặt mật khẩu → Tạo tài khoản + auto-login
     Bước 4: Hoàn thiện hồ sơ → /onboarding/create-child
```

### Bước 1 — Nhập SĐT hoặc Email

> Hỗ trợ **cả SĐT Việt Nam** (`VN_PHONE_RE = /^(0[35789])\d{8}$/`) và **Email**.
> Schema dùng `superRefine` → message lỗi riêng từng trường hợp. `id1Value` không strip non-digits (giữ email).
> Outline động: `outline-lime-400` (hợp lệ) → `outline-red-400` (lỗi) → `outline-zinc-400` (mặc định).

| # | Tình huống | Logic | Hiển thị cho User |
|---|---|---|---|
| 1 | URL chứa `?identifier=` | `prefillIdentifier` từ searchParams → `defaultValues` | Ô nhập sẵn SĐT hoặc email |
| 2 | Ô trống, bấm Submit | Zod `min(1)` fail | ❌ *"Vui lòng nhập SĐT hoặc email"* |
| 3 | Nhập SĐT có chữ (abc...) | `superRefine` fail | ❌ *"Số điện thoại chỉ được nhập số"* |
| 4 | Nhập chưa đủ 10 số | `superRefine` fail | ❌ *"Số điện thoại phải gồm đúng 10 chữ số"* |
| 5 | Nhập đầu số sai (VD: 044, 02x) | `VN_PHONE_RE` fail | ❌ *"Đầu số không hợp lệ (hỗ trợ: 03, 05, 07, 08, 09)"* |
| 6 | SĐT +84 | `replace(/^\+84/, '0')` normalize | ✅ Tự chuyển về 0xxx trước validate |
| 7 | Nhập đúng 10 số, đầu số hợp lệ | Zod pass, `id1Channel = 'phone'` | ✅ Outline `lime-400` |
| 8 | Blur — SĐT hợp lệ | `apiCheckAccount(val, 'phone')` | Spinner quay → kiểm tra backend |
| 9 | Blur → SĐT đã đăng ký | `exists: true` → `setAlreadyExistsId(val)` | Ẩn nút "Gửi OTP". Hiện: *"Đã có tài khoản? Đăng nhập →"* |
| 10 | Blur → SĐT chưa đăng ký | `exists: false` | ✅ Hiện nút "Gửi mã OTP" |
| 11 | Nhập email đúng định dạng | `superRefine` email path pass | ✅ Outline `lime-400`, `id1Channel = 'email'` |
| 12 | Nhập email sai định dạng | `z.string().email()` fail | ❌ *"Email không đúng định dạng (vd: ten@gmail.com)"* |
| 13 | Blur — Email hợp lệ | `apiCheckAccount(val, 'email')` | Spinner → kiểm tra backend |
| 14 | Blur → Email đã đăng ký | `exists: true` | Ẩn nút "Gửi OTP". Hiện: *"Đã có tài khoản? Đăng nhập →"* |
| 15 | Bấm "Gửi mã OTP" → 200 OK | `generateOtp({ channel, identifier })` | Toast ✅ *"Đã gửi mã OTP!"*, chuyển Bước 2 |
| 16 | Backend **409** (race) | `status === 409` | Ẩn nút OTP, hiện prompt Đăng nhập |
| 17 | Backend **429** Too Many Requests | `status === 429` | Toast 🔴 *"Vui lòng chờ 60 giây trước khi gửi lại mã OTP."* |
| 18 | Backend **403** hoặc lỗi mạng | `status === 403 / 0` | Toast 🔴 riêng biệt theo loại lỗi |

### Bước 2 — Xác thực OTP (13 tình huống)

> Validate real-time khi gõ (`mode: 'onChange'`). Giới hạn 5 lần nhập sai.

| # | Tình huống | Logic | Hiển thị cho User |
|---|---|---|---|
| 16 | Ô OTP trống, Submit | Zod `length(6)` fail | ❌ *"Mã OTP gồm đúng 6 chữ số"* |
| 17 | Nhập dưới 6 số | Zod `length(6)` fail | ❌ *"Mã OTP gồm đúng 6 chữ số"* |
| 18 | Nhập chữ cái thay vì số | Zod `regex(/^\d+$/)` fail — **real-time khi gõ** | ❌ *"Chỉ nhập số"* |
| 19 | Nhập đúng 6 số → Submit | Gọi `POST /validate-otp` | Nút *"Đang xử lý..."* + spinner |
| 20 | OTP **đúng** | `valid=true` → `setStep(3)`, reset attempts về 0 | Toast ✅ *"Xác thực thành công!"* |
| 21 | OTP **sai** (< 5 lần) | `valid=false` → tăng `otpAttempts` +1 | ❌ *"Mã OTP không đúng (còn X lần thử)"* |
| 22 | OTP sai **lần thứ 5** | `otpAttempts >= 5` → block Submit | ❌ *"Đã nhập sai quá 5 lần. Vui lòng gửi lại mã mới."* |
| 23 | OTP **hết hạn** (5 phút) | `valid=false`, message contains "hết hạn" → reset `countdown=0` | ❌ *"Mã OTP đã hết hạn"* + nút **"Gửi lại ngay"** hiện ngay |
| 24 | OTP **đã dùng rồi** | `error_code=otp_already_used` | ❌ *"Mã OTP đã được sử dụng"* |
| 25 | OTP **không tồn tại** | `error_code=otp_not_found` | ❌ *"Mã OTP không tồn tại hoặc không hợp lệ"* |
| 26 | Lỗi mạng / Server 500 | `onError`, `status === 0 hoặc >= 500` | Toast 🔴 *"Không thể kết nối. Kiểm tra mạng và thử lại."* |
| 27 | Countdown 60s đang chạy | Timer state | Link *"Gửi lại (59s...)"* disabled |
| 28 | Hết countdown → bấm Gửi lại | `handleResendOtp()` → `form2.reset()` + generate-otp | Ô OTP **được xóa trắng** + Toast ✅ *"Đã gửi lại mã OTP!"* |

### Bước 3 — Mật khẩu

> Validate real-time khi gõ (`mode: 'onChange'`). Có nút toggle 👁 ẩn/hiện mật khẩu.

| # | Tình huống | Logic | Hiển thị |
|---|---|---|---|
| 29 | Mật khẩu < 8 ký tự | Zod `min(8)` fail — **real-time** | ❌ *"Mật khẩu phải có ít nhất 8 ký tự"* |
| 30 | Thiếu chữ hoặc số | Zod `regex` fail — **real-time** | ❌ *"Mật khẩu phải có cả chữ và số"* |
| 31 | Nhập lại không khớp | Zod `refine` fail — **real-time** | ❌ *"Mật khẩu không khớp"* |
| 32 | Bấm 👁 toggle hiện mật khẩu | `setShowPassword(true/false)` | Ô đổi `type=text/password` |
| 33 | Đăng ký thành công + auto-login OK | register → doLogin → setStep(4) | Toast ✅ *"Tài khoản đã được tạo!"* → Bước 4 |
| 34 | Đăng ký OK nhưng auto-login lỗi | catch → toast.success + redirect `/login?identifier=...` | Toast ✅ *"Tạo tài khoản thành công! Vui lòng đăng nhập..."* + ô sẵn SĐT |
| 35 | API lỗi **409** (đã tồn tại) | onError `status === 409` | Toast 🔴 *"SĐT/Email này đã được đăng ký. Vui lòng đăng nhập."* |
| 36 | API lỗi **422** (invalid) | onError `status === 422` | Toast 🔴 *"Thông tin không hợp lệ. Vui lòng kiểm tra lại."* |
| 37 | Lỗi mạng / **500** | onError `status === 0 hoặc >= 500` | Toast 🔴 *"Không thể kết nối. Kiểm tra mạng và thử lại."* |

### Bước 4 — Hồ sơ

> `POST /api/v1/auth/complete-profile` → `{ full_name, birth_date, gender }` → void
> Form mode: `all` (onChange + onBlur)

| # | Tình huống | Logic | Hiển thị |
|---|---|---|---|
| 38 | Bỏ trống họ tên → Submit | Zod `min(2)` fail | ❌ *"Họ tên phải có ít nhất 2 ký tự"* |
| 39 | Nhập 1 ký tự → click ra ngoài hoặc tiếp tục gõ | Zod `min(2)` fail **real-time** | ❌ *"Họ tên phải có ít nhất 2 ký tự"* |
| 40 | Nhập đủ họ tên (≥ 2 ký tự) | Zod pass | Ô bình thường |
| 41 | Bỏ trống ngày sinh → Submit | Zod `min(1)` fail | ❌ *"Vui lòng chọn ngày sinh"* |
| 42 | Nhập dở ngày sinh (VD: 20/12/yyyy) → click ra ngoài | `mode: 'all'` → onBlur trigger → Zod `min(1)` fail (value rỗng) | ❌ *"Vui lòng chọn ngày sinh"* |
| 43 | Nhập đủ ngày sinh hợp lệ | Zod pass | Ô bình thường |
| 44 | Giới tính | `defaultValues: { gender: 'male' }` — mặc định Nam | Card Nam được chọn sẵn |
| 45 | Submit thành công | `apiCompleteProfile()` → redirect | Toast ✅ *"Đăng ký hoàn tất! 🎉"* → `/onboarding/create-child` |
| 46 | API lỗi 401 | onError map | Toast 🔴 *"Phiên đăng nhập hết hạn..."* |
| 47 | API lỗi 422 | onError map | Toast 🔴 *"Thông tin không hợp lệ..."* |
| 48 | API lỗi 500 / mất mạng | onError map | Toast 🔴 *"Không thể kết nối..."* |

> ⚠️ **Cần fix:** Nút "Quay lại" vẫn hiển thị ở bước 4 — nếu quay về bước 3 bấm lại → lỗi 409.
> **Fix:** Đổi điều kiện `step > 1` → `step > 1 && step < 4`

---

## 5. Luồng Quên Mật Khẩu (3 bước)

> **File:** `src/app/(auth)/forgot-password/page.tsx`
> Backend đầy đủ: `generate-otp` → `validation-otp` → `reset-password`
> Sau reset: **toàn bộ sessions bị revoke** — user buộc phải đăng nhập lại.

```
[/forgot-password]
     │
     Bước 1: Nhập SĐT/Email → generate-otp (action=forgot_password)
             + blur check tài khoản tồn tại
             + rate limit 5 lần / khóa 5 phút
     │
     Bước 2: Nhập OTP 6 số → validation-otp → nhận otp_verified_token
             + Gửi lại OTP trực tiếp (KHÔNG về bước 1)
     │
     Bước 3: Nhập mật khẩu mới (min 8 ký tự, có chữ + số)
             + show/hide password
             → reset-password → redirect /login?identifier=...
```

### Bước 1 — Nhập SĐT/Email (18 tình huống)

> Hỗ trợ cả SĐT Việt Nam (`VN_PHONE_RE = /^(0[35789])\d{8}$/`) và Email.
> `mode: 'onChange'` — validate realtime. Outline: `lime-400` (OK) / `red-400` (lỗi) / `zinc-400` (mặc định).

| # | Tình huống | Logic | Hiển thị cho User |
|---|---|---|---|
| 1 | URL chứa `?identifier=` | prefill từ searchParams → `defaultValues` | Ô nhập sẵn giá trị |
| 2 | Để trống → Gửi | Zod `min(1)` fail | ❌ *"Vui lòng nhập email hoặc số điện thoại"* |
| 3 | Gõ SĐT có chữ (abc...) | `superRefine` fail | ❌ *"Số điện thoại chỉ được nhập số"* |
| 4 | SĐT quá ngắn (< 10 số) | `superRefine` fail | ❌ *"Số điện thoại phải gồm đúng 10 chữ số"* |
| 5 | SĐT sai đầu số (01x, 02x) | `VN_PHONE_RE` fail | ❌ *"Đầu số không hợp lệ (hỗ trợ: 03, 05, 07, 08, 09)"* |
| 6 | SĐT dạng +84 | `replace(/^\+84/, '0')` tự normalize | ✅ Tự chuyển về 0xxx trước validate |
| 7 | SĐT có khoảng trắng | `trim()` + `replace(/\D/g, '')` | ✅ Tự strip space |
| 8 | SĐT hợp lệ đang gõ | `mode: 'onChange'` + `id1Valid = true` | ✅ Outline `lime-400` |
| 9 | Blur — SĐT hợp lệ | `apiCheckAccount(v, 'phone')` | Spinner trong ô → kiểm tra backend |
| 10 | Blur → SĐT chưa có tài khoản | `exists: false` → `setError` + `setNotFound(true)` | ❌ *"Số điện thoại chưa có tài khoản"* + nút **"Đăng ký ngay →"** hiện ngay |
| 11 | Blur → SĐT có tài khoản | `exists: true` → không set lỗi | ✅ Outline xanh, sẵn sàng gửi OTP |
| 12 | Email đúng định dạng | `z.string().email()` pass | ✅ Outline `lime-400` |
| 13 | Email sai định dạng | `z.string().email()` fail | ❌ *"Email không đúng định dạng (vd: ten@gmail.com)"* |
| 14 | Blur → Email chưa có tài khoản | `apiCheckAccount(v, 'email')` → `exists: false` | ❌ *"Email chưa có tài khoản"* + nút **"Đăng ký ngay →"** hiện ngay |
| 15 | Nhấn "Đăng ký ngay →" | `router.push('/register?identifier=...')` | Chuyển sang trang đăng ký, tự điền sẵn SĐT/email |
| 16 | Nút “Gửi mã OTP” khi `notFound=true` | `disabled={... || notFound}` | Nút mờ (opacity 40%), khá bấm được |
| 17 | Nhấn “Gửi mã OTP” (account tồn tại) | `apiCheckAccount` lại trước khi gửi — guard thứ 2 | Nếu không tồn tại: block + hiện lỗi |
| 18 | Gửi OTP thành công | `apiGenerateOtp` 200 | ✅ Toast *"Đã gửi mã OTP!"* → chuyển bước 2 |
| 19 | Gửi OTP lần 6+ (vượt giới hạn) | `sendCount >= 5` → `startBlock(300s)` | ❌ *"Đã gửi OTP 5 lần. Vui lòng thử lại sau MM:SS"* |
| 20 | Backend trả 429 rate limit | Parse error message | ❌ Toast *"Đã gửi quá nhiều lần. Vui lòng thử lại sau 5 phút."* + Block |
| 21 | Network/server lỗi | catch generic | ❌ Toast *"Không thể gửi OTP, vui lòng thử lại sau."* |

### Bước 2 — Nhập OTP (đồng bộ logic với Đăng ký)

> Validate realtime (`mode: 'onChange'`). Giới hạn 5 lần nhập sai. Cooldown 60s khi gửi lại.
> OTP hết hạn → reset countdown về 0 để nút "Gửi lại" sáng lên ngay.

| # | Tình huống | Logic | Hiển thị cho User |
|---|---|---|---|
| 1 | Mở bước 2 | Bắt đầu countdown 60s | *"Nhập mã 6 số đã gửi đến [email/SĐT]"* + "Gửi lại sau 60s" |
| 2 | Gõ ký tự chữ | Zod `regex(/^\d+$/)` fail | ❌ *"Chỉ nhập số"* |
| 3 | Chưa đủ 6 số → Xác nhận | Zod `length(6)` fail | ❌ *"Mã OTP phải đúng 6 số"* |
| 4 | OTP sai lần 1-4 | `otpAttempts++`, `remaining = 5 - attempts` | ❌ *"Mã OTP không đúng (còn {X} lần thử)"* |
| 5 | OTP sai lần 5 | `otpAttempts >= 5` → disable input + button | ❌ *"Bạn đã nhập sai quá 5 lần."* |
| 6 | OTP hết hạn | Parse `expired/hết hạn` → `setCountdown(0)` | ❌ *"Mã OTP đã hết hạn. Vui lòng gửi lại."* + nút Gửi lại sáng ngay |
| 7 | Nhấn "Gửi lại" (cooldown = 0) | Gọi `apiGenerateOtp` trực tiếp, reset `otpAttempts` | ✅ Toast *"Đã gửi lại mã OTP!"* + countdown 60s |
| 8 | Gửi lại vượt 5 lần | `sendCount >= 5` → `startBlock(300s)` | ❌ *"Đã gửi OTP 5 lần. Thử lại sau MM:SS"* |
| 9 | Lỗi mạng / server | catch generic | ❌ Toast *"Không thể kết nối. Kiểm tra mạng và thử lại."* |
| 10 | OTP đúng (còn hạn) | `apiValidateOtp` 200 → `otp_verified_token` | ✅ Toast *"Xác thực thành công!"* → bước 3 |

### Bước 3 — Đặt mật khẩu mới

| # | Tình huống | Logic | Hiển thị cho User |
|---|---|---|---|
| 1 | Mật khẩu < 8 ký tự | Zod `min(8)` fail | ❌ *"Mật khẩu phải có ít nhất 8 ký tự"* |
| 2 | Mật khẩu thiếu chữ/số | Zod `regex` fail | ❌ *"Mật khẩu phải có cả chữ và số"* |
| 3 | Xác nhận không khớp | Zod `refine` fail | ❌ *"Mật khẩu xác nhận không khớp"* |
| 4 | Nhấn Eye icon | Toggle `type=password/text` | Hiện/ẩn mật khẩu cho từng ô riêng biệt |
| 5 | Reset thành công | `apiResetPassword` 200 | ✅ Toast *"Đặt lại mật khẩu thành công!"* → `/login?identifier=...` |
| 6 | Mật khẩu mới giống cũ | BE `auth_service.py` kiểm tra `verify_password` → `ValueError("same_password")` | ❌ Toast *"Vui lòng điền khác mật khẩu cũ."* ✅ **Đã fix (FE + BE)** |
| 7 | Token hết hạn | API 4xx `expired/token` | ❌ Toast *"Phiên xác thực đã hết hạn. Vui lòng bắt đầu lại."* → bước 1 |

---

## 6. Luồng Onboarding — Thêm Bé ⭐ (Bước 1/3)

```
[/onboarding/create-child]  ← Bước 1/3
    │
    ├── Form: Tên bé + Ngày sinh (3 dropdown) + Giới tính + Avatar (tuỳ chọn)
    │   Validation:
    │   - Tên bé: ≥2 ký tự, chỉ chữ cái
    │   - Ngày sinh: đủ Ngày/Tháng/Năm, bé 0–18 tuổi (chặn ở Zod)
    │   - Avatar: JPG/PNG/WEBP, ≤5MB, ≥100×100px (validate trước khi upload)
    │
    ├── Bấm "Tiếp tục →"
    │
    ├── apiCreateChild(data) ──────► THÀNH CÔNG (201)
    │       │
    │       ├── (Phase 2) Nếu user đã chọn ảnh:
    │       │   └── apiUploadChildAvatar(newChild.id, avatarFile)
    │       │           POST /api/v1/children/{id}/avatar (multipart/form-data)
    │       │           → Backend: validate → upload_buffer() → R2 Storage
    │       │           → Trả LearnerOut với avatar_url
    │       │           ├── Upload OK  → setCurrentChild(updatedChild có avatar_url)
    │       │           └── Upload FAIL → toast warning + setCurrentChild(newChild) - KHÔNG BLOCK
    │       │
    │       ├── setCurrentChild(child) → Zustand store
    │       └── Redirect: /onboarding/select-subject?child_id={newChild.id}
    │
    ├── apiCreateChild() ──────► LỖI 403 → Toast "Tài khoản phụ huynh không hợp lệ"
    ├── apiCreateChild() ──────► LỖI 400 → Toast "Thông tin bé không đúng định dạng"
    ├── apiCreateChild() ──────► LỖI 500 → Toast "Không thể kết nối máy chủ"
    │
    └── apiCreateChild() ──────► LỖI 401 (token hết hạn)
            │
            └── client.ts tự refresh silently:
                ├── Refresh OK → retry → tạo bé OK → redirect survey ✅
                └── Refresh FAIL → /login?redirect=/onboarding/create-child
```

**Storage path:** `avatars/children/{child_id}.{ext}` trên R2 (Cloudflare)
**API avatar:** `POST /api/v1/children/{id}/avatar` — Auth: Bearer token phụ huynh

---

## 7. Luồng Onboarding — Chọn Môn Học (Bước 2/3)

```
[/onboarding/select-subject?child_id={id}]   ← Bước 2/3
    │
    ├── 3 môn hiển thị: Toán (math_001) | Tiếng Việt (viet_001) | Tạo hình (art_001)
    │
    ├── Bấm 1 môn → PUT /api/v1/children/{id}/preferences
    │   { selected_subject_id: "math_001" }
    │   ├── 200 OK  → Redirect: /onboarding/survey?child_id=...&subject_id=math_001
    │   ├── 400     → Toast lỗi "Subject không hợp lệ"
    │   └── 500     → Toast lỗi "Không thể lưu môn học"
    │
    └── Thiếu child_id → Redirect: /onboarding/create-child
```

**Subject IDs hợp lệ (đồng bộ FE ↔ BE ↔ DB):**

| Môn | ID gửi lên | DB slug | Axis slug |
|---|---|---|---|
| Toán | `math_001` | `math_001` | `math` |
| Tiếng Việt | `viet_001` | `viet_001` | `vietnamese` |
| Tạo hình | `art_001` | `art_001` | `art` |

---

## 8. Luồng Onboarding — Khảo Sát Bé (Bước 3/3)

```
[/onboarding/survey?child_id=xxx&subject_id=viet_001]   ← Bước 3/3
    │
    ├── Guard thiếu child_id  → /onboarding/create-child
    ├── Guard thiếu subject_id → /onboarding/select-subject?child_id=...
    │
    ├── GET /snapshot-surveys/onboarding?child_id=...&subject_id=viet_001
    │   BE xử lý:
    │   ① birth_date bé → compute_age_band() → "age_34"|"age_45"|"age_56"
    │   ② normalize_subject("viet_001") → "vietnamese"
    │   ③ Query snapshot_axis_nodes:
    │      4 câu foundation (rating 0/5/10)
    │      1 câu support (rating 0/5/10)
    │      1 câu approach (single_choice)
    │      2 câu interest (multi_choice)
    │   → Trả về 8 SnapshotQuestionOut
    │
    ├── FE hiển thị từng câu, header màu theo trục:
    │   📚 foundation → Sky Blue
    │   🧠 support    → Violet
    │   ⚡ approach   → Amber
    │   🌟 interest   → Emerald
    │
    ├── Câu cuối → "Hoàn thành" → POST /snapshot-surveys/submit
    │   { child_id, subject_id, source:"app", answers:[...] }
    │   answers conversion:
    │   - rating      → answer_number: 0 | 5 | 10
    │   - single      → answer_value: "balanced"
    │   - multi       → answer_json: { values: ["music","toys"] }
    │   └── 201 OK → submission_id
    │
    ├── POST /snapshot-surveys/generate  (3–30s LLM)
    │   { child_id, submission_id, subject_id, recompute:true }
    │   ├── OK  → /onboarding/snapshot-report?snapshot_id=rpt_xxx&child_name=...&subject_id=...
    │   └── FAIL → toast warning + /learn/today (không block user)
    │
    └── Error cases:
        ├── 404 (chưa có data) → "Môn học chưa có bộ câu hỏi"
        ├── 500 (lỗi hệ thống) → "Lỗi hệ thống, thử lại sau"
        └── câu hỏi < 6      → Block form + "Dữ liệu câu hỏi chưa đủ"
```

**Tình huống đầy đủ màn hình Survey:**

| # | Tình huống | Hiển thị | Trạng thái |
|---|---|---|---|
| 1 | Load thành công 8 câu | Survey hiển thị đúng theo trục + màu | ✅ |
| 2 | Thiếu `child_id` | Redirect `/create-child` | ✅ |
| 3 | Thiếu `subject_id` | Redirect `/select-subject` | ✅ |
| 4 | `currentChild` null trong store | Text "Vui lòng chọn bé trước" | ✅ |
| 5 | Load câu hỏi → 404 | "Môn học chưa có bộ câu hỏi" + nút Quay lại | ✅ |
| 6 | Load câu hỏi → 500 | "Lỗi hệ thống, vui lòng thử lại sau" | ✅ |
| 7 | Câu hỏi < 6 | Block form + "Dữ liệu câu hỏi chưa đủ" | ✅ |
| 8 | Chưa chọn đáp án bấm "Tiếp" | Nút disabled | ✅ |
| 9 | Submit OK → generate đang chạy | Loading screen "iruKa đang phân tích chân dung của bé..." | ✅ |
| 10 | Generate OK | Toast + redirect `/onboarding/snapshot-report?snapshot_id=xxx` | ✅ |
| 11 | Generate FAIL | Toast warning + redirect `/learn/today` (không block user) | ✅ |
| 12 | Submit FAIL | toast.error, ở lại survey | ✅ |
| 13 | Reload giữa chừng | `child_id` giữ từ URL param | ✅ |

---

## 9. Kết Quả "Tọa Độ Vàng" → /learn/today ✅

```
[/onboarding/snapshot-report?snapshot_id=rpt_xxx&child_name=Bình&subject_id=math_001]
    │
    ├── Guard: thiếu snapshot_id → "Không tìm thấy kết quả" + nút "Vào học ngay"
    │
    ├── GET /api/v1/snapshot-surveys/reports/{snapshot_id}
    │   Auth: Bearer token (auto từ http client, xử lý 401 tự động)
    │   ├── isLoading → spinner "Đang tải kết quả phân tích..."
    │   ├── error/null → "Không thể tải kết quả" + nút "Bỏ qua, vào học ngay"
    │   └── 200 OK  → hiển thị:
    │             Header: tên bé + môn học + tuổi + stage_label
    │             4 AxisCard: trục năng lực, score_percent, level_label
    │             Tags sở thích (interest_tags) tách riêng
    │             Tags thiên hướng (approach_tags) tách riêng
    │             4 lời khuyên cho Mẹ (advice.items)
    │
    └── Bấm "Bắt đầu học"
            └── router.replace('/learn/today')  ← replace (không có lịch sử back)
                         ▼
                  /learn/today ✅
```

**Tình huống đầy đủ màn Snapshot Report:**

| # | Tình huống | Hiển thị | Trạng thái |
|---|---|---|---|
| 1 | URL có đủ params, fetch OK | Hiển full report (axes, tags, advice) | ✅ |
| 2 | Tên bé từ URL param | `child_name` được decode và hiển đúng | ✅ |
| 3 | Store trống (reload trang) | Lấy `child_name` từ URL param trước | ✅ |
| 4 | Không có `snapshot_id` | "Không tìm thấy" + giải thích + nút vào học | ✅ |
| 5 | API 401 (token hết hạn) | http client tự refresh, retry; nếu fail → `/login` | ✅ |
| 6 | API 404/500 | "Không thể tải kết quả" + bỏ qua | ✅ |
| 7 | `stage_label` null | Bỏ qua banner stage | ✅ |
| 8 | `stage_note` null | Chỉ hiển label, không hiển note | ✅ |
| 9 | `interest_tags` rỗng | Section sở thích ẩn | ✅ |
| 10 | `approach_tags` rỗng | Section thiên hướng ẩn | ✅ |
| 11 | `advice.items` rỗng | Section lời khuyên ẩn | ✅ |
| 12 | `axis_key` không khớp config | Dùng DEFAULT_COLOR (xám) | ✅ |
| 13 | Bấm "Bắt đầu học" | `router.replace('/learn/today')` | ✅ |
| 14 | Back từ `/learn/today` | Không quay lại đây (replace đã xóa history) | ✅ |

---

## 10. Tổng Hợp Trạng Thái Các Fix

| # | Mô tả | File | Loại | Trạng thái |
|---|---|---|---|---|
| 1 | Nút "Bắt đầu học" không có action | `src/app/page.tsx` | BUG | ✅ **Đã fix** |
| 2 | Middleware không guard `pathname="/"` | `src/middleware.ts` | BUG | ✅ **Đã fix** |
| 3 | HTTP interceptor 401 (token expired) | `src/lib/http/client.ts` | FEAT | ✅ **Đã có sẵn** |
| 4 | Catch block `resolvePostLoginRoute` không phân loại lỗi | `login/page.tsx` + `usePostLoginRedirect.ts` | BUG | ✅ **Đã fix** |
| 5 | Facebook OAuth | `SocialIcons` + BE | FEAT | ⏳ BE đang làm |
| 6 | UI Splash Page (logo cá heo + animation) | `src/app/page.tsx` | UX | ✅ **Đã làm** |
| 7 | Cache check tài khoản | `login/page.tsx` | PERF | ✅ **Đã làm** |
| 8 | Nút quay lại từ `/login` | `login/page.tsx` | UX | ✅ **Đã làm** |
| F1 | Middleware bỏ qua `?redirect=` khi login xong | `middleware.ts` | BUG | ✅ **Đã fix** |
| F2 | Survey page không redirect về `create-child` khi thiếu `child_id` | `survey/page.tsx` | BUG | ✅ Đã có guard |
| EC5 | User đã login, click Google → trigger OAuth thừa | `login/page.tsx` | BUG | ✅ **Đã fix** |

---

## 11. Kế Hoạch Fix (Code tham khảo)

### Fix F1 — `middleware.ts`

```ts
// TRƯỚC (sai):
if (AUTH_PAGES.some(p => pathname.startsWith(p)) && token) {
  return NextResponse.redirect(new URL('/learn/today', request.url));
}

// SAU (đúng):
if (AUTH_PAGES.some(p => pathname.startsWith(p)) && token) {
  const redirectTo = request.nextUrl.searchParams.get('redirect') ?? '/learn/today';
  return NextResponse.redirect(new URL(redirectTo, request.url));
}
```

### Fix F2 — `survey/page.tsx`

```ts
// Thêm guard đầu component:
if (!childId) {
  router.replace('/onboarding/create-child');
  return null;
}
```

---

## 12. Checklist Test Toàn Luồng

### Auth
- [ ] Đăng nhập → không có `?redirect=` → vào `/learn/today` ✅
- [ ] Token hết hạn trên `create-child` → refresh fail → login → quay lại `create-child`
- [ ] Nhập SĐT sai định dạng → hiển thị đúng message
- [ ] Blur SĐT hợp lệ, tài khoản không tồn tại → lỗi inline ngay
- [ ] Gửi OTP 5 lần → lần 6 bị block + đếm ngược 5 phút
- [ ] OTP đúng → sang bước tiếp
- [ ] OTP sai → lỗi inline "Mã OTP không đúng"
- [ ] Gửi lại OTP → không về bước 1, gọi API trực tiếp
- [ ] Show/hide password hoạt động từng ô
- [ ] Mật khẩu < 8 ký tự → lỗi inline
- [ ] Mật khẩu xác nhận không khớp → lỗi inline

### Quên Mật Khẩu
- [ ] Reset thành công → redirect `/login?identifier=...`

### Onboarding
- [ ] Tạo bé thành công → vào đúng `/onboarding/select-subject?child_id=xxx`
- [ ] Survey không có `child_id` → redirect về `create-child`
- [ ] Hoàn thành survey → vào `/onboarding/snapshot-report`
- [ ] Snapshot fail → vào `/learn/today` (không block)
- [ ] Bấm "Bắt đầu học" → `/learn/today`, không có lịch sử back
