# ComfyUI VRAM 用量異常問題

## 問題
在 ComfyUI 自定義節點中使用 Prestartup Script 進行 monkey patching 時，發現 VRAM 用量異常增加 **10-12 GB**（從 58 GB 增加到 70 GB，增幅約 17-20%）。

## 環境資訊
- **ComfyUI 版本**: v0.3.75
- **Python 版本**: 3.10.0 (conda-forge)
- **PyTorch 版本**: 2.5.1
- **硬體**: NVIDIA H200 NVL (143156 MB VRAM)
- **記憶體模式**: HIGH_VRAM

## 問題分析
在 `prestartup_script.py` 中提前 import 包含 PyTorch 依賴的模組，導致 PyTorch CUDA allocator 在錯誤的時機點初始化，影響記憶體配置策略。

### 問題觸發流程
```
1. ComfyUI 啟動 → 執行 prestartup_script.py
2. prestartup_script.py → import comfyui_classic_cache_white_list
3. comfyui_classic_cache_white_list.py → from comfy_execution.caching import ...
4. comfy_execution.caching → 內部觸發 torch import
5. PyTorch CUDA allocator 提前初始化（在 ComfyUI 正式啟動前）
6. 導致記憶體配置策略異常，多佔用 10-12 GB VRAM
```
### ComfyUI 的檢查機制

```python
if 'torch' in sys.modules:
    logging.warning("WARNING: Potential Error in code: Torch already imported, "
                   "torch should never be imported before this point.")
```

這個檢查會在 ComfyUI 準備好環境後，驗證 torch 是否已被提前載入：
- 如果 `torch` 在 `sys.modules` 中表示已被 import
- 觸發警告，提醒開發者有程式碼違反了 import 順序規則
- 此時 `PYTORCH_CUDA_ALLOC_CONF` 設定已無效，導致 VRAM 使用異常
### 關鍵警告訊息
```log
WARNING: Potential Error in code: Torch already imported, torch should never be imported before this point.
```

此警告出現在 ComfyUI 正式啟動時，表明 PyTorch 已在 prestartup 階段被提前載入。

### ComfyUI 啟動順序說明

理解 ComfyUI 的啟動順序是解決此問題的關鍵：

```
1. Prestartup 階段（prestartup_script.py）
   ├─ 執行時機：ComfyUI 核心初始化之前
   ├─ 目的：預先載入自定義配置、monkey patching
   └─ 此時 import torch → 觸發警告 + VRAM 異常
   
2. ComfyUI 核心初始化
   ├─ 設定 CUDA 環境變數
   ├─ 初始化 comfy.model_management
   ├─ 配置 PyTorch CUDA allocator
   └─ 準備接受 torch import
   
3. Custom Nodes 載入（__init__.py）
   ├─ 執行時機：ComfyUI 已完成初始化
   ├─ 載入自定義節點類別
   └─ 此時 import torch → 安全，不會觸發警告
   
4. ComfyUI 正式啟動
   └─ 開始提供服務
```

**關鍵差異：**
- **Prestartup 階段**：ComfyUI 還未準備好，此時 import torch 會導致 PyTorch 自行初始化 CUDA allocator，與 ComfyUI 的配置衝突
- **Custom Nodes 階段**：ComfyUI 已完成初始化，此時 import torch 是安全的，會使用 ComfyUI 預先配置好的環境

**實際 Log 驗證：**
```log
[PF Prestart] All prestart applied successfully!    # ← Prestartup 完成
Prestartup times: 3.3 seconds

Total VRAM 143156 MB, total RAM 515596 MB           # ← ComfyUI 初始化
pytorch version: 2.5.1
Set vram state to: HIGH_VRAM
Device: cuda:0 NVIDIA H200 NVL : native              # ← CUDA 環境已設定

[ComfyUI_PF_essentials] torch.cuda.current_device() # ← __init__.py 執行（此時安全）
Import times for custom nodes: ...                  # ← Custom nodes 載入
```
## ComfyUI 對 PyTorch Import 時機的控制機制

### 為什麼 Prestartup 階段不能 import torch？

ComfyUI 在 `main.py` 中實施了嚴格的 import 順序控制，確保在載入 PyTorch 之前先設定所有必要的環境變數和 CUDA 配置。

#### 關鍵code位置

**`main.py` (第 130-148 行):**
```python
# 1. 先設定 CUDA 環境變數
if args.cuda_device is not None:
    os.environ['CUDA_VISIBLE_DEVICES'] = str(args.cuda_device)
    os.environ['HIP_VISIBLE_DEVICES'] = str(args.cuda_device)
    os.environ["ASCEND_RT_VISIBLE_DEVICES"] = str(args.cuda_device)

if args.deterministic:
    if 'CUBLAS_WORKSPACE_CONFIG' not in os.environ:
        os.environ['CUBLAS_WORKSPACE_CONFIG'] = ":4096:8"

# 2. 載入 cuda_malloc 設定（設定 PYTORCH_CUDA_ALLOC_CONF）
import cuda_malloc

# 3. 檢查 torch 是否已被提前載入
if 'torch' in sys.modules:
    logging.warning("WARNING: Potential Error in code: Torch already imported, "
                   "torch should never be imported before this point.")

# 4. 此時才正式 import torch 相關模組
import comfy.utils
import execution
import comfy.model_management
```

