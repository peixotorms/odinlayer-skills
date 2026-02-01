---
name: rust-unsafe
description: Use when writing unsafe code, FFI bindings, raw pointers, or C interop in Rust. Covers bindgen, cbindgen, extern "C", #[repr(C)], transmute, UnsafeCell, NonNull, Pin, PhantomData, soundness, undefined behavior, aliasing rules, safety invariants, SAFETY comments, Miri testing, common pitfalls, FFI wrapper patterns, CString handling, opaque types, and safe abstraction design.
---

# Unsafe & FFI

## Safety Rules

| Rule | Guideline |
|------|-----------|
| `unsafe` only for UB risk (not "dangerous") | M-UNSAFE-IMPLIES-UB |
| Must have valid reason: novel abstraction, perf, FFI | M-UNSAFE |
| All code must be sound — no exceptions | M-UNSOUND |
| Must pass Miri, include safety comments | M-UNSAFE |
| Never use `unsafe` to escape borrow checker | general-01 |
| Never use `unsafe` blindly for performance | general-02 |
| No aliasing violations | general-03 |

## Before Writing Unsafe — Checklist

1. **Do you really need unsafe?** Try safe alternatives first: restructure for borrow checker, use `Cell`/`RefCell`/`Mutex`, check for a safe crate
2. **Identify the operation**: pointer deref, `unsafe fn` call, mutable static, unsafe trait impl, union field, FFI
3. **Document invariants** with `// SAFETY:` comments explaining what must hold and why it does
4. **Test with Miri**: `cargo miri test`

## Top 10 Unsafe Pitfalls

| # | Pitfall | Detection | Fix |
|---|---------|-----------|-----|
| 1 | Dangling pointer from local | Miri | Heap-allocate or return value |
| 2 | CString dropped, pointer dangling | Miri | Caller keeps CString alive, or `into_raw` |
| 3 | `Vec::set_len` with uninitialized data | Miri | Use `push`/`resize` instead |
| 4 | Reference to `#[repr(packed)]` field | Miri, UBsan | `read_unaligned` via `addr_of!` |
| 5 | Mutable aliasing through raw pointers | Miri | Use single pointer, sequential access |
| 6 | `transmute` to wrong size | Compile error/Miri | Use `as` conversion |
| 7 | Invalid enum discriminant via transmute | Manual review | Use `match` or `TryFrom` |
| 8 | Panic unwinding across FFI boundary | Testing | Wrap with `catch_unwind` |
| 9 | Double free from `Clone` + `Drop` on raw ptr | ASan | Don't impl Clone, or use refcounting |
| 10 | `mem::forget` prevents destructor (lock leak) | Manual review | Let guard drop naturally |

## Unsafe Rules by Category

| Category | Count | Key Rules |
|----------|-------|-----------|
| General Principles | 3 | No borrow-checker escape, no perf-only unsafe, no aliasing |
| Safety Abstraction | 11 | Panic safety, invariant verification, Send/Sync soundness, SAFETY comments |
| Raw Pointers | 6 | Prefer `NonNull`, use `PhantomData` for variance, check alignment, never cast `*const` to `*mut` |
| Memory Layout | 6 | `#[repr(C)]` for FFI, use `MaybeUninit` (not `mem::uninitialized`), check reentrancy |
| FFI | 18 | No direct `String`, proper CString/CStr, Drop for C ptrs, panic boundaries, portable types |
| Union | 2 | Avoid except for FFI, no cross-lifetime unions |
| I/O Safety | 1 | Raw handle ownership |

```rust
// SAFETY comment examples:
// SAFETY: We checked that index < len above, so this is in bounds.
// SAFETY: The pointer was created from a valid reference and hasn't been invalidated.
// SAFETY: We hold the lock, guaranteeing exclusive access.
// SAFETY: The type is #[repr(C)] and all fields are initialized.
```

## Safe Abstraction Pattern

Wrap unsafe in safe public APIs:

