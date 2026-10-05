# Báo cáo Bài 3: Xử lý xung đột phức tạp trong quá trình Rebase

- **Môn học:** Java Fundamental - Session 05
- **Đường dẫn nộp bài trên GitHub:** `homework/session_05/ex3/`
- **Người thực hiện:** Huy-Tuan (sieuka1990@gmail.com)

---

## 1. Mục tiêu bài tập
- **Hiểu sâu bản chất Rebase vs Merge:** Khi Merge, Git giải quyết xung đột 1 lần duy nhất để tạo ra 1 Merge Commit. Khi Rebase, Git "tua lại" (replay) từng commit một lên đầu nhánh đích, do đó xung đột được xử lý từng bước (**step-by-step conflict resolution**).
- **Làm chủ quy trình giải quyết xung đột từng chặng:** Biết cách nhận diện các mốc xung đột, đọc đúng các thẻ đánh dấu (`<<<<<<< HEAD`, `=======`, `>>>>>>> <commit>`), sửa thủ công đảm bảo không làm hỏng code của nhánh chính, đánh dấu giải quyết bằng `git add` và tiếp tục bằng `git rebase --continue`.
- **Tạo ra lịch sử Git thẳng tắp (Linear History):** Đưa toàn bộ các commit của nhánh tính năng lên nối tiếp ngay sau commit mới nhất của nhánh chính, loại bỏ hoàn toàn các merge commit phụ lộn xộn.

---

## 2. Kịch bản & Thiết lập môi trường thực hành

### Bước 1: Khởi tạo commit gốc trên nhánh `main`
Tại nhánh `main`, tạo file `config.json` với cấu hình cơ bản:
```json
{
  "port": 8080,
  "debug": false
}
```
Lệnh commit:
```bash
git add config.json
git commit -m "init config"
```

---

### Bước 2: Nhánh `feature-api` phát triển độc lập (2 commit)
Tạo và chuyển sang nhánh `feature-api`:
```bash
git checkout -b feature-api
```

- **Commit 1:** Đổi port sang 9000:
  ```json
  {
    "port": 9000,
    "debug": false
  }
  ```
  ```bash
  git commit -am "feat: change port"
  ```

- **Commit 2:** Kích hoạt chế độ debug:
  ```json
  {
    "port": 9000,
    "debug": true
  }
  ```
  ```bash
  git commit -am "feat: enable debug"
  ```

---

### Bước 3: Nhánh `main` cập nhật đồng thời từ thành viên khác (2 commit)
Quay trở lại nhánh `main`:
```bash
git checkout main
```

- **Commit 1:** Sửa port trên main thành 8081:
  ```json
  {
    "port": 8081,
    "debug": false
  }
  ```
  ```bash
  git commit -am "update port on main"
  ```

- **Commit 2:** Bổ sung cấu hình môi trường production:
  ```json
  {
    "port": 8081,
    "debug": false,
    "env": "production"
  }
  ```
  ```bash
  git commit -am "add env config"
  ```

---

### 🔍 Đồ thị phân nhánh TRƯỚC KHI REBASE

Kiểm tra bằng lệnh:
```bash
git log --graph --all --oneline
```

**Kết quả:**
```text
* 5991363 (main) add env config
* 4b28ef1 update port on main
| * 56b352d (feature-api) feat: enable debug
| * 1f1417b feat: change port
|/  
* 35ed324 init config
```

> **Nhận định:** Hai nhánh `main` và `feature-api` đã phân kỳ (diverged) từ commit gốc `35ed324`. Cả hai nhánh đều can thiệp vào cùng file `config.json` và cùng các dòng cấu hình.

---

## 3. Quy trình Rebase & Giải quyết xung đột từng chặng (Step-by-Step)

Đứng tại nhánh `feature-api`, tiến hành rebase lên `main`:
```bash
git checkout feature-api
git rebase main
```

---

### 🔴 CHẶNG 1: Xung đột tại commit `feat: change port` (1/2)

