# Danh sách Chủ đề (Topics)

## Cấu trúc Repository

```
/{topic}/
  - {issue}.md
```

Mỗi **topic** là một folder, mỗi **issue** là một file với format:
- Frontmatter metadata (ID, Date, Tags)
- Câu hỏi
- Câu trả lời
- Tình huống ví dụ

## Topic → File Mapping

### livedata/

| ID | File | Issue | Tags |
|----|------|-------|------|
| 001 | config-change.md | LiveData Config Change / Rotate | config-change, rotate, observer-duplication, async-operation, lifecycle |

### viewmodel/
*(chưa có câu hỏi)*

### coroutines/
*(chưa có câu hỏi)*

### jetpack-compose/
*(chưa có câu hỏi)*

---

## Quy tắc đặt tên

### Folder (topic)
- Tiếng Anh, lowercase: `livedata`, `viewmodel`, `coroutines`, `jetpack-compose`
- Dùng `-` cho multi-word: `dependency-injection`

### File (issue)
- Tiếng Anh, lowercase, dash separator: `config-change.md`
- Mô tả issue cụ thể, không quá dài
- Không dùng space, dùng `-` thay vì `_`

### Format file

```markdown
---
ID: 001
Date: 2026-03-06
Tags: config-change, rotate, observer-duplication
---

# {Tiêu đề}

## Câu hỏi
...

## Câu trả lời
...

## Tình huống ví dụ
...
```

---

## Lưu ý

Khi thêm câu hỏi mới:
1. Check topic có chưa (có folder chưa?)
   - Nếu có: tạo file mới trong folder đó (tăng ID tự động)
   - Nếu không: tạo folder mới + file đầu tiên (ID: 001)

2. ID tăng theo từng topic:
   - livedata: 001, 002, 003...
   - viewmodel: 001, 002, 003...

3. Update TOPICS.md để track mapping

---

## Topics cần làm (TODO)

### livedata/
- [x] 001 - Config Change / Rotate
- [ ] 002 - Observer Pattern & Lifecycle
- [ ] 003 - LiveData vs StateFlow

### viewmodel/
- [ ] 001 - ViewModel Lifecycle
- [ ] 002 - ViewModel Scope & Coroutines
- [ ] 003 - SavedStateHandle

### coroutines/
- [ ] 001 - Coroutine Scope (ViewModelScope, LifecycleScope, etc.)
- [ ] 002 - Dispatchers (Main, IO, Default)
- [ ] 003 - Structured Concurrency
- [ ] 004 - Exception Handling

### jetpack-compose/
- [ ] 001 - State & Remember
- [ ] 002 - Recomposition
- [ ] 003 - Side Effects

### dependency-injection/
- [ ] 001 - Hilt Basics
- [ ] 002 - Dagger vs Hilt
- [ ] 003 - Scopes

### networking/
- [ ] 001 - Retrofit Basics
- [ ] 002 - OkHttp Interceptors
- [ ] 003 - Serialization (Moshi, Gson)

### database/
- [ ] 001 - Room Basics
- [ ] 002 - Database Migration
- [ ] 003 - Room vs SQLite

### architecture/
- [ ] 001 - MVVM vs MVI
- [ ] 002 - Clean Architecture
- [ ] 003 - Single Source of Truth

### testing/
- [ ] 001 - Unit Testing
- [ ] 002 - UI Testing (Espresso)
- [ ] 003 - ViewModel Testing

### performance/
- [ ] 001 - Memory Leaks
- [ ] 002 - ANR (Application Not Responding)
- [ ] 003 - App Size Optimization
