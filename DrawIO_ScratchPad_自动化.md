# DrawIO(ScratchPad)自动化

[toc]

可以走“本地 CLI 处理 + 我只处理结构”的路线，而且这才是正解。你担心的点也成立：**整包 base64 往聊天里来回扔，必然 token 爆炸**。解决方法是：**把大块 base64 永远留在你本地，只让 AI 看“骨架”**。





## 核心思路：本地“去肥” + AI 轻量处理 + 本地回填

目标只有一个：**AI 不碰 base64，最终输出必须是可直接导入 drawio 的项目文件（mxfile）**。

### 资产划分（先想清楚这三样）

1. **组件库**（mxlibrary / Scratchpad）：只负责“有哪些组件”
2. **规范模板**（可选）：线型、颜色、标签规则
3. **项目图纸**（mxfile）：最终交付物

### 流程（完整闭环）

#### 1. 本地脱敏 / 瘦身（strip）

对以下所有文件执行同一规则：

- 抽出 `image=data:image/png,...`
- 存入缓存：`img-cache.json`（hash → base64）
- 原位置替换为：`image=@IMG:<hash>`

涉及文件：

- `components.xml` → `components.slim.xml`
- `styles.xml` → `styles.slim.xml`（如有）
- `project.base.xml` → `project.base.slim.xml`

**强制校验**
 每个 slim 文件都必须立刻 inject 回去并能导入 drawio，否则停止。

#### 2. AI 只处理瘦身后的 XML

AI 的输入：

- `components.slim.xml`
- `styles.slim.xml`（如有）
- `project.base.slim.xml`

AI 的输出：

- **项目图纸轻量版**：`project.out.slim.drawio.xml`

AI 只做这些事：

- 选组件
- 画节点 / 连线 / 标签
- 调整布局
   **不包含任何 base64**

#### 3. 本地回填（inject）

对 AI 输出的项目图纸执行回填：

- 输入：`project.out.slim.drawio.xml` + `img-cache.json`
- 输出：`project.out.full.drawio.xml`

导入 diagrams.net 即用。



### 关键结论（必须写清）

- **组件库只 strip 一次**，后续复用
- **每次 inject 的对象都是项目图纸（mxfile）**
- AI 永远不接触 base64
- 是否成功的唯一标准：**inject 后能否导入 drawio**



**血泪教训：** :cry:  :cry::cry: 

1. 使用Py脚本而不要使用powershell脚本（否则AI会各种输出不兼容脚本如本地ps是5.1，她输出6的脚本，需要一次次矫正）。另外py脚本在Mac笔记本下也应该可以用。
2. 每一步都确保可以还原。如第一步Scratchpad使用hash瘦身之后Scratchpad.slim.xml，AI对报文内容进行encode，结果在decode过程遇到了天大的困难，整整刷了一下午没解决。因此：为了确保每一步正确，每个步骤要求使用脚本复原新的xml， 再导入看看



## 给你一套可直接用的 PowerShell 脚本

### 1) 从 Scratchpad.xml 抽 base64 → 生成 cache + slim

新建文件：

```powershell
notepad .\drawio_extract_slim.py
```

粘贴保存：

