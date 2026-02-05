# endfield-cat-metadata

用于 `endfield-cat` 项目的角色/武器缩略图与元数据仓库。

## 目录结构

- `locale/{lang}/character.json`：角色元数据列表（JSON 数组）
- `locale/{lang}/gacha_pool.json`：卡池本地化数据（JSON 数组）
- `locale/{lang}/weapon.json`：武器元数据列表（JSON 数组）
- `images/character/icon/`：角色缩略图（`chr_*.png`）
- `images/weapon/icon/`：武器缩略图（`wpn_*.png`）

## 元数据文件

- `locale/{lang}/character.json`：角色数组
- `locale/{lang}/weapon.json`：武器数组

单条示例（存于对应数组）：

```json
{
  "type": "weapon",
  "name": "熔铸火焰",
  "itemid": "wpn_sword_0006",
  "rarity": 6
}
```

- `type`：`character` / `weapon`（小写）。
- `itemid`：唯一 ID；对应图片文件名（`images/<category>/icon/<itemid>.png`）。
- `name`：显示名。
- `rarity`：稀有度（整数）。

## 资源命名约定

- 仅收录并提交以 `chr_` / `wpn_` 命名的资源文件。
- 图片文件名必须与 `itemid` 完全一致：`images/character/icon/chr_XXXX_xxx.png`、`images/weapon/icon/wpn_XXXX_xxx.png`。
- 元数据中的每个 `itemid` 都应存在对应的 PNG 文件；反之亦然。
- 角色/武器缩略图统一放在：`images/character/icon/`、`images/weapon/icon/`。

## 快速校验（PowerShell）

使用 `manifest.json`（仅校验 `role=icon` 的记录）检查是否有缺失文件：

```powershell
$manifest = Get-Content -Raw -Encoding UTF8 .\manifest.json | ConvertFrom-Json
$icons = $manifest.entries | Where-Object { $_.role -eq 'icon' }
$missing = $icons | Where-Object { -not (Test-Path $_.path) }
$missing | Select-Object category,path,checksum
```

检查目录中是否存在“未在 `manifest.json` 声明”的 `chr_` / `wpn_` 图片：

```powershell
$manifest = Get-Content -Raw -Encoding UTF8 .\manifest.json | ConvertFrom-Json
$root = Get-Location
$declared = $manifest.entries | Where-Object { $_.role -eq 'icon' } | ForEach-Object {
  (Join-Path $root $_.path).Replace('\','/').ToLower()
}
$extra = Get-ChildItem -Recurse -File -Filter *.png -Path .\images |
  Where-Object {
    $_.BaseName -match '^(chr_|wpn_)' -and ($declared -notcontains ($_.FullName.Replace('\','/').ToLower()))
  } |
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

- `manifest.json` 由元数据生成，包含每条资源的 `type`（MIME，如 `image/png`）、`role`（用途，如 `icon`/`localization`）、`category`（如 `character`/`weapon`）、`path`、`checksum`、`size`、`mtime`，以及整体 `metadata_checksum`、`package_version`。
- 元数据文件（`locale/zh-CN/character.json`、`locale/zh-CN/weapon.json`、`locale/zh-CN/gacha_pool.json`）作为 `role=localization` 的记录加入 `entries`。
- 生成命令（PowerShell）：

```powershell
$char   = Get-Content -Raw -Encoding UTF8 .\locale\zh-CN\character.json | ConvertFrom-Json
$gacha  = Get-Content -Raw -Encoding UTF8 .\locale\zh-CN\gacha_pool.json | ConvertFrom-Json
$weapon = Get-Content -Raw -Encoding UTF8 .\locale\zh-CN\weapon.json   | ConvertFrom-Json

$entries = @()
foreach($i in $weapon){
  $path = "images/weapon/icon/$($i.itemid).png"
  $file = Get-Item $path
  $entries += [pscustomobject]@{
    type     = 'image/png'
    role     = 'icon'
    category = 'weapon'
    path     = $path
    size     = $file.Length
    mtime    = $file.LastWriteTimeUtc.ToString('o')
    checksum = (Get-FileHash $path -Algorithm SHA256).Hash
  }
}
foreach($i in $char){
  $path = "images/character/icon/$($i.itemid).png"
  $file = Get-Item $path
  $entries += [pscustomobject]@{
    type     = 'image/png'
    role     = 'icon'
    category = 'character'
    path     = $path
    size     = $file.Length
    mtime    = $file.LastWriteTimeUtc.ToString('o')
    checksum = (Get-FileHash $path -Algorithm SHA256).Hash
  }
}
$metaPaths = @(
  @{ path = 'locale/zh-CN/character.json'; category = 'character' },
  @{ path = 'locale/zh-CN/gacha_pool.json'; category = 'gacha_pool' },
  @{ path = 'locale/zh-CN/weapon.json'    ; category = 'weapon'    }
)
foreach($m in $metaPaths){
  $file = Get-Item $m.path
  $entries += [pscustomobject]@{
    type     = 'application/json'
    role     = 'localization'
    category = $m.category
    path     = $m.path.Replace('\','/')
    size     = $file.Length
    mtime    = $file.LastWriteTimeUtc.ToString('o')
    checksum = (Get-FileHash $m.path -Algorithm SHA256).Hash
  }
}
$metaRaw = Get-Content -Raw -Encoding UTF8 -Path ($metaPaths.path)
$metaHash = [BitConverter]::ToString([Security.Cryptography.SHA256]::Create().ComputeHash([Text.Encoding]::UTF8.GetBytes($metaRaw))).Replace('-','')

$manifest = [pscustomobject]@{
  package_version   = [DateTime]::UtcNow.ToString('yyyyMMddHHmmss')
  generated_at      = [DateTime]::UtcNow.ToString('o')
  metadata_checksum = $metaHash
  item_count        = $entries.Count
  entries           = $entries
}
$manifest | ConvertTo-Json -Depth 6 | Set-Content -Path .\manifest.json -Encoding UTF8
```

- 客户端可先比对 `package_version` / `metadata_checksum`，再按需校验单个资源的 `checksum` 与文件大小。
