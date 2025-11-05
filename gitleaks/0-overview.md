Gitleaks là một công cụ mã nguồn mở mạnh mẽ giúp phát hiện và ngăn chặn việc rò rỉ thông tin nhạy cảm như mật khẩu, khóa API, token trong các kho Git. Nó hỗ trợ quét toàn bộ lịch sử commit, các tệp tin, thư mục và cả dữ liệu từ stdin để tìm kiếm các chuỗi có thể là bí mật.([github.com](https://github.com/gitleaks?utm_source=chatgpt.com "Gitleaks - GitHub"))

---

## 🔍 Gitleaks là gì?

Gitleaks là một công cụ SAST (Static Application Security Testing) chuyên dùng để phát hiện và ngăn chặn việc rò rỉ thông tin nhạy cảm như mật khẩu, khóa API, token trong các kho Git. Nó hỗ trợ quét toàn bộ lịch sử commit, các tệp tin, thư mục và cả dữ liệu từ stdin để tìm kiếm các chuỗi có thể là bí mật.

---

## ⚙️ Cách cài đặt Gitleaks

### 1. Cài đặt bằng Homebrew (macOS)

```bash
brew install gitleaks
```

### 2. Sử dụng Docker

```bash
docker pull zricethezav/gitleaks:latest
docker run -v $(pwd):/path zricethezav/gitleaks:latest detect --source=/path
```

### 3. Cài đặt từ mã nguồn (yêu cầu Go)

```bash
git clone https://github.com/gitleaks/gitleaks.git
cd gitleaks
make build
```

---

## 🛠️ Các chế độ hoạt động chính

### 1. `detect` – Quét lịch sử Git để tìm kiếm bí mật

Lệnh này quét toàn bộ lịch sử commit để phát hiện các thông tin nhạy cảm đã được commit trước đó.

```bash
gitleaks detect --source=. --report=gitleaks-report.json
```

### 2. `protect` – Ngăn chặn commit chứa bí mật

Lệnh này kiểm tra các thay đổi hiện tại (chưa commit) để ngăn chặn việc thêm thông tin nhạy cảm vào commit.

```bash
gitleaks protect --staged --verbose
```

---

## 🔁 Tích hợp vào quy trình phát triển

### 1. Pre-commit Hook

Tích hợp Gitleaks vào hook pre-commit để tự động kiểm tra trước mỗi lần commit.

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.16.1
    hooks:
      - id: gitleaks
```

Sau đó, chạy các lệnh sau để cài đặt:

```bash
pre-commit install
pre-commit autoupdate
```

### 2. GitHub Actions

Sử dụng Gitleaks trong GitHub Actions để tự động quét mỗi khi có pull request hoặc push.([infracloud.io](https://www.infracloud.io/blogs/prevent-secret-leaks-in-repositories/?utm_source=chatgpt.com "How to Prevent Secret Leaks in Your Repositories"))

```yaml
name: Gitleaks Scan
on:
  push:
  pull_request:
jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

---

## 🧩 Tùy chỉnh cấu hình

Gitleaks cho phép tùy chỉnh các quy tắc quét thông qua tệp `gitleaks.toml`. Người dùng có thể định nghĩa các biểu thức chính quy (regex) để xác định các mẫu thông tin nhạy cảm cụ thể. Ngoài ra, có thể sử dụng tệp `.gitleaksignore` để bỏ qua các tệp hoặc thư mục không cần quét.([medium.com](https://medium.com/%40ibm_ptc_security/securing-your-repositories-with-gitleaks-and-pre-commit-27691eca478d?utm_source=chatgpt.com "Securing Your Repositories with gitleaks and pre-commit - Medium"))

---

## 📈 Hiệu suất và độ chính xác

Theo một nghiên cứu so sánh các công cụ phát hiện bí mật, Gitleaks đạt độ recall (khả năng phát hiện đúng) lên đến 88%, cao nhất trong số các công cụ được đánh giá. Tuy nhiên, độ precision (độ chính xác) của Gitleaks là 46%, cho thấy vẫn có khả năng xảy ra false positives (phát hiện sai). Do đó, việc tùy chỉnh cấu hình và quy tắc quét là cần thiết để giảm thiểu các cảnh báo không chính xác.

---

## ✅ Kết luận

Gitleaks là một công cụ mạnh mẽ và linh hoạt để phát hiện và ngăn chặn việc rò rỉ thông tin nhạy cảm trong các kho Git. Với khả năng tích hợp vào các quy trình phát triển như pre-commit hook và GitHub Actions, cùng với khả năng tùy chỉnh cao, Gitleaks là một lựa chọn hàng đầu cho các tổ chức và nhà phát triển quan tâm đến bảo mật mã nguồn.([infracloud.io](https://www.infracloud.io/blogs/prevent-secret-leaks-in-repositories/?utm_source=chatgpt.com "How to Prevent Secret Leaks in Your Repositories"), [akashchandwani.medium.com](https://akashchandwani.medium.com/what-is-gitleaks-and-how-to-use-it-a05f2fb5b034?utm_source=chatgpt.com "What is Gitleaks and how to use it? | by Akash Chandwani - Medium"))

---