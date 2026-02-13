# exp1 Changes to chatllm.cpp

This document summarizes all modifications made to the chatllm.cpp fork for the Qwen3-ASR exp1 experiment.

**Upstream**: https://github.com/foldl/chatllm.cpp.git  
**Fork**: https://github.com/vieenrose/chatllm.cpp (branch: `feature/exp1-qwen3-asr`)  
**Base Commit**: origin/master  
**Head Commit**: 52561d9

## Summary

| Category | Files Changed | Lines Added | Lines Removed |
|----------|---------------|-------------|---------------|
| C++ Core | 4 | 42 | 8 |
| Python Bindings | 2 | 37 | 1 |
| **Total** | **7** | **74** | **8** |

## Changed Files

1. `src/layers.h` - Memory safety fix
2. `src/audio_process.cpp` - Platform compatibility fix
3. `src/main.cpp` - ASR support + new APIs
4. `src/models_priv.h` - Memory leak fix (destructor)
5. `src/models.cpp` - Memory leak fix (implementation)
6. `bindings/libchatllm.h` - API declarations
7. `bindings/chatllm.py` - Python bindings
8. `scripts/binding.py` - Path resolution fix

---

## Detailed Changes

### 1. Memory Safety Fix: CoreAttention pos_helper

**File**: `src/layers.h`  
**Commit**: 7475ebc

**Problem**: `pos_helper` was using a pointer to a local variable (`def_pos_helper`) which could be freed prematurely, causing invalid memory access.

**Fix**: Allocate `pos_helper` on the heap instead.

```cpp
// Before
pos_helper(helper ? helper : &def_pos_helper)

// After  
pos_helper(helper ? helper : new BaseTensorPosHelper(max_length))
```

**Impact**: Prevents potential crashes during long-running inference.

---

### 2. Platform Compatibility: popen Mode

**File**: `src/audio_process.cpp`  
**Commit**: ee3537a

**Problem**: `popen()` with mode `"rb"` fails on some Linux systems (works on BSD/macOS).

**Fix**: Use mode `"r"` instead.

```cpp
// Before
popen(oss.str().c_str(), "rb")

// After
popen(oss.str().c_str(), "r")
```

**Impact**: Audio loading via ffmpeg now works on Linux.

---

### 3. New API: chatllm_set_additional_args

**File**: `src/main.cpp`, `bindings/libchatllm.h`, `bindings/chatllm.py`  
**Commit**: 423453f

**Purpose**: Allow dynamic setting of model arguments after initialization (e.g., changing language settings).

**C API** (`libchatllm.h`):
```cpp
DLL_DECL int API_CALL chatllm_set_additional_args(struct chatllm_obj *obj, const char *utf8_str);
```

**C++ Implementation** (`main.cpp`):
```cpp
int chatllm_set_additional_args(struct chatllm_obj *obj, const char *utf8_str)
{
    Chat *chat = reinterpret_cast<Chat *>(obj);
    if (!chat->pipeline || !chat->pipeline->is_loaded()) return -1;
    
    std::string str(utf8_str);
    size_t eq_pos = str.find("=");
    if (eq_pos == std::string::npos) return -2;
    
    std::string key = str.substr(0, eq_pos);
    std::string value = str.substr(eq_pos + 1);
    std::map<std::string, std::string> args;
    args[key] = value;
    chat->pipeline->set_additional_args(args);
    return 0;
}
```

**Python Binding** (`chatllm.py`):
```python
def set_additional_args(self, key: str, value: str) -> int:
    key_value = f"{key}={value}"
    return self._lib.set_additional_args(self._chat, key_value)
```

---

### 4. ASR Model Support

**File**: `src/main.cpp`  
**Commit**: 8e13a50

**Problem**: ASR models were rejected by `chatllm_user_input()` and related functions because only `ModelPurpose::Chat` was allowed.

**Fix**: Allow `ModelPurpose::ASR` in input functions.

```cpp
// Before
if (chat->pipeline->model->get_purpose() != chatllm::ModelPurpose::Chat)

// After
if (chat->pipeline->model->get_purpose() != chatllm::ModelPurpose::Chat && 
    chat->pipeline->model->get_purpose() != chatllm::ModelPurpose::ASR)
```

