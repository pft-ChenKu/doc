## ComfyUIExecutionException

ComfyUI 執行錯誤的例外類別,當 workflow 執行過程中發生錯誤時拋出。

### 屬性

- **message** (`str`): 錯誤訊息描述
- **error_code** (`str`): 錯誤發生的 class path
- **error_type** (`int`): 錯誤類型 (如: `ErrorType.ERROR` 或數字代碼)
- **error_node_class** (`str|None`): 發生錯誤的 node class 名稱 (對應 JSON 中的 `class_type`)
- **error_node_id** (`str|None`): 發生錯誤的 node ID (對應 JSON 中的 node key)
- **error_metadata** (`dict`): 從 workflow JSON 中 node 的 `_meta` 欄位取得的額外資訊

### Workflow JSON 結構

```json
{
  "1": {
    "inputs": { ... },
    "class_type": "NodesFaceDetector",
    "_meta": {
      "title": "targetface"
    }
  }
}
```
輸出範例
```
Error Message: No face detected
Error Code: /workspace/ComfyUI/custom_nodes/ComfyUI_PF_facefusion.faceless.nodes.nodes_face_swap2.NoFaceError
Error Type: 0
Error Node Class: NodesFaceDetector
Error Node ID: 1
Error Metadata: {'title': 'targetface'}
```
錯誤處理範例程式碼：

```python
error_title = e.error_metadata.get('title', '')
if error_title.lower() == "srcface":
    print("來源圖片未偵測到人臉")
elif error_title.lower() == "targetface":
    print("目標圖片未偵測到人臉")
```