#### CUDA Memory Allocator 設定機制

**`cuda_malloc.py` (第 85-93 行):**
```python
if args.cuda_malloc and not args.disable_cuda_malloc:
    env_var = os.environ.get('PYTORCH_CUDA_ALLOC_CONF', None)
    if env_var is None:
        env_var = "backend:cudaMallocAsync"
    else:
        env_var += ",backend:cudaMallocAsync"
    
    os.environ['PYTORCH_CUDA_ALLOC_CONF'] = env_var
```

### 為什麼會影響 VRAM 使用？

PyTorch 的 CUDA allocator 會在**第一次 import torch 時**讀取 `PYTORCH_CUDA_ALLOC_CONF` 環境變數來決定記憶體管理策略：

1. **正確時機（ComfyUI 控制）:**
   ```
   設定 PYTORCH_CUDA_ALLOC_CONF=backend:cudaMallocAsync
   → import torch
   → PyTorch 使用 cudaMallocAsync allocator
   → 高效的記憶體管理，VRAM ~58 GB
   ```

2. **錯誤時機（Prestartup 提前 import）:**
   ```
   import torch（在 prestartup 階段）
   → PyTorch 使用預設 allocator（未設定 PYTORCH_CUDA_ALLOC_CONF）
   → 後續 ComfyUI 設定的環境變數無效（已初始化）
   → 低效的記憶體管理，VRAM ~70 GB (+12 GB)
   ```

### PyTorch CUDA Allocator 初始化特性

PyTorch 的 CUDA allocator 有以下特性：

1. **單次初始化**: 第一次 import torch 時初始化，之後無法更改
2. **環境變數依賴**: 只在初始化時讀取 `PYTORCH_CUDA_ALLOC_CONF`
3. **全局影響**: allocator 策略影響整個 Python 進程的 CUDA 記憶體管理