```powershell
import json
import re
import html
import hashlib
from pathlib import Path

INPUT_XML = "Scratchpad.xml"
OUT_SLIM_XML = "Scratchpad.slim.xml"
OUT_CACHE_JSON = "img-cache.json"

# 匹配 style 里 image=data:image/...;base64,XXXX
IMG_RE = re.compile(
    r'image=data:image/(?P<fmt>png|jpeg|jpg|svg\+xml);base64,(?P<b64>[^";]+)',
    re.IGNORECASE
)

LIB_RE = re.compile(r"<mxlibrary>(?P<inner>[\s\S]*?)</mxlibrary>", re.IGNORECASE)

def sha1_hex(s: str) -> str:
    return hashlib.sha1(s.encode("utf-8")).hexdigest()

def load_text(path: str) -> str:
    return Path(path).read_text(encoding="utf-8-sig")  # 自动吃 BOM

def save_text(path: str, text: str) -> None:
    Path(path).write_text(text, encoding="utf-8", newline="\n")

def main():
    raw = load_text(INPUT_XML)
    m = LIB_RE.search(raw)
    if not m:
        raise SystemExit("输入文件不是 mxlibrary 格式：找不到 <mxlibrary>...</mxlibrary>")

    inner = m.group("inner").strip()

    # Scratchpad.xml 的 inner 本质是 JSON（通常是 [ {...}, {...} ]）
    # 但里面 xml 字段是 &lt;...&gt; 这种 HTML 实体
    try:
        items = json.loads(inner)
    except json.JSONDecodeError:
        # 少数情况下 inner 可能被额外 escape 过
        items = json.loads(html.unescape(inner))

    cache = {}  # hash -> { "base64": "...", "format": "png" }
    replaced = 0
    found = 0

    for it in items:
        if not isinstance(it, dict):
            continue
        if "xml" not in it or not isinstance(it["xml"], str):
            continue

        # 先把 &lt; &gt; &quot; 反成真正的 < > "
        decoded = html.unescape(it["xml"])

        def _sub(match: re.Match) -> str:
            nonlocal replaced, found
            found += 1
            fmt = match.group("fmt").lower()
            if fmt == "jpg":
                fmt = "jpeg"
            b64 = match.group("b64")
            h = sha1_hex(f"{fmt}:{b64}")  # fmt + b64 一起 hash，防止同 b64 不同格式
            if h not in cache:
                cache[h] = {"base64": b64, "format": fmt}
            replaced += 1
            return f"image=@IMG:{h}"

        new_decoded = IMG_RE.sub(_sub, decoded)

        # 再把 < > " 转回 &lt; &gt; &quot;（保持与原 Scratchpad.xml 一致的写法）
        it["xml"] = html.escape(new_decoded, quote=True)

    # 输出 slim：外层 <mxlibrary> 里面直接放 JSON（不要再额外 xml.escape 一次，否则你会得到 &amp;lt;）
    out_inner = json.dumps(items, ensure_ascii=False, indent=2)
    out_xml = f"<mxlibrary>{out_inner}</mxlibrary>\n"
    save_text(OUT_SLIM_XML, out_xml)

    Path(OUT_CACHE_JSON).write_text(
        json.dumps(cache, ensure_ascii=False, indent=2),
        encoding="utf-8",
        newline="\n"
    )

    print("OK")
    print(f"Items              : {len(items)}")
    print(f"Found images       : {found}")
    print(f"Replaced images    : {replaced}")
    print(f"Unique images      : {len(cache)}")
    print(f"Out slim xml       : {OUT_SLIM_XML}")
    print(f"Out cache json     : {OUT_CACHE_JSON}")

if __name__ == "__main__":
    main()

```

用法：

```powershell
Set-ExecutionPolicy -Scope Process Bypass
.\drawio-strip-library.ps1 -InputXml ".\Scratchpad.xml" -OutSlimXml ".\Scratchpad.slim.xml" -OutCacheJson ".\img-cache.json"
```

跑完后，正常应该看到:

- `Scratchpad.slim.xml` 体积明显下降
- `img-cache.json` 体积明显上升

再快速验证一下 slim 里有没有占位符：

```powershell
Select-String -Path .\Scratchpad.slim.xml -Pattern "@IMG:" -List
```

能搜到就代表替换成功。





### 2): 把“组件库”变成可调用清单 

新建脚本：

```powershell
notepad .\drawio-manifest.ps1
```

粘贴保存：