```rust
pub struct CBuffer {
    ptr: NonNull<u8>,
    len: usize,
}

impl CBuffer {
    pub fn new(size: usize) -> Option<Self> {
        let ptr = unsafe { c_alloc(size) };
        NonNull::new(ptr).map(|ptr| Self { ptr, len: size })
    }

    pub fn as_slice(&self) -> &[u8] {
        // SAFETY: ptr is valid for len bytes (from c_alloc contract)
        unsafe { std::slice::from_raw_parts(self.ptr.as_ptr(), self.len) }
    }
}

impl Drop for CBuffer {
    fn drop(&mut self) {
        unsafe { c_free(self.ptr.as_ptr()); }
    }
}
```

Key principles: encapsulate unsafe behind safe API, use `PhantomData` for lifetime tracking, use `Drop` for cleanup, use private fields to maintain invariants.

---

## FFI Patterns

### Key FFI Rules

1. Always use `#[repr(C)]` for types crossing FFI
2. Handle null pointers at the boundary
3. Catch panics before returning to C (`catch_unwind`)
4. Document ownership clearly (who allocates, who frees)
5. Use opaque types for type safety
6. No `String` in FFI — use `CString`/`CStr`
7. Use portable types (`c_int`, `c_char`, etc.)

### FFI Wrapper Pattern

```rust
// Raw C API in private module
mod ffi {
    extern "C" {
        pub fn lib_create(name: *const c_char) -> *mut c_void;
        pub fn lib_destroy(handle: *mut c_void);
    }
}

// Safe public wrapper with RAII
pub struct Library {
    handle: NonNull<c_void>,
}

impl Library {
    pub fn new(name: &str) -> Result<Self, LibraryError> {
        let c_name = CString::new(name).map_err(|_| LibraryError("invalid name".into()))?;
        let handle = unsafe { ffi::lib_create(c_name.as_ptr()) };
        NonNull::new(handle)
            .map(|handle| Self { handle })
            .ok_or_else(|| LibraryError("creation failed".into()))
    }
}

impl Drop for Library {
    fn drop(&mut self) {
        unsafe { ffi::lib_destroy(self.handle.as_ptr()); }
    }
}
```

### Error Handling Across FFI

```rust
// BAD: panic unwinding across FFI is UB
#[no_mangle]
extern "C" fn callback(x: i32) -> i32 {
    if x < 0 { panic!("negative!"); } // UB!
    x * 2
}

// GOOD: catch panics at FFI boundary
#[no_mangle]
extern "C" fn callback(x: i32) -> i32 {
    std::panic::catch_unwind(|| {
        if x < 0 { panic!("negative!"); }
        x * 2
    }).unwrap_or(-1) // error code on panic
}
```

### Opaque Handle Types

```rust
// Prevent mixing up handles with phantom types
#[repr(C)]
pub struct DatabaseHandle {
    _data: [u8; 0],
    _marker: PhantomData<(*mut u8, PhantomPinned)>,
}

pub struct Database { handle: NonNull<DatabaseHandle> }

pub struct Connection<'db> {
    handle: NonNull<ConnectionHandle>,
    _db: PhantomData<&'db Database>, // lifetime ties to Database
}
```

### Callback Registration

```rust
pub struct CallbackGuard<F> {
    _closure: Box<F>,
}

impl<F: FnMut(i32) -> i32 + 'static> CallbackGuard<F> {
    pub fn register(closure: F) -> Self {
        let boxed = Box::new(closure);
        let user_data = Box::into_raw(boxed) as *mut c_void;

        extern "C" fn trampoline<F: FnMut(i32) -> i32>(
            value: c_int, user_data: *mut c_void,
        ) -> c_int {
            catch_unwind(AssertUnwindSafe(|| {
                let closure = unsafe { &mut *(user_data as *mut F) };
                closure(value as i32) as c_int
            })).unwrap_or(-1)
        }

        unsafe { register_callback(trampoline::<F>, user_data); }
        Self { _closure: unsafe { Box::from_raw(user_data as *mut F) } }
    }
}

impl<F> Drop for CallbackGuard<F> {
    fn drop(&mut self) {
        unsafe { unregister_callback(); }
    }
}
```

### C-Compatible Structs

```rust
#[repr(C)]
pub struct Config {
    pub version: c_int,
    pub flags: u32,
    pub name: [c_char; 64],
    pub name_len: usize,
}

// Verify layout at compile time
const _: () = {
    assert!(std::mem::size_of::<Config>() == 80);
    assert!(std::mem::align_of::<Config>() == 8);
};
```
