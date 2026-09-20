# Refactoring Tugas: Memecah Monolith Python

Repository ini berisi hasil refactoring dari `app.py` (monolith) menjadi beberapa module dengan tanggung jawab yang lebih jelas.

## 1. Dependency Map

### 1.1 Sebelum Refactoring (Monolith)

Sebelum refactoring, seluruh kode berada dalam satu file `app.py`. Domain logic, storage, dan CLI saling terhubung tanpa abstraksi.

**Hubungan antar fungsi:**
main → create_user
create_user → load_users
create_user → validate_user
create_user → save_users
list_users → load_users
load_users → users.json
save_users → users.json

**Diagram dependency:**
┌─────────────────────────────────────────────┐
│ app.py │
│ │
│ ┌──────────┐ │
│ │ main │ │
│ └────┬─────┘ │
│ │ │
│ ↓ │
│ ┌──────────────┐ │
│ │ create_user │ │
│ └──┬────┬───┬──┘ │
│ │ │ │ │
│ ↓ ↓ ↓ │
│ ┌──────┐ │ ┌────────┐ │
│ │load_ │ │ │save_ │ │
│ │users │ │ │users │ │
│ └───┬──┘ │ └───┬────┘ │
│ │ │ │ │
│ │ ↓ │ │
│ │ ┌────────────┐ │
│ │ │validate_ │ │
│ │ │user │ │
│ │ └────────────┘ │
│ │ │
│ ↓ │
│ ┌─────────────┐ │
│ │ users.json │ │
│ └─────────────┘ │
│ │
│ ┌───────────┐ │
│ │list_users │──→ load_users() ──→ json │
│ └───────────┘ │
└─────────────────────────────────────────────┘

```
**Simplified Diagram:**
main
 └── create_user
      ├── load_users ──► users.json
      ├── validate_user
      └── save_users ──► users.json

list_users
 └── load_users ──► users.json
```

**Karakteristik:**
- Domain logic (`create_user`, `validate_user`) bergantung langsung ke storage (`load_users`, `save_users`)
- CLI (`main`) bergantung langsung ke domain
- Tidak ada abstraksi/interface
- Sulit di-test karena semua tergantung file JSON

### 1.2 Sesudah Refactoring (Modular)

Setelah refactoring, dependency menjadi satu arah dan bersih.

**Hubungan antar module:**
main.py → UserService
main.py → JsonUserStorage
UserService → UserStorage (interface)
UserService → validate_user
JsonUserStorage → UserStorage (implements)
JsonUserStorage → users.json
validate_user → (tidak depend ke apa pun)

**Diagram dependency:**
┌─────────────────────────────┐
│ main.py │
│ (Presentation) │
│ │
│ main() → UserService │
│ main() → JsonUserStorage │
└──────────────┬──────────────┘
│
↓
┌─────────────────────────────┐
│ services/user_service.py │
│ (Domain Logic) │
│ │
│ UserService(storage) │
│ └→ validate_user │
└──────────────┬──────────────┘
│ depends on
↓
┌─────────────────────────────┐
│ storage/user_storage.py │
│ (Interface / Port) │
│ │
│ class UserStorage(ABC) │
└──────────────┬──────────────┘
↑ implements
│
┌─────────────────────────────┐
│ storage/json_storage.py │
│ (Adapter) │
│ │
│ JsonUserStorage(UserStorage)│
└──────────────┬──────────────┘
↓
users.json

┌─────────────────────────────┐
│ domain/user.py │
│ validate_user() │
│ (tidak depend ke apa pun) │
└─────────────────────────────┘

```
**Simpified Diagram:**
main.py
 ├── UserService
 │    ├── UserStorage (interface)
 │    └── validate_user
 │
 └── JsonUserStorage
      ├── implements UserStorage
      └── reads/writes users.json

validate_user
 └── does not depend on any module
```

### 1.3 Penjelasan Perubahan Dependency

**Perubahan utama:**

| Aspek | Sebelum | Sesudah |
|-------|---------|---------|
| Arah dependency | Bercampur dalam satu file | Satu arah: CLI → Service → Interface ← Adapter |
| Domain ↔ Storage | Langsung ke fungsi konkret | Lewat abstraksi `UserStorage` |
| Abstraksi | Tidak ada | Ada ABC `UserStorage` |
| Jumlah file | 1 file (`app.py`) | 5 module terpisah |
| Testability | Sulit (harus ada file JSON) | Mudah (bisa pakai fake storage) |

**Narasi penjelasan:**

Sebelum refactoring, semua fungsi berada dalam satu file `app.py`. Domain logic (`create_user`, `validate_user`) bergantung langsung pada detail storage (`load_users`, `save_users`). Akibatnya, perubahan pada storage (misalnya dari JSON ke database) akan memaksa perubahan pada domain logic. Selain itu, CLI (`main`) juga bergantung langsung pada domain, sehingga sulit di-test secara terpisah.

Sesudah refactoring, dependency menjadi satu arah dan bersih:
- `main.py` (CLI) hanya bergantung pada `UserService` dan `JsonUserStorage`
- `UserService` (domain logic) bergantung pada abstraksi `UserStorage`, bukan implementasi konkret
- `JsonUserStorage` (adapter) mengimplementasikan `UserStorage` dan menangani detail file JSON
- `validate_user` di `domain/user.py` berdiri sendiri, tidak bergantung ke apa pun