**Affected Functions**:
- `chatllm_user_input()`
- `chatllm_user_input_multimedia_msg()`
- `chatllm_ai_continue()`

**Impact**: Enables Qwen3-ASR model to accept audio input.

---

### 5. Memory Management: chatllm_destroy

**File**: `src/main.cpp`, `bindings/libchatllm.h`, `bindings/chatllm.py`  
**Commit**: 8e13a50, 69c3f99

**Purpose**: Expose `chatllm_destroy()` to Python for explicit cleanup of model resources.

**C API** (`libchatllm.h`):
```cpp
DLL_DECL int API_CALL chatllm_destroy(struct chatllm_obj *obj);
```

**C++ Implementation** (`main.cpp`):
```cpp
int chatllm_destroy(struct chatllm_obj *obj)
{
    Chat *chat = reinterpret_cast<Chat *>(obj);
    auto it = std::find_if(chat_objects.begin(), chat_objects.end(), 
                           [chat](auto &c) { return c.get() == chat; });
    if (it != chat_objects.end()) {
        chat_objects.erase(it);
        return 0;
    }
    return -1;
}
```

**Python Binding** (`chatllm.py`):
```python
def destroy(self) -> int:
    if hasattr(self, "_chat") and self._chat:
        if self.is_generating: self.abort()
        self._lib.destroy(self._chat)
        self._chat = None
    return 0
```

**Known Issue**: Current implementation only erases from vector but doesn't fully free GGML resources, causing ~50MB/iteration memory leak. See `MEMORY_LEAK_ANALYSIS.md` for details.

---

### 6. Path Resolution Fix

**File**: `scripts/binding.py`  
**Commit**: 8e13a50

**Problem**: `sys.argv[0]` returns different values depending on how Python is invoked, causing path resolution failures.

**Fix**: Use `__file__` instead for reliable path resolution.

```python
# Before
this_dir = os.path.dirname(os.path.abspath(sys.argv[0]))

# After
this_dir = os.path.dirname(os.path.abspath(__file__))
```

---

## Commit History

```
52561d9 fix: delete transformer in BaseModelForConditionalGeneration destructor
69c3f99 Fix chatllm_destroy implementation and lambda capture
d670aab Merge feature/python-asr-support
0a2fd47 Merge feature/set-additional-args-api
5618b83 Merge fix/audio-popen-mode
c17ee89 Merge fix/core-attention-memory-safety
8e13a50 Enable ASR support: allow user_input for ASR models, add chatllm_destroy for memory management, and fix binding path
423453f Add chatllm_set_additional_args to C API and Python bindings
ee3537a Fix audio loading: use mode 'r' instead of 'rb' for popen on Linux
7475ebc Fix memory safety: allocate pos_helper on heap to avoid invalid free in CoreAttention
```

---

## 7. Memory Leak Fix: BaseModelForConditionalGeneration Destructor

**File**: `src/models_priv.h`, `src/models.cpp`  
**Commit**: 52561d9

**Problem**: `HeterogeneousModel *transformer` was allocated with `new` but never deleted. The destructor was `= default` which doesn't delete raw pointers.

**Fix**: Implement proper destructor that deletes `transformer`.

```cpp
// Before (models_priv.h)
virtual ~BaseModelForConditionalGeneration() = default;

// After (models_priv.h)
virtual ~BaseModelForConditionalGeneration();

// Implementation (models.cpp)
BaseModelForConditionalGeneration::~BaseModelForConditionalGeneration()
{
    if (transformer)
    {
        delete transformer;
        transformer = nullptr;
    }
}
```

**Impact**: Reduces memory leak from ~50MB/iteration to ~43MB/iteration (14% improvement). Further investigation needed for remaining leak.

---

## Upstream Merge Status

These changes are **not yet submitted upstream** to foldl/chatllm.cpp. They are specific to the exp1 experiment and may need refinement before upstreaming:

1. **Memory leak partially fixed** - transformer deletion added, but ~43MB/iteration leak remains
2. **ASR-specific logic** - May need abstraction for other model types
3. **set_additional_args** - Generic API, could be useful upstream

---

## Testing

All changes were verified through:
- Docker build from fresh start
- 10-iteration ASR benchmark (0% WER)
- Memory profiling (leak partially fixed: 50MB/iter -> 43MB/iter)