**相關文檔:**
- [PyTorch CUDA Semantics - Memory Management](https://pytorch.org/docs/stable/notes/cuda.html#memory-management)
- [cudaMallocAsync Allocator](https://pytorch.org/docs/stable/notes/cuda.html#cudasync-allocator)
### 為什麼 `__init__.py` 中的 import torch 不會觸發問題？

這是因為執行時機不同：

| 階段 | 執行時機 | import torch 結果 | VRAM 影響 |
|------|---------|------------------|----------|
| **prestartup_script.py** | ComfyUI 初始化**之前** |  **觸發警告**<br>PyTorch 自行初始化 CUDA allocator | +10-12 GB |
| **`__init__.py`** | ComfyUI 初始化**之後** |  **正常運作**<br>使用 ComfyUI 預設的 CUDA 配置 | 正常 |

**結論：**
- Prestartup 階段必須避免任何直接或間接的 torch import
- Custom nodes 的 `__init__.py` 可以安全地 import torch
- 使用 import hook 是在 prestartup 階段進行 monkey patching 的最佳實踐
- 

## 解決方案

### 核心策略：延遲 Import + Import Hook

使用 Python 的 `sys.meta_path` import hook 機制，將 monkey patching 延遲到 ComfyUI 正式啟動並載入目標模組時才執行。

### 實作細節

#### 1. Execution 模組 Patching
```python
class ExecutionPatcher:
    """當 execution 模組被載入時自動進行 monkey patch"""
    def find_module(self, fullname, path=None):
        if fullname == 'execution':
            return self
        return None
    
    def load_module(self, fullname):
        if fullname in sys.modules:
            return sys.modules[fullname]
        
        # 找到並載入 execution.py
        import importlib.util
        for module_path in sys.path:
            exec_path = os.path.join(module_path, 'execution.py')
            if os.path.exists(exec_path):
                spec = importlib.util.spec_from_file_location('execution', exec_path)
                module = importlib.util.module_from_spec(spec)
                sys.modules['execution'] = module
                spec.loader.exec_module(module)
                
                # 立即進行 monkey patch
                module._original_CacheType = module.CacheType
                module._original_CacheSet = module.CacheSet
                module.CacheType = comfyui_classic_cache_white_list.CacheType
                module.CacheSet = comfyui_classic_cache_white_list.CacheSet
                
                return module
        
        raise ImportError(f"No module named '{fullname}'")

# 註冊 hook
sys.meta_path.insert(0, ExecutionPatcher())
```

#### 2. ModelPatcher 模組 Patching
```python
class ModelPatcherPatcher:
    """當 comfy.model_patcher 模組被載入時自動進行 monkey patch"""
    def find_module(self, fullname, path=None):
        if fullname == 'comfy.model_patcher':
            return self
        return None
    
    def load_module(self, fullname):
        if fullname in sys.modules:
            return sys.modules[fullname]
        
        # 移除自己避免無限遞迴
        sys.meta_path.remove(self)
        
        try:
            # 正常載入 comfy.model_patcher
            import importlib
            module = importlib.import_module(fullname)
            
            # 此時才 import comfyui_lora_cache_policy（PyTorch 已正確初始化）
            import comfyui_lora_cache_policy
            
            # 立即進行 monkey patch
            module._original_ModelPatcher = module.ModelPatcher
            module.ModelPatcher = comfyui_lora_cache_policy.ModelPatcher
            module.ChunkedPinMemory = comfyui_lora_cache_policy.ChunkedPinMemory
            
            return module
        except Exception as e:
            sys.meta_path.insert(0, self)
            raise

# 註冊 hook
sys.meta_path.insert(0, ModelPatcherPatcher())
```

#### 3. Cache 模組延遲 Import
`comfyui_classic_cache_white_list.py` 中，將頂層 import 移到 `__init__` 方法內：

**修改前（錯誤）:**
```python
# 頂層 import - 會在 prestartup 階段觸發
from comfy_execution.caching import HierarchicalCache, CacheKeySetInputSignature
```

**修改後（正確）:**
```python
class CacheSet:
    def __init__(self, cache_type=None, cache_args={}):
        # 延遲 import - 只在實際初始化時才執行
        from comfy_execution.caching import (
            HierarchicalCache, LRUCache, RAMPressureCache, NullCache,
            CacheKeySetInputSignature, CacheKeySetID
        )
        from comfyui_classic_caching import HierarchicalCache_WhiteList
        
        # 將類別存為實例變數供後續使用
        self._HierarchicalCache = HierarchicalCache
        self._CacheKeySetInputSignature = CacheKeySetInputSignature
        # ... 其他初始化邏輯
```

#### 4. HierarchicalCache_WhiteList 動態繼承
```python
class HierarchicalCache_WhiteList:
    """延遲繼承的 HierarchicalCache，避免 prestartup 時 import comfy_execution.caching"""
    def __init__(self, key_class, whitelist=None):
        # 在 __init__ 時才 import，避免 prestartup 階段觸發 PyTorch
        from comfy_execution.caching import HierarchicalCache
        
        # 動態設置父類
        self.__class__.__bases__ = (HierarchicalCache,)
        super().__init__(key_class)
        self.whitelist = set(whitelist) if whitelist else set()
```


## 技術重點

### 1. Import Hook 機制
- 利用 Python `sys.meta_path` 攔截模組載入
- 在目標模組被 ComfyUI 首次載入時才執行 patch
- 確保 PyTorch 在正確時機初始化

### 2. 避免循環依賴
- `ModelPatcherPatcher.load_module` 在 import 前先將自己從 `sys.meta_path` 移除
- 避免 `importlib.import_module` 再次觸發 `find_module` 造成無限遞迴

### 3. 延遲初始化模式
- 所有涉及 PyTorch 的 import 都延遲到實際使用時
- Prestartup 階段只註冊 hook，不載入任何重型依賴

### 4. 動態繼承
- 使用 `__class__.__bases__` 動態設置父類
- 在 `__init__` 階段才完成類別繼承關係



### 關鍵原則
1. **絕不在 prestartup 階段 import torch**
2. **使用 import hook 延遲 monkey patch 執行時機**
3. **在 `__init__` 或函數內部進行延遲 import**
4. **避免頂層 import 任何會連鎖觸發 torch 的模組**


## 總結

透過 Import Hook 機制，成功將 monkey patching 延遲到 ComfyUI 正式啟動後執行，避免 PyTorch 提前初始化導致的 VRAM 配置異常，節省 **10-12 GB VRAM**（約 17-20%），且不影響功能正常運作。

這個解決方案展示了在受限環境下（無法修改 ComfyUI 源碼）進行深度系統整合時，如何透過 Python 高級特性（import hook、動態繼承）來解決複雜的初始化順序問題。





### 解決方案的原理

使用 import hook 延遲所有會觸發 torch import 的模組：

```python
# Prestartup 階段：只註冊 hook，不 import 任何東西
sys.meta_path.insert(0, ExecutionPatcher())
sys.meta_path.insert(0, ModelPatcherPatcher())

# ComfyUI 啟動階段：
# 1. main.py 設定 PYTORCH_CUDA_ALLOC_CONF
# 2. main.py import execution → ExecutionPatcher 攔截 → 執行 monkey patch
# 3. 此時才 import comfy_execution.caching → 觸發 torch import
# 4. PyTorch 使用正確的 CUDA allocator 設定
```

這樣確保：
- Prestartup 階段完全不觸發 torch import
- 所有 torch 相關的 import 都在 ComfyUI 設定好環境之後
- CUDA allocator 使用正確的配置（cudaMallocAsync）
- VRAM 使用回到正常水平





