# endfield-cat-metadata

用于 `endfield-cat` 项目的角色/武器图标与元数据仓库。

## 目录结构

- `metadata.json`：元数据列表（JSON 数组）
- `Character/`：角色图标（`chr_*.png`）
- `Weapon/`：武器图标（`wpn_*.png`）

## `metadata.json` 格式

根节点为数组，每个元素示例：

```json
{
  "type": "Weapon",
  "name": "熔铸火焰",
  "itemid": "wpn_sword_0006",
  "rarity": 6
}
```

- `type`：`Character` 或 `Weapon`（同时也是图片所在目录名）
- `name`：显示名
- `itemid`：唯一 ID；对应图片文件名（`<type>/<itemid>.png`）
- `rarity`：稀有度（整数）

## 资源命名约定

- 仅收录并提交以 `chr_` / `wpn_` 命名的资源文件。
- 图片文件名必须与 `itemid` 完全一致：`Character/chr_XXXX_xxx.png`、`Weapon/wpn_XXXX_xxx.png`。
- `metadata.json` 中的每个 `itemid` 都应存在对应的 PNG 文件；反之亦然。

## 快速校验（PowerShell）

检查 `metadata.json` 中是否存在缺失图片：

```powershell
$items = Get-Content -Raw -Encoding UTF8 .\metadata.json | ConvertFrom-Json
$missing = $items | Where-Object {
  -not (Test-Path (Join-Path $_.type ($_.itemid + '.png')))
}
$missing | Format-Table type,itemid,name,rarity
```

检查目录中是否存在“未在 metadata.json 声明”的 `chr_` / `wpn_` 图片：

```powershell
$items = Get-Content -Raw -Encoding UTF8 .\metadata.json | ConvertFrom-Json
$declared = $items.itemid
$extra = Get-ChildItem -Recurse -File -Filter *.png |
  Where-Object { $_.BaseName -match '^(chr_|wpn_)' -and ($declared -notcontains $_.BaseName) } |
  Select-Object FullName
$extra
```

## 提交流程

1. 添加 PNG（按命名约定放入 `Character/` 或 `Weapon/`）。
2. 在 `metadata.json` 新增/更新对应条目。
3. 运行上面的校验脚本确认一致性。
4. 提交并推送。

## 说明

请确保你有权提交/分发相关资源文件。

## manifest.json

- `manifest.json` 由 `metadata.json` 生成，包含每条资源的 `path`、`checksum`、`size`、`mtime`，以及整体 `metadata_checksum`、`package_version`。
- 生成命令（PowerShell）：

```powershell
$items = Get-Content -Raw -Encoding UTF8 .\metadata.json | ConvertFrom-Json
$entries = foreach($i in $items){
  $path = Join-Path $i.type ($i.itemid + '.png')
  $file = Get-Item $path
  [pscustomobject]@{
    type     = $i.type
    itemid   = $i.itemid
    name     = $i.name
    rarity   = $i.rarity
    path     = $path
    size     = $file.Length
    mtime    = $file.LastWriteTimeUtc.ToString('o')
    checksum = (Get-FileHash $path -Algorithm SHA256).Hash
  }
}
$manifest = [pscustomobject]@{
  package_version   = [DateTime]::UtcNow.ToString('yyyyMMddHHmmss')
  generated_at      = [DateTime]::UtcNow.ToString('o')
  metadata_checksum = (Get-FileHash .\metadata.json -Algorithm SHA256).Hash
  item_count        = $items.Count
  entries           = $entries
}
$manifest | ConvertTo-Json -Depth 5 | Set-Content -Path .\manifest.json -Encoding UTF8
```

- 客户端可先比对 `package_version` / `metadata_checksum`，再按需校验单个资源的 `checksum` 与文件大小。
