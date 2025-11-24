# GitHub OTA theo địa chỉ MAC

## Tổng quan
Tính năng này cho phép ESP32 tự động cập nhật firmware từ GitHub Releases dựa trên địa chỉ MAC của thiết bị.

## Cách hoạt động

### 1. Kiểm tra phiên bản mới
```cpp
Ota ota;
esp_err_t err = ota.CheckGitHubVersion("username/repository");
```

### 2. Kiểm tra có bản cập nhật không
```cpp
if (ota.HasGitHubUpdate()) {
    ESP_LOGI(TAG, "New version: %s", ota.GetGitHubVersion().c_str());
    ESP_LOGI(TAG, "Download URL: %s", ota.GetGitHubUrl().c_str());
}
```

### 3. Thực hiện cập nhật
```cpp
bool success = ota.StartGitHubUpgrade([](int progress, size_t speed) {
    ESP_LOGI(TAG, "Progress: %d%%, Speed: %zu B/s", progress, speed);
});

if (success) {
    ESP_LOGI(TAG, "Upgrade successful, rebooting...");
    esp_restart();
}
```

## Chuẩn bị firmware trên GitHub

### Bước 1: Build firmware
```bash
idf.py build
idf.py merge-bin
```

### Bước 2: Đổi tên file theo MAC address
Firmware phải có tên chứa địa chỉ MAC của thiết bị (không phân biệt hoa thường, có thể có hoặc không có dấu hai chấm):

**Định dạng tên file:**
- `merged-binary_AABBCCDDEEFF.bin` (không có dấu hai chấm)
- `firmware_AA:BB:CC:DD:EE:FF.bin` (có dấu hai chấm)
- `xiaozhi_AABBCCDDEEFF.bin`
- Bất kỳ tên nào chứa MAC address

**Ví dụ:**
```bash
# Lấy MAC address từ log
# MAC: D0:CF:13:19:CA:18

# Đổi tên file
cp build/merged-binary.bin merged-binary_D0CF1319CA18.bin
```

### Bước 3: Tạo GitHub Release

1. Đi đến repository trên GitHub
2. Click vào **Releases** > **Draft a new release**
3. Điền thông tin:
   - **Tag version**: `v1.0.1` (phiên bản mới)
   - **Release title**: `Version 1.0.1`
   - **Description**: Mô tả các thay đổi
4. Upload file firmware đã đổi tên (có chứa MAC)
5. Click **Publish release**

### Bước 4: Upload nhiều thiết bị
Nếu có nhiều thiết bị với MAC khác nhau, upload tất cả vào cùng 1 release:
```
Release v1.0.1:
  - merged-binary_D0CF1319CA18.bin  (Device 1)
  - merged-binary_A4CF1234ABCD.bin  (Device 2)
  - merged-binary_B8CF5678EF01.bin  (Device 3)
```

## Ví dụ code hoàn chỉnh

### Trong application.cc:

```cpp
void Application::CheckGitHubUpdate() {
    const char* GITHUB_REPO = "username/repository"; // Thay đổi repository của bạn
    
    auto& board = Board::GetInstance();
    auto display = board.GetDisplay();
    
    ESP_LOGI(TAG, "Checking GitHub for updates...");
    display->SetStatus("Checking updates...");
    
    Ota ota;
    esp_err_t err = ota.CheckGitHubVersion(GITHUB_REPO);
    
    if (err == ESP_ERR_NOT_FOUND) {
        ESP_LOGW(TAG, "No firmware found for this device's MAC address");
        display->SetStatus("No update for this device");
        return;
    }
    
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Failed to check GitHub version: %d", err);
        display->SetStatus("Update check failed");
        return;
    }
    
    if (!ota.HasGitHubUpdate()) {
        ESP_LOGI(TAG, "Firmware is up to date");
        display->SetStatus("Up to date");
        return;
    }
    
    // Có bản cập nhật mới
    ESP_LOGI(TAG, "New version available: %s", ota.GetGitHubVersion().c_str());
    char buffer[128];
    snprintf(buffer, sizeof(buffer), "Updating to v%s...", ota.GetGitHubVersion().c_str());
    display->SetStatus(buffer);
    
    // Bắt đầu cập nhật
    bool success = ota.StartGitHubUpgrade([&](int progress, size_t speed) {
        snprintf(buffer, sizeof(buffer), "Updating: %d%%", progress);
        display->SetStatus(buffer);
        ESP_LOGI(TAG, "Update progress: %d%%, Speed: %zu KB/s", progress, speed / 1024);
    });
    
    if (success) {
        ESP_LOGI(TAG, "Update successful! Rebooting...");
        display->SetStatus("Update done! Rebooting...");
        vTaskDelay(pdMS_TO_TICKS(2000));
        esp_restart();
    } else {
        ESP_LOGE(TAG, "Update failed!");
        display->SetStatus("Update failed!");
    }
}

// Gọi trong Start()
void Application::Start() {
    // ... existing code ...
    
    // Kiểm tra update từ GitHub
    CheckGitHubUpdate();
    
    // ... rest of code ...
}
```