Perubahan ini menerapkan **Dependency Inversion Principle**: domain tidak lagi bergantung pada detail teknis, melainkan pada abstraksi. Akibatnya:
1. Domain logic bisa di-test tanpa menyentuh file JSON
2. Storage bisa diganti (JSON → database) tanpa mengubah domain
3. CLI bisa diganti (terminal → web) tanpa mengubah domain

## 2. Pemisahan Module

### 2.1 Tabel Pemetaan File

| File Lama (`app.py`) | File Baru | Tanggung Jawab |
|----------------------|-----------|----------------|
| `validate_user` | `domain/user.py` | Validasi entity (murni) |
| `load_users` | `storage/json_storage.py` | Baca dari file JSON |
| `save_users` | `storage/json_storage.py` | Tulis ke file JSON |
| *(baru)* | `storage/user_storage.py` | Interface abstrak (kontrak) |
| `create_user` | `services/user_service.py` | Business logic |
| `list_users` | `services/user_service.py` | Business logic |
| `main` | `main.py` | CLI / entry point |

### 2.2 Arah Dependency

```
main.py ──→ services/user_service.py ──→ storage/user_storage.py
                                              ↑
                                              │ implements
                                        storage/json_storage.py ──→ users.json

domain/user.py ← digunakan oleh services
                 (tidak depend ke apa pun)
```

### 2.3 Alasan Pembagian Module

1. **`domain/user.py`** — dipisahkan karena berisi **aturan bisnis murni** (validasi). Tidak boleh tergantung pada storage atau UI.

2. **`storage/user_storage.py`** — dibuat sebagai **interface abstrak** agar domain bisa bergantung pada abstraksi, bukan implementasi konkret (Dependency Inversion Principle).

3. **`storage/json_storage.py`** — dipisahkan karena berisi **detail teknis** (baca/tulis file JSON). Detail teknis harus diisolasi di adapter.

4. **`services/user_service.py`** — dipisahkan karena berisi **orkestrasi business logic** (gabungan validasi + storage). Tidak tahu detail storage apa yang dipakai.

5. **`main.py`** — dipisahkan karena berisi **CLI**. Presentation layer harus bisa diganti tanpa mengubah domain.

### 2.4 Arah Dependency

Dependency mengalir **satu arah**:
- `main.py` → `services` → `storage` (interface)
- `storage/json_storage.py` → mengimplementasikan interface
- `domain/user.py` → berdiri sendiri, tidak depend ke apa pun

Ini menerapkan **Dependency Inversion Principle**: domain tidak bergantung pada detail, tapi pada abstraksi.

---

## 3. Test

### 3.1 Struktur Test

Test dipisahkan sesuai module yang diuji:

| File Test | Yang Diuji | Jumlah Test |
|-----------|-----------|-------------|
| `tests/test_user.py` | `validate_user` (domain) | 3 test |
| `tests/test_user_service.py` | `UserService` dengan `FakeUserStorage` | 3 test |
| `tests/test_json_storage.py` | `JsonUserStorage` (adapter) | 2 test |

### 3.2 Keuntungan Struktur Test Baru

1. **Test Service Tanpa File JSON** — dengan `FakeUserStorage`, test `UserService` tidak perlu menyentuh file JSON sama sekali. Ini bukti nyata manfaat Dependency Inversion.

2. **Test Terisolasi** — setiap test fokus pada satu tanggung jawab.

3. **Test Cepat** — karena tidak ada I/O file, test berjalan dalam **0.12 detik**.

### 3.3 Contoh Fake Storage

```python
class FakeUserStorage(UserStorage):
    def __init__(self):
        self.users = []

    def load_users(self):
        return list(self.users)

    def save_users(self, users):
        self.users = list(users)
```

`FakeUserStorage` **mengimplementasikan interface** `UserStorage`, tapi **tidak menyentuh file system**. Inilah keuntungan utama Dependency Inversion.

### 3.4 Cara Menjalankan Test

```bash
pytest tests/ -v
```

**Hasil:**

```
collected 8 items

tests/test_json_storage.py::test_save_and_load_users PASSED
tests/test_json_storage.py::test_load_nonexistent_file_returns_empty PASSED
tests/test_user.py::test_validate_user PASSED
tests/test_user.py::test_invalid_email PASSED
tests/test_user.py::test_empty_name PASSED
tests/test_user_service.py::test_create_user PASSED
tests/test_user_service.py::test_duplicate_email PASSED
tests/test_user_service.py::test_list_users PASSED

8 passed in 0.12s
```

---

## 4. Manfaat Struktur Akhir

1. **Testability** — Test tanpa file JSON (pakai fake storage)
2. **Flexibility** — Ganti storage (JSON → database) tanpa ubah domain
3. **Reusability** — `UserService` bisa dipakai di CLI, web, atau mobile
4. **Maintainability** — Tanggung jawab tiap file jelas
5. **Scalability** — Mudah tambah fitur/adapter baru

---

## 5. Struktur Repository

```
refactoring-tugas/
├── README.md
├── .gitignore
├── main.py
├── users.json
│
├── domain/
│   ├── __init__.py
│   └── user.py
│
├── services/
│   ├── __init__.py
│   └── user_service.py
│
├── storage/
│   ├── __init__.py
│   ├── user_storage.py
│   └── json_storage.py
│
└── tests/
    ├── __init__.py
    ├── test_user.py
    ├── test_user_service.py
    └── test_json_storage.py
```