```powershell
param(
  [Parameter(Mandatory=$true)][string]$SlimXml,
  [Parameter(Mandatory=$true)][string]$OutManifestJson
)

$raw = Get-Content -Raw -Encoding UTF8 $SlimXml
$raw = $raw.Trim() -replace '^\s*<mxlibrary>\s*','' -replace '\s*</mxlibrary>\s*$',''

$items = $raw | ConvertFrom-Json

$manifest = @()

foreach ($it in $items) {
  $title = $it.title
  $w = $it.w
  $h = $it.h
  $xmlEsc = $it.xml

  # 这里的 xml 字符串常见是 \u0026lt; 这种（JSON 里再套一层转义），我们先把它还原
  # 方式：先把 \u0026lt; 这种转回 &lt;，再 HtmlDecode 得到真实 <mxGraphModel...>
  $step1 = $xmlEsc -replace '\\u0026','&'
  $decoded = [System.Net.WebUtility]::HtmlDecode($step1)

  # 抓 @IMG hash
  $hash = $null
  if ($decoded -match 'image=@IMG:(?<hash>[a-f0-9]{40})') {
    $hash = $Matches['hash']
  }

  # 抓 value（可能为空）
  $value = $null
  if ($decoded -match 'value="(?<v>[^"]*)"') {
    $value = $Matches['v']
  }

  $manifest += [pscustomobject]@{
    title = $title
    value = $value
    w = $w
    h = $h
    img_hash = $hash
  }
}

$manifest | ConvertTo-Json -Depth 5 | Set-Content -Path $OutManifestJson -Encoding UTF8
Write-Host "OK -> $OutManifestJson"
Write-Host ("Items: {0}" -f $manifest.Count)
```

运行：

```powershell
.\drawio-manifest.ps1 -SlimXml ".\Scratchpad.slim.xml" -OutManifestJson ".\manifest.json"
```

你会得到一个 `manifest.json`，里面每个组件一条记录，核心字段是：

- `title`（你看到的 IFP、NDP100、NDP600、PS610…）
- `w/h`（原始尺寸）
- `img_hash`（对应本地图片 key）



**下一步你怎么跟我对接（不爆 token）**

1. **把 `manifest.json` 发我**（这个很小）
2. 你再告诉我你想把哪些 title 归类成哪几类，比如：
   - Devices：NDP100、NDP600、LCS810…
   - Audio：PS610、EKA-415、Audio Mixer…
   - Cameras：CV810、AI Tracking…
   - Templates：Recording_2Cam_2Mic（你右下角那种）

我就能基于 manifest 给你出第一版“组件调用规范”，比如：

- `ADD NDP600 at (x,y)`
- `ADD Recording_2Cam_2Mic template`
- `CONNECT NDP600 -> IFP by HDMI`



**Note:**

你之前上传给我的某些文件在这边的检索环境里会“过期”，所以**别指望我能自动回读你本地目录**。

你只要把 `manifest.json` 或 `Scratchpad.slim.xml` 重新上传到聊天里，我就能继续分析。



### 3) Inject：把占位符回填回 base64

新建：

```powershell
notepad .\drawio-inject-library.ps1
```

粘贴保存：

```powershell
param(
  [Parameter(Mandatory=$true)][string]$SlimXml,
  [Parameter(Mandatory=$true)][string]$CacheJson,
  [Parameter(Mandatory=$true)][string]$OutFullXml
)

$slim = Get-Content -Raw -Encoding UTF8 $SlimXml
$cache = Get-Content -Raw -Encoding UTF8 $CacheJson | ConvertFrom-Json

$rawJson = $slim.Trim() -replace '^\s*<mxlibrary>\s*','' -replace '\s*</mxlibrary>\s*$',''
$items = $rawJson | ConvertFrom-Json

$pattern = 'image=@IMG:(?<hash>[a-f0-9]{40})'

for ($i=0; $i -lt $items.Count; $i++) {
  $x = $items[$i].xml
  if ([string]::IsNullOrEmpty($x)) { continue }

  $decoded = [System.Net.WebUtility]::HtmlDecode($x)

  $fullDecoded = [regex]::Replace($decoded, $pattern, {
    param($m)
    $hash = $m.Groups['hash'].Value
    if (-not $cache.PSObject.Properties.Name.Contains($hash)) {
      throw "Missing hash in cache: $hash"
    }
    $fmt = $cache.$hash.format
    $b64 = $cache.$hash.base64
    return "image=data:image/$fmt,$b64"
  })

  $items[$i].xml = [System.Net.WebUtility]::HtmlEncode($fullDecoded)
}

$outJson = $items | ConvertTo-Json -Depth 50
$final = "<mxlibrary>`n$outJson`n</mxlibrary>"
Set-Content -Path $OutFullXml -Value $final -Encoding UTF8

Write-Host "OK -> $OutFullXml"
```

