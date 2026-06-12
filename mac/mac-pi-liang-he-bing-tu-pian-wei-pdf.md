# Mac 批量合并图片为 PDF

### 使用场景

将当前目录中的 JPG 图片按照文件名排序，然后合并为一个 PDF。

### 脚本

```python
from PIL import Image
from pathlib import Path

files = sorted(Path(".").glob("*.jpg"))

images = [
    Image.open(file).convert("RGB")
    for file in files
]

images[0].save(
    "合并结果.pdf",
    save_all=True,
    append_images=images[1:]
)
```