Ngay khi lệnh rebase bắt đầu, Git tua lại commit đầu tiên của `feature-api` là `1f1417b feat: change port` đặt lên trên đỉnh của `main` (`5991363`). Do cả hai bên đều sửa dòng `"port"`, xung đột xảy ra ngay lập tức:

```text
Auto-merging config.json
CONFLICT (content): Merge conflict in config.json
error: could not apply 1f1417b... feat: change port
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
```

#### 1. Kiểm tra trạng thái `git status`:
```text
interactive rebase in progress; onto 5991363
Last command done (1 command done):
   pick 1f1417b feat: change port
Next command to do (1 remaining command):
   pick 56b352d feat: enable debug
You are currently rebasing branch 'feature-api' on '5991363'.
Unmerged paths:
	both modified:   config.json
```

#### 2. Dấu vết xung đột trong file `config.json`:
```json
{
<<<<<<< HEAD
  "port": 8081,
  "debug": false,
  "env": "production"
=======
  "port": 9000,
  "debug": false
>>>>>>> 1f1417b (feat: change port)
}
```

- **`HEAD` (nhánh main hiện tại):** Mang giá trị `"port": 8081` và cấu hình mới `"env": "production"`.
- **`1f1417b` (commit đang apply):** Muốn đổi giá trị port thành `"port": 9000`.

#### 3. Xử lý xung đột chặng 1:
- Giữ lại tính năng của nhánh `feature-api`: `"port": 9000`.
- Đồng thời bảo toàn cấu hình mới của `main`: `"env": "production"`.
- Nội dung file `config.json` sau khi sửa:
  ```json
  {
    "port": 9000,
    "debug": false,
    "env": "production"
  }
  ```

#### 4. Đánh dấu đã giải quyết và tiếp tục rebase:
```bash
git add config.json
git rebase --continue
```

---

### 🔴 CHẶNG 2: Xung đột tại commit `feat: enable debug` (2/2)

Sau khi commit 1 được áp dụng thành công, Git tiếp tục replay commit thứ hai: `56b352d feat: enable debug`. Xung đột tiếp tục phát sinh vì commit này vốn được tạo trên ngữ cảnh file cũ (chưa có `"env": "production"`):

```text
Auto-merging config.json
CONFLICT (content): Merge conflict in config.json
error: could not apply 56b352d... feat: enable debug
```

#### 1. Kiểm tra trạng thái `git status`:
```text
interactive rebase in progress; onto 5991363
Last commands done (2 commands done):
   pick 1f1417b feat: change port
   pick 56b352d feat: enable debug
No commands remaining.
You are currently rebasing branch 'feature-api' on '5991363'.
Unmerged paths:
	both modified:   config.json
```

#### 2. Dấu vết xung đột trong file `config.json`:
```json
{
  "port": 9000,
<<<<<<< HEAD
  "debug": false,
  "env": "production"
=======
  "debug": true
>>>>>>> 56b352d (feat: enable debug)
}
```

#### 3. Xử lý xung đột chặng 2:
- Áp dụng thay đổi của commit hiện tại: `"debug": true`.
- Bảo toàn cấu hình của `main`: `"env": "production"`.
- Giữ nguyên port đã giải quyết ở chặng 1: `"port": 9000`.
- Nội dung file `config.json` sau khi sửa hoàn chỉnh:
  ```json
  {
    "port": 9000,
    "debug": true,
    "env": "production"
  }
  ```

#### 4. Đánh dấu giải quyết và hoàn tất Rebase:
```bash
git add config.json
git rebase --continue
```

**Thông báo hoàn tất từ Git:**
```text
[detached HEAD 1831f21] feat: enable debug
 1 file changed, 1 insertion(+), 1 deletion(-)
Successfully rebased and updated refs/heads/feature-api.
```

---

## 4. Kiểm tra kết quả sau cùng (Verification)

### 4.1. Kiểm tra trạng thái làm việc sạch sẽ
```bash
git status
```
**Kết quả:**
```text
On branch feature-api
nothing to commit, working tree clean
```

---

### 4.2. Kiểm tra đồ thị lịch sử Git (`git log --graph --oneline`)
```bash
git log --graph --oneline
```