用法：

```powershell
.\drawio-inject-library.ps1 -SlimXml ".\Scratchpad.slim.xml" -CacheJson ".\img-cache.json" -OutFullXml ".\Scratchpad.full.xml"
```













# Reference

1. https://chatgpt.com/g/g-p-6906e898fea48191a456216ddde382ae-qnex-shou-qian/c/69380294-91f0-8323-8ee5-58301c08f461



## DrawIO报文结构解析

```json
<mxlibrary>[
  {
    "xml": "&lt;mxGraphModel&gt;&lt;root&gt;&lt;mxCell id=\"0\"/&gt;&lt;mxCell id=\"1\" parent=\"0\"/&gt;&lt;mxCell id=\"2\" parent=\"1\" style=\"shape=image;verticalLabelPosition=bottom;labelBackgroundColor=default;verticalAlign=top;aspect=fixed;imageAspect=0;image=data:image/png,iVBORw0KGgoAAAANSUhEUgAAAeUAAAICCAYAAADxmdXFAAAAAXNSR0IArs4c6QAAIABJREFUeAHsfQdYFVf+9vffze5mN5vNbgpye6/ABaR3BKQ3C6CIBRUF7BXEhhpFxYJdo4C9K1FURBALVlCxgQVREEQUEUWxAvd73nGGJYZkNy5gyfF55rk4d+6U95z5vefX/9//I/8I【这里是一串非常非常长的base64】tVo/LCwsjMA6R8wp93or/UgEiAARIAJEgAg8DwJMnLE041V0cmOnMzMawXNUO34e0OmYRIAIEAEiQASIABEgAkSACBABIkAEiAARIAJEgAgQASJABIgAESACRIAIEAEiQAT+DwT+CwnxxymcWuvoAAAAAElFTkSuQmCC;fontStyle=1\" value=\"NDP100\" vertex=\"1\"&gt;&lt;mxGeometry height=\"140\" width=\"132.1\" x=\"4.547473508864641e-13\" as=\"geometry\"/&gt;&lt;/mxCell&gt;&lt;/root&gt;&lt;/mxGraphModel&gt;",
    "w": 132.1,
    "h": 140,
    "aspect": "fixed",
    "title": "NDP100"
  },
  {
    "xml": "&lt;mxGraphModel&gt;&lt;root&gt;&lt;mxCell id=\"0\"/&gt;&lt;mxCell id=\"1\" parent=\"0\"/&gt;&lt;mxCell id=\"2\" parent=\"1\" style=\"shape=image;verticalLabelPosition=bottom;labelBackgroundColor=default;verticalAlign=top;aspect=fixed;imageAspect=0;image=data:image/png,iVBORw0KGgoAAAANSUhEUgAAAf4AAAHBCAYAAACIQ9ldAAAAAXNSR0IArs4c6QAAIABJREFUeAHsfQe4HUX5/tzdnZ0+s+2Ue84t6QkkEMSGIN0CIiAgSBGxYcSCiAIqIBZE9CcigqAUBWmCKCKCICBVmiK9CYL0FpIQ0tv9n/dkD8/h/i+QkFuSsPs8N6fvzr67mfnK+70fIcVWIFAgUCBQIFAgUCBQIFAgUCB【这里是一串非常非常长的base64】ASYABNgAkyACTABJsAEmAATYAJMgAkwASbABJgAE2ACTIAJMAEmwASYABNgAkyACTABJsAEho3A/wAdGIRXZo9AjQAAAABJRU5ErkJggg==;fontStyle=1\" value=\"NDP600\" vertex=\"1\"&gt;&lt;mxGeometry height=\"129\" width=\"146.53\" x=\"-5.684341886080801e-13\" as=\"geometry\"/&gt;&lt;/mxCell&gt;&lt;/root&gt;&lt;/mxGraphModel&gt;",
    "w": 146.53,
    "h": 129,
    "aspect": "fixed",
    "title": "NDP600"
  }
]</mxlibrary>
```

