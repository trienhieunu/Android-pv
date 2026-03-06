# Android Phỏng Vấn (Interview Questions)

Bộ câu hỏi ôn tập Android/Kotlin cho phỏng vấn.

## Cấu trúc Repository

```
/{topic}/
  - {issue}.md
```

- **Folder** theo chủ đề lớn: `livedata`, `viewmodel`, `coroutines`, `jetpack-compose`
- **File** theo vấn đề cụ thể: `config-change.md`, `observer-pattern.md`

## Format File

Mỗi file có:

### Frontmatter (metadata)
```yaml
---
ID: 001
Date: 2026-03-06
Tags: config-change, rotate, observer-duplication
---
```

### Content
- **Câu hỏi** - Câu hỏi phỏng vấn
- **Câu trả lời** - Giải đáp chi tiết (gốc rễ + theo kiểu phỏng vấn)
- **Tình huống ví dụ** - Code minh họa + timeline + giải pháp

## Topics hiện có

| Topic | Issues |
|-------|--------|
| `livedata/` | Config Change / Rotate |

## Topics cần làm

- [x] `livedata/` - LiveData
- [ ] `viewmodel/` - ViewModel
- [ ] `coroutines/` - Coroutines
- [ ] `jetpack-compose/` - Jetpack Compose
- [ ] `dependency-injection/` - Dependency Injection (Hilt/Dagger)
- [ ] `networking/` - Networking (Retrofit, OkHttp)
- [ ] `database/` - Database (Room)
- [ ] `architecture/` - Architecture (MVVM, MVI, Clean Architecture)
- [ ] `testing/` - Testing
- [ ] `performance/` - Performance

Xem chi tiết trong `TOPICS.md`

## Hướng dẫn sử dụng

1. Clone repo: `git clone https://github.com/trienhieunu/Android-pv.git`
2. Lựa chọn topic folder theo chủ đề muốn ôn tập
3. Đọc từng file issue theo thứ tự ID (001, 002, 003...)
4. Thực hành với code trong "Tình huống ví dụ"

## Khi thêm câu hỏi mới

1. **Check topic có chưa?**
   - Có folder chưa? Nếu không → tạo folder mới

2. **Tìm ID tiếp theo**
   - Check các file trong topic folder
   - ID tăng theo: 001, 002, 003...

3. **Tạo file mới**
   - Format: `{issue}.md` (lowercase, dash separator)
   - Frontmatter: ID, Date, Tags
   - Content: Câu hỏi, Câu trả lời, Tình huống ví dụ

4. **Update TOPICS.md**
   - Thêm mapping mới vào table

## Ví dụ

Thêm câu hỏi về ViewModel lifecycle:

1. Folder `viewmodel/` chưa có → tạo
2. ID: 001 (lần đầu)
3. File: `viewmodel/lifecycle.md`
4. Update `TOPICS.md` thêm vào table `viewmodel/`

---

**Mục tiêu:** Tổng hợp kiến thức Android/Kotlin cho phỏng vấn một cách hệ thống.
