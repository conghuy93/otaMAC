# ESP32 GitHub OTA với MAC Address Filtering

Tính năng OTA (Over-The-Air) cho ESP32 từ GitHub Releases, tự động phân phối firmware theo địa chỉ MAC của thiết bị.

## 📁 Files trong thư mục này

- **ota.h** - Header file với API declarations
- **ota.cc** - Implementation của GitHub OTA
- **GITHUB_OTA_README.md** - Hướng dẫn chi tiết cách sử dụng

## 🚀 Tích hợp vào project

### Bước 1: Copy files vào project
```bash
# Copy vào thư mục main của ESP-IDF project
cp ota.h your_project/main/
cp ota.cc your_project/main/
```

### Bước 2: Thêm vào CMakeLists.txt
```cmake
# main/CMakeLists.txt
idf_component_register(
    SRCS 
        "ota.cc"
        # ... other source files
    INCLUDE_DIRS "."
    REQUIRES 
        esp_http_client
        esp_https_ota
        json
        # ... other components
)
```

### Bước 3: Sử dụng trong code
```cpp
#include "ota.h"

void check_github_update() {
    Ota ota;
    
    // Check firmware từ GitHub repository
    if (ota.CheckGitHubVersion("username/repository") == ESP_OK) {
        if (ota.HasGitHubUpdate()) {
            ESP_LOGI(TAG, "New version: %s", ota.GetGitHubVersion().c_str());
            
            // Bắt đầu update
            bool success = ota.StartGitHubUpgrade([](int progress, size_t speed) {
                ESP_LOGI(TAG, "Progress: %d%%, Speed: %zu KB/s", progress, speed / 1024);
            });
            
            if (success) {
                esp_restart();
            }
        }
    }
}
```

## 📦 Chuẩn bị firmware để upload

### 1. Build firmware
```bash
idf.py build
idf.py merge-bin
```

### 2. Đổi tên file theo MAC address
```bash
# Lấy MAC từ device log hoặc esptool
# VD: MAC = D0:CF:13:19:CA:18

# Đổi tên file (bỏ dấu hai chấm)
cp build/merged-binary.bin merged-binary_D0CF1319CA18.bin
```

### 3. Upload lên GitHub Release
1. Vào repository trên GitHub
2. Tạo Release mới với tag `v1.0.1`
3. Upload file `merged-binary_D0CF1319CA18.bin`
4. Publish release

## 📋 Dependencies

Các component ESP-IDF cần thiết:
- `esp_http_client`
- `esp_https_ota` 
- `esp_app_format`
- `json` (cJSON)
- `esp_partition`

Các file khác cần có trong project:
- `board.h` - Board management
- `system_info.h` - Để lấy MAC address
- `settings.h` - Settings storage (optional)

## 🔧 API Reference

### CheckGitHubVersion()
```cpp
esp_err_t CheckGitHubVersion(const std::string& github_repo);
```
Kiểm tra phiên bản mới từ GitHub Releases API.
- **Tham số**: `github_repo` - Format: "owner/repository"
- **Trả về**: `ESP_OK` nếu thành công

### HasGitHubUpdate()
```cpp
bool HasGitHubUpdate();
```
Kiểm tra có phiên bản mới không (sau khi gọi CheckGitHubVersion).

### GetGitHubVersion()
```cpp
const std::string& GetGitHubVersion() const;
```
Lấy version tag từ GitHub (đã bỏ prefix 'v').

### GetGitHubUrl()
```cpp
const std::string& GetGitHubUrl() const;
```
Lấy URL download firmware từ GitHub.

### StartGitHubUpgrade()
```cpp
bool StartGitHubUpgrade(std::function<void(int progress, size_t speed)> callback);
```
Bắt đầu quá trình OTA update.
- **Callback**: Nhận progress (%) và speed (bytes/s)
- **Trả về**: `true` nếu thành công

## 🎯 Ưu điểm

✅ **Không cần OTA server** - Sử dụng GitHub Releases miễn phí  
✅ **Phân phối theo MAC** - Mỗi device có firmware riêng  
✅ **Version control tốt** - Quản lý bằng Git tags  
✅ **GitHub CDN** - Tốc độ download nhanh  
✅ **Dễ triển khai** - Chỉ cần upload lên GitHub  

## 📝 Ví dụ tên file firmware

Device có MAC `D0:CF:13:19:CA:18` có thể sử dụng:
- `merged-binary_D0CF1319CA18.bin`
- `firmware_D0:CF:13:19:CA:18.bin`
- `xiaozhi_D0CF1319CA18.bin`
- Bất kỳ tên nào chứa MAC address

## 🔍 Cách lấy MAC address

### Từ log khi boot:
```
I (xxx) Application: Device MAC: D0:CF:13:19:CA:18
```

### Dùng esptool:
```bash
esptool.py --port COM31 read_mac
```

### Trong code:
```cpp
#include "system_info.h"
std::string mac = SystemInfo::GetMacAddress();
ESP_LOGI(TAG, "MAC: %s", mac.c_str());
```

## 📖 Chi tiết

Xem file **GITHUB_OTA_README.md** để biết thêm:
- Hướng dẫn chi tiết từng bước
- Troubleshooting
- Ví dụ code hoàn chỉnh
- So sánh với OTA server truyền thống

## 📄 License

Tương tự license của project gốc.

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📞 Support

Nếu có vấn đề, hãy tạo issue trên GitHub repository.
