# Rabe-ffi — Hướng dẫn Build & Sử dụng

C FFI binding cho thư viện [rabe](https://github.com/Fraunhofer-AISEC/rabe) (Attribute-Based Encryption viết bằng Rust), hỗ trợ CP-ABE và KP-ABE encrypt/decrypt cho các scheme AC17, BSW...

Repo: https://github.com/Aya0wind/Rabe-ffi

---

## Mục lục

- [Yêu cầu chung](#yêu-cầu-chung)
- [Build trên Linux](#build-trên-linux)
- [Build trên Windows (MSVC)](#build-trên-windows-msvc)
- [Build trên Windows (MinGW) — tùy chọn](#build-trên-windows-mingw--tùy-chọn)
- [Tích hợp vào project C/C++](#tích-hợp-vào-project-cc)
- [Vì sao cần Rust nightly](#vì-sao-cần-rust-nightly)
- [Lưu ý bảo mật](#lưu-ý-bảo-mật)
- [Known issues](#known-issues)

---

## Yêu cầu chung

Bất kể Linux hay Windows, quy trình build cần 3 thành phần:

1. **Rust toolchain (nightly)** — bắt buộc, không dùng được stable (xem lý do ở [phần riêng](#vì-sao-cần-rust-nightly)).
2. **`cargo-expand`** — build script (`build.rs`) dùng `cbindgen` với `with_parse_expand`, cần công cụ này để macro-expand code trước khi sinh header `.h`.
3. **Linker + C build tools** của hệ điều hành tương ứng (gcc trên Linux, MSVC Build Tools trên Windows).

---

## Build trên Linux

### 1. Cài Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

### 2. Cài build tools của hệ điều hành

```bash
# Debian / Ubuntu
sudo apt update && sudo apt install build-essential

# Fedora / RHEL
sudo dnf groupinstall "Development Tools"

# Arch
sudo pacman -S base-devel
```

### 3. Set toolchain nightly + cài cargo-expand

```bash
rustup default nightly
cargo install cargo-expand
```

### 4. Clone và build

```bash
git clone https://github.com/Aya0wind/Rabe-ffi.git
cd Rabe-ffi
cargo build --release
```

### 5. Output

```
target/release/librabe_ffi.so     # dynamic library
target/release/librabe_ffi.a      # static library
rabe.h                             # header sinh tự động bởi cbindgen (nằm ở thư mục gốc project)
```

### 6. Copy vào project của bạn

```bash
cp target/release/librabe_ffi.so /your/project/path
cp rabe.h /your/project/path/rabe.h
```

### 7. Compile & link project C/C++ trên Linux

```bash
g++ your_app.cpp -I/your/project/path -L/your/project/path -lrabe_ffi -o your_app
```

Chạy thử (cần trỏ đúng đường dẫn `.so` lúc runtime):

```bash
LD_LIBRARY_PATH=/your/project/path ./your_app
```

Hoặc copy `librabe_ffi.so` vào `/usr/local/lib` rồi chạy `sudo ldconfig` để không cần set `LD_LIBRARY_PATH` mỗi lần.

---

## Build trên Windows (MSVC)

### 1. Cài Visual Studio Build Tools (bắt buộc — cần trước khi cài Rust MSVC)

Tải: https://visualstudio.microsoft.com/visual-cpp-build-tools/

Trong installer, tick chọn:
- ✅ **Desktop development with C++**
  (kèm theo MSVC v143 build tools + Windows 10/11 SDK)

### 2. Cài Rust — dùng PowerShell hoặc CMD thuần (KHÔNG dùng MSYS2/Git Bash)

> ⚠️ Nếu chạy installer từ MSYS2/Git Bash, rustup sẽ tự chọn nhầm target `gnu` thay vì `msvc`. Luôn cài từ PowerShell hoặc CMD.

```powershell
Invoke-WebRequest -Uri https://win.rustup.rs/x86_64 -OutFile rustup-init.exe
.\rustup-init.exe
```

Khi hỏi installation options, chọn mặc định (Enter) — trên PowerShell/CMD thuần, rustup tự nhận diện `x86_64-pc-windows-msvc` làm host triple mặc định.

**Nếu máy đã từng cài rustup với target `gnu` từ trước**, sửa lại bằng:

```powershell
rustup target add x86_64-pc-windows-msvc
rustup toolchain install nightly-x86_64-pc-windows-msvc
rustup default nightly-x86_64-pc-windows-msvc
```

Kiểm tra:

```powershell
rustup show
```

Phải thấy:
```
Default host: x86_64-pc-windows-msvc
active toolchain: nightly-x86_64-pc-windows-msvc
```

### 3. Cài cargo-expand

```powershell
cargo install cargo-expand
```

### 4. Clone và build

```powershell
git clone https://github.com/Aya0wind/Rabe-ffi.git
cd Rabe-ffi
cargo build --release
```

### 5. Output

```
target\release\rabe_ffi.dll         # dynamic library
target\release\rabe_ffi.dll.lib     # import lib (dùng để link vào .dll)
target\release\rabe_ffi.lib         # static library
rabe.h                               # header
```

### 6. Tích hợp vào project Visual Studio

1. Copy `rabe_ffi.dll`, `rabe_ffi.dll.lib`, `rabe.h` vào project.
2. Project Properties → **VC++ Directories**:
   - Include Directories: thêm đường dẫn chứa `rabe.h`
   - Library Directories: thêm đường dẫn chứa `rabe_ffi.dll.lib`
3. Project Properties → **Linker → Input → Additional Dependencies**: thêm `rabe_ffi.dll.lib`
4. Copy `rabe_ffi.dll` vào cùng thư mục với file `.exe` output (hoặc thêm vào PATH) để chạy được lúc runtime.

Nếu muốn static-link (không cần phân phối `.dll` kèm theo), dùng `rabe_ffi.lib` thay vì `rabe_ffi.dll.lib` trong Additional Dependencies.

---

## Build trên Windows (MinGW) — tùy chọn

Chỉ dùng nếu toàn bộ project C++ của bạn (kể cả các thư viện khác như CryptoPP) đã build bằng MinGW/g++, để tránh trộn 2 runtime khác nhau.

```powershell
rustup toolchain install nightly-x86_64-pc-windows-gnu
rustup default nightly-x86_64-pc-windows-gnu
cargo install cargo-expand

git clone https://github.com/Aya0wind/Rabe-ffi.git
cd Rabe-ffi
cargo build --release
```

Nếu thiếu linker (`gcc.exe`/`ld.exe` not found), cài thêm:

```powershell
winget install --id=BrechtSanders.WinLibs.POSIX.UCRT -e
```
(hoặc dùng MSYS2: `pacman -S mingw-w64-x86_64-gcc`), rồi thêm vào PATH.

Output:
```
target\release\rabe_ffi.dll
target\release\librabe_ffi.dll.a   # import lib (định dạng MinGW)
target\release\librabe_ffi.a       # static lib
```

Link bằng g++:
```bash
g++ your_app.cpp -L./target/release -lrabe_ffi -I. -o your_app.exe
```

> Về hiệu năng: MinGW và MSVC dùng chung LLVM backend để sinh mã máy, nên tốc độ thực thi (đặc biệt là phần pairing/AES nặng tính toán) gần như tương đương nhau. Khác biệt chỉ nằm ở cơ chế linking/exception handling, không ảnh hưởng tốc độ ABE.

---

## Tích hợp vào project C/C++

Ví dụ tối thiểu dùng `rabe.h`:

```c
#include "rabe.h"

int main() {
    Ac17SetupResult keys = rabe_ac17_init();
    // ... dùng keys.public_key, keys.master_key
    rabe_ac17_free_public_key(keys.public_key);
    rabe_ac17_free_master_key(keys.master_key);
    return 0;
}
```

Xem thêm ví dụ đầy đủ trong unit test `#[cfg(test)] mod test` của file `src/cp_abe/ac17.rs` trong source Rabe-ffi.

---

## Vì sao cần Rust nightly

`build.rs` của project dùng `cbindgen` với cấu hình:

```rust
cbindgen::Builder::new()
    .with_parse_expand(&["rabe-ffi"])
    ...
```

`with_parse_expand` yêu cầu macro-expand toàn bộ crate trước khi sinh header (vì code dùng các macro `to_json_impl!`, `from_json_impl!`, `free_impl!` để sinh hàm C export — `cbindgen` không tự đọc được nội dung macro nếu không expand trước). Việc expand này cần `cargo-expand`, mà `cargo-expand` phụ thuộc `-Z unpretty`, chỉ có ở **nightly rustc**.

---

## Lưu ý bảo mật

- `Cargo.toml` hiện ghim `rabe = "0.2.6"`, trong khi bản mới nhất trên crates.io là `0.4.2`. Theo changelog chính thức của `rabe`, bản `v0.3.1` trở đi mới chuyển sang dùng `aes-gcm` (AEAD, có auth tag) cho tầng mã hoá đối xứng nội bộ — các bản trước đó (bao gồm `0.2.6`) nhiều khả năng dùng chế độ AES không có xác thực toàn vẹn. Cân nhắc nâng cấp dependency `rabe` lên bản mới hơn nếu dùng cho môi trường production.
- Hàm `rabe_free_json` trong `common.rs` hiện dùng `Box::from_raw` để giải phóng con trỏ được tạo bởi `CString::into_raw` — sai layout giải phóng bộ nhớ (cần dùng `CString::from_raw` thay vào đó). Nên vá lại trước khi dùng cho production.
- Không có RUSTSEC advisory chính thức nào được ghi nhận cho `rabe`/`rabe-bn` tại thời điểm viết tài liệu này — nhưng đây là thư viện crypto học thuật, chưa qua audit bảo mật độc lập. Khuyến nghị chạy `cargo audit` định kỳ và tự đánh giá rủi ro trước khi dùng cho dữ liệu nhạy cảm.

---

## Known issues

| Vấn đề | Ảnh hưởng | Trạng thái |
|---|---|---|
| `rabe_free_json` free sai kiểu con trỏ (`Box::from_raw` thay vì `CString::from_raw`) | UB tiềm ẩn, có thể heap corruption tùy allocator | Chưa vá, cần tự sửa |
| `rabe = "0.2.6"` quá cũ so với `0.4.2` hiện tại | Tầng AES nội bộ có thể thiếu AEAD | Nên nâng cấp |
| Yêu cầu nightly + `cargo-expand` | Gây khó khăn khi build trong CI/CD không chuẩn bị sẵn | Cần tài liệu hóa rõ trong pipeline build |