**Kết quả màn hình Terminal:**
```text
* 1831f21 (HEAD -> feature-api) feat: enable debug
* 5e15c4e feat: change port
* 5991363 (main) add env config
* 4b28ef1 update port on main
* 35ed324 init config
```

> **ĐÁNH GIÁ KẾT QUẢ:**
> - Lịch sử Git **thẳng tắp 1 đường thẳng**, hoàn toàn không có commit gộp phụ (`Merge branch...`).
> - Hai commit của `feature-api` (`5e15c4e` và `1831f21`) đã được đặt tiếp nối ngay trên đầu commit mới nhất của `main` (`5991363`).
> - Lưu ý rằng mã hash của các commit trên `feature-api` đã được Git cấp mới (do commit base đã thay đổi).

---

### 4.3. Kiểm tra nội dung hoàn thiện của `config.json`
```json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```
Mọi giá trị cấu hình đều chuẩn xác: Port 9000 (từ feature), Debug true (từ feature), Env production (từ main). Không làm mất bất kỳ mã nguồn nào của nhánh chính.

---

## 5. So sánh chuyên sâu: Xung đột trong Rebase vs Merge

| Tiêu chí so sánh | Git Merge | Git Rebase |
| :--- | :--- | :--- |
| **Cơ chế hoạt động** | Tạo một commit gộp 3 vùng (3-way merge commit) kết nối hai nhánh lại với nhau. | Tua lại (replay) từng commit của nhánh hiện tại đặt lên đỉnh của nhánh đích. |
| **Cách phát sinh xung đột** | Xung đột phát sinh **1 lần duy nhất** giữa trạng thái cuối cùng của 2 nhánh. | Xung đột phát sinh **từng bước (step-by-step)** tại từng commit được replay. |
| **Đồ thị lịch sử (History Graph)** | Phân nhánh và hợp nhánh hình mạng nhện (phi tuyến tính). | **Thẳng hàng (Linear History)**, sạch sẽ, đọc từ trên xuống rất trực quan. |
| **Commit ID (Hash)** | Giữ nguyên toàn bộ mã hash của các commit cũ, chỉ sinh thêm 1 commit merge mới. | **Viết lại lịch sử**: Tạo ra các commit mới với hash mới thay thế commit cũ. |
| **Khi nào nên dùng?** | Khi gộp nhánh tính năng vào nhánh chính chung của team (giữ lại ngữ cảnh nhánh). | Khi muốn cập nhật code mới nhất từ `main` về nhánh cá nhân trước khi mở Pull Request. |

---

## 6. Hướng dẫn chụp ảnh minh chứng trên máy cá nhân để nộp bài

Để chụp ảnh thực tế nộp giáo viên:

1. **Chuẩn bị môi trường:** 
   Trong thư mục `Bai03`, đã có sẵn nhánh `demo-conflict` lưu lại trạng thái trước khi rebase.
2. **Thao tác:**
   - Mở Terminal tại `Bai03`.
   - Gõ: `git checkout demo-conflict` rồi chạy: `git rebase main`.
   - Khi terminal báo `CONFLICT ...`: Nhấn **`Windows + Shift + S`** chụp ảnh xung đột chặng 1 (`hinh1_conflict_step1.png`).
   - Sửa file `config.json`, gõ `git add config.json` rồi `git rebase --continue`.
   - Khi báo tiếp xung đột chặng 2: Nhấn **`Windows + Shift + S`** chụp ảnh xung đột chặng 2 (`hinh2_conflict_step2.png`).
   - Sửa xong file, `git add config.json` và `git rebase --continue`.
   - Chạy lệnh kiểm tra sau cùng:
     ```bash
     git log --graph --oneline
     ```
   - Nhấn **`Windows + Shift + S`** chụp lại đồ thị commit thẳng tắp lưu thành `hinh3_log_graph.png`.
3. **Chèn ảnh vào báo cáo:** Sử dụng cú pháp `![Mô tả ảnh](duong_dan_anh.png)`.

---
*Báo cáo hoàn thành ngày 05/10/2026.*