## Kiểm tra MAC address của thiết bị

### Cách 1: Từ log khi boot
```
I (xxx) Application: Device MAC: D0:CF:13:19:CA:18
```

### Cách 2: Dùng esptool
```bash
esptool.py --port COM31 read_mac
```

### Cách 3: Thêm vào code
```cpp
#include "system_info.h"

void print_mac() {
    std::string mac = SystemInfo::GetMacAddress();
    ESP_LOGI(TAG, "Device MAC: %s", mac.c_str());
}
```

## Cấu trúc GitHub Release API Response

```json
{
  "tag_name": "v1.0.1",
  "name": "Version 1.0.1",
  "assets": [
    {
      "name": "merged-binary_D0CF1319CA18.bin",
      "browser_download_url": "https://github.com/user/repo/releases/download/v1.0.1/merged-binary_D0CF1319CA18.bin",
      "size": 8805748
    }
  ]
}
```

## Troubleshooting

### Lỗi: "No firmware found for MAC address"
- Kiểm tra tên file có chứa đúng MAC address không
- MAC có thể có hoặc không có dấu hai chấm `:`
- Không phân biệt hoa thường

### Lỗi: "Failed to get GitHub release info, status code: 403"
- GitHub API rate limit (60 requests/hour cho IP không xác thực)
- Đợi 1 tiếng hoặc sử dụng GitHub token

### Lỗi: "Failed to parse GitHub API response"
- Kiểm tra repository name có đúng không
- Repository phải là public hoặc có token

## Lưu ý bảo mật

1. **Repository phải là PUBLIC** để device có thể download firmware
2. Không cần GitHub token cho public repositories
3. Nếu muốn dùng private repository, cần thêm GitHub token vào code

## So sánh với OTA server thông thường

| Tính năng | OTA Server | GitHub OTA |
|-----------|-----------|------------|
| Cài đặt server | Cần | Không cần |
| Chi phí | Cần server | Miễn phí |
| Phân phối theo MAC | Cần code custom | Có sẵn |
| Version control | Phải tự quản lý | GitHub Releases |
| Rate limit | Không | 60 req/hour |
| Băng thông | Giới hạn server | GitHub CDN |

## API Reference

### CheckGitHubVersion()
```cpp
esp_err_t CheckGitHubVersion(const std::string& github_repo);
```
- Kiểm tra phiên bản mới từ GitHub
- **Tham số:** `github_repo` - format "owner/repository"
- **Trả về:** `ESP_OK` nếu thành công, `ESP_ERR_NOT_FOUND` nếu không tìm thấy firmware cho MAC này

### HasGitHubUpdate()
```cpp
bool HasGitHubUpdate();
```
- Kiểm tra có bản cập nhật mới không
- **Trả về:** `true` nếu có phiên bản mới hơn

### GetGitHubVersion()
```cpp
const std::string& GetGitHubVersion() const;
```
- Lấy version tag từ GitHub (đã loại bỏ prefix 'v')

### GetGitHubUrl()
```cpp
const std::string& GetGitHubUrl() const;
```
- Lấy URL download firmware

### StartGitHubUpgrade()
```cpp
bool StartGitHubUpgrade(std::function<void(int progress, size_t speed)> callback);
```
- Bắt đầu quá trình cập nhật
- **Callback:** Nhận progress (%) và speed (bytes/s)
- **Trả về:** `true` nếu thành công

## Ví dụ thực tế

Xem file `main/application.cc` để biết cách tích hợp vào ứng dụng chính.