base64 不在外层 XML 里直接可读，而是藏在 `<mxlibrary>` 里面的 JSON 字段 `xml`（而且还被 HTML escape 过）。

1. **外层：XML 包了一段 JSON**

   ```xml
   <mxlibrary>[
     { "xml": "...", "w":..., "h":..., ... },
     { "xml": "...", "w":..., "h":..., ... }
   ]</mxlibrary>
   ```

   `<mxlibrary>` 的文本内容本质上是 **JSON array**

2. **中层：JSON 里的 `xml` 字段是“HTML 转义后的 XML 字符串”**

   ```xml
   "xml": "<mxGraphModel>...</mxGraphModel>"
   ```

   这里的 `<` `>` `"` 都是 **HTML escape**，你必须先 `html.unescape()` 才能变回真正的 `<mxGraphModel>...`

3. **内层：真正的 mxGraphModel 里，图片在 `mxCell` 的 style 属性里**

   片段（反转义后）类似：

   ```xml
   <mxCell
     style="shape=image;...;image=data:image/png,iVBORw0K...QmCC;fontStyle=1"
     ...
   />
   ```

   **base64 就在 `style` 里 `image=data:image/png,` 之后，到下一个 `;` 之前结束。**

   > 注意：你这里不是 `data:image/png;base64,`，而是 `data:image/png,`。
   >  对 drawio 来说，这一段逗号后面仍然是 base64（它省略了 `;base64` 这段）。

4. **如何“精准截取”那段很长的 base64（避免误伤）**

   在 `style` 字符串里做定位：

   - 起点：找到子串 `image=data:image/png,`
   - 终点：从起点往后找 **下一个分号 `;`**
   - 取中间那段就是 base64 payload（不包含逗号、不包含分号）

   伪逻辑：

   - `start = style.find("image=data:image/png,")`
   - `b64_start = start + len("image=data:image/png,")`
   - `b64_end = style.find(";", b64_start)`
     - 如果找不到 `;`（极少数情况 image 在末尾），那就到字符串结尾

5. **hash 应该怎么做才“跨机器稳定”**

   强烈建议用这条规则（最稳、不会被换行/空格影响）：

   1. base64 字符串 `b64`：先去掉任何空白（一般不会有，但保险）
   2. `raw = base64.b64decode(b64)`
   3. `hash = sha256(raw).hexdigest()`

   为什么不直接 hash base64 文本？

   - base64 文本有可能在不同导出/格式化时出现换行或空白差异（虽然 drawio 通常不会），hash 原始 bytes 更稳。

   替换形态：

   - 原：`image=data:image/png,<HUGE_B64>;`
   - 目标：`image=@IMG:<hash>;`

   也就是说保留 `image=` 这个键，只替换 value，**并且保留 style 里其他 `;key=value` 不动**。

6. 最容易踩坑的点（之前刷一下午那种）

   1. **必须走“反转义 → 修改 → 再转义”这条链**
       因为你修改的是 `xml` 字段里的真实 XML，但它存储时是 `<` 这种形式。
       正确流程是：

   - 读取 `<mxlibrary>` 内容 -> `json.loads()` 得到对象
   - 对每个 item 的 `item["xml"]`：`html.unescape()` 得到真实 xml 字符串
   - 在真实 xml 字符串里找 `style="..."` 并修改 style
   - 再 `html.escape(真实xml, quote=True)` 放回 `item["xml"]`
   - 最后 `json.dumps()` 写回 `<mxlibrary>...</mxlibrary>`

   1. **不要用“XML pretty print”**
       任何自动格式化都可能改变 drawio 对 library 的兼容性（尤其是属性顺序/实体编码）。我们要做的是**最小修改**。



## 数据分类



### 一、总体分层规则（先看这个）

你现在的售前库，**只分三层就够了**：

1. **Components（单设备 / 单图标）**
2. **Templates（组合方案 / 拓扑模块）**
3. **Solutions（完整方案示例，可选）**

⚠️ 注意：

- **AI 自动生成主要用 1 + 2**
- 3 更多是给售前 PPT / 方案展示，不一定参与自动拼图

### 二、一级分类：Components（单设备）

```css
CMP_QNX_*        核心系统
CMP_VID_*        视频 / 信号
CMP_IFC_*        接口 / 转换
CMP_AUD_*        音频
CMP_NET_*        网络
CMP_FAC_*        环境 / 设施
```



#### 命名统一规则（强制）

```css
[Domain]_[Category]_[Model or Name]_[Optional Role]
```

- 全大写
- 下划线 `_`
- 不用空格
- 不用中文



#### 2.1 Q-NEX 核心三大套（重点）

##### A. Podium / Control（讲台 & 控制）

| 类别     | 命名示例            | 说明        |
| -------- | ------------------- | ----------- |
| 数字讲台 | `QNX_PODIUM_NDP100` | 小讲台      |
| 数字讲台 | `QNX_PODIUM_NDP600` | 大讲台      |
| 控制盒   | `QNX_CTRL_CBX`      | 扩展控制    |
| 控制盒   | `QNX_CTRL_IO_IFP`   | IFP IO 控制 |

##### B. Audio（音频系统）

| 类别   | 命名示例                 | 说明     |
| ------ | ------------------------ | -------- |
| 功放   | `QNX_AUDIO_PS610`        | 内置功放 |
| 吸顶麦 | `QNX_AUDIO_MIC_S210`     | 吸顶麦   |
| 音箱   | `QNX_AUDIO_SPK_PS410`    | 被动音箱 |
| 音箱组 | `QNX_AUDIO_SPK_PS410_X4` | 4 只     |

##### C. Lecture Capture / Recording（录播）

| 类别     | 命名示例                   | 说明 |
| -------- | -------------------------- | ---- |
| 录播主机 | `QNX_LCS_LCS810`           | 录播 |
| 教师机位 | `QNX_CAM_CV810_G2_TEACHER` | 教师 |
| 学生机位 | `QNX_CAM_CV810_G2_STUDENT` | 学生 |



#### 2.2 显示与视频（Display / Video）

| 类别     | 命名示例            | 说明         |
| -------- | ------------------- | ------------ |
| 交互屏   | `DSP_IFP_75IN`      | IFP          |
| 广告机   | `DSP_SIGNAGE_55IN`  | Signage      |
| 分配器   | `VID_HDMI_SPLITTER` | HDMI Split   |
| 文档展台 | `VID_DOC_CAM_E4521` | Document Cam |



#### 2.3 网络 / IT

| 类别   | 命名示例         | 说明   |
| ------ | ---------------- | ------ |
| 交换机 | `NET_SWITCH`     | LAN    |
| 路由器 | `NET_ROUTER`     | Router |
| PoE    | `NET_POE_SWITCH` | PoE    |



#### 2.4 辅助与环境（Aux / Facility）

| 类别 | 命名示例         | 说明       |
| ---- | ---------------- | ---------- |
| 灯光 | `FAC_LIGHT`      | Relay 控制 |
| 空调 | `FAC_AC`         | IR 控制    |
| 窗帘 | `FAC_CURTAIN`    | Relay      |
| 红外 | `FAC_IR_BLASTER` | IR         |



### 三、二级分类：Templates（组合拓扑模块）

#### 命名规则（非常重要）

```css
TPL_[Scene]_[KeyConfig]
```

#### 教育场景（你现在用得最多）

| 模板名                   | 说明                |
| ------------------------ | ------------------- |
| `TPL_CLASSROOM_STANDARD` | 标准教室            |
| `TPL_LECTURE_2CAM_2MIC`  | 2 摄像头 + 2 吸顶麦 |
| `TPL_LECTURE_2CAM_4MIC`  | 大教室              |
| `TPL_RECORDING_LCS810`   | 录播模块            |
| `TPL_QNEX_AUDIO_BASIC`   | PS610 + S210        |
| `TPL_QNEX_PODIUM_NDP600` | 讲台核心            |

⚠️ 规则：

- Template 内 **可以有多个设备**
- **允许连线**
- 必须整体可拖拽（Group）



### 四、Solutions（完整方案，可选）

这个层级 **不是必须给 AI 用**，但对售前极其友好。

| 方案名                             | 说明       |
| ---------------------------------- | ---------- |
| `SOL_EDU_LECTURE_STANDARD_40_90M2` | 你图里这个 |
| `SOL_EDU_RECORDING_PRO`            | 高配录播   |
| `SOL_MEETING_STANDARD`             | 会议室     |

> Solution 本质 = 多个 Template + 少量 Component



### 五、分类疑问

> 但是我这里事想说，S210这种麦克风，100%是和LCS搭配，不可能和NDP搭配，那么还是按照这类划分，对于后期维护方便吗  

你现在的想法是：

> “S210 100% 和 LCS 搭配，那我是不是干脆把 S210 归到 LCS 下面？”

从短期看，很爽；
 从中长期看，一定出问题。

#### 坑 1：产品演进一定会打脸

今天：

- S210 = LCS 专用

明年很可能：

- 新一代 NDP 支持 PoE 吸顶麦
- 或 S210 出一个 “USB / Dante / Network” 版本
- 或第三方项目要求 **S210 → 外置 DSP → NDP**

如果你现在在**结构层**就把它“焊死”，
 那到时候不是改一条规则，而是**重构整个库**。



#### 坑 2：AI / 自动生成最怕“隐式耦合”

你后面想做的是：自然语言 → 自动选型 → 自动生成拓扑

AI 非常擅长处理：

- ✅ 显式规则
- ❌ 隐式假设

如果你把 S210 直接藏进 LCS 组件里：

- AI **看不到**“麦克风”这个概念
- 后期你想问：“这个方案里一共有几个麦？”会非常痛苦

#### 坑 3：售前维护角度，其实更难查

真实售前维护经常会问：

- 这个项目用了几只 S210？
- S210 对应的是哪个录播？
- 哪些模板用了 S210？

如果 S210 是独立组件 + 规则绑定 → **一查 manifest 就知道**
 如果 S210 被吞进 LCS → **只能靠人工看图**



#### 推荐的结构设计（你可以直接照抄）

1️⃣ Component 层（保持物理独立）

```
QNX_AUDIO_MIC_S210
QNX_LCS_LCS810
QNX_PODIUM_NDP600
```

**不在组件层表达“谁和谁一起用”**



2️⃣ Template 层（表达强业务绑定）

```css
TPL_LECTURE_LCS_2CAM_2MIC
TPL_LECTURE_LCS_2CAM_4MIC
```

这些 Template **内部一定包含**：

- LCS810
- S210 × N
- Camera × M

而且你可以明确约定：

> 所有带 `TPL_LECTURE_LCS_*` 的模板
>  → 默认包含 S210
>  → 不支持 NDP-only



3️⃣ Rule / Constraint 层（给 AI 用）

这是最关键、但不需要画图的部分。

你可以用 JSON / YAML / 纯约定：

```json
{
  "constraints": [
    {
      "component": "QNX_AUDIO_MIC_S210",
      "requires": ["QNX_LCS_LCS810"],
      "forbidden_with": ["QNX_PODIUM_NDP100", "QNX_PODIUM_NDP600"]
    }
  ]
}
```

或者更简单一点（售前够用）：

```css
S210 → ONLY_WITH → LCS
```



#### 这样设计的好处（现实收益）

对你现在

- 不影响你现有 100% 的项目逻辑
- 模板已经天然限制 S210 不会乱用

对 1–2 年后

- 新产品只要改规则，不动库
- AI 自动生成不会“幻觉搭配”

对售前新人

- 看 Template 就知道“这是录播体系”
- 不会拿 S210 去乱接 NDP



## 数据清洗



## Reference

1. https://chatgpt.com/c/694b9a86-a2b4-8323-9dfe-94793fbb2a41
