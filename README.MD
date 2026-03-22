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

- `manifest.json` 由仓库中的资源文件生成，包含每条资源的 `type`（MIME，如 `image/png` / `application/json`）、`role`（用途，如 `icon` / `cover` / `localization`）、`category`（如 `character` / `weapon` / `gacha_pool`）、`path`、`checksum`、`size`、`mtime`，以及整体 `metadata_checksum`、`package_version`。
- 当前 `entries` 会收录以下资源：
  - `images/character/icon/*.png`
  - `images/character/cover/*.png`
  - `images/weapon/icon/*.png`
  - `images/weapon/cover/*.png`
  - `locale/*/*.json`
- 生成命令（PowerShell）：

```powershell
function New-ManifestEntry {
  param(
    [Parameter(Mandatory = $true)][string]$Path,
    [Parameter(Mandatory = $true)][string]$Role,
    [Parameter(Mandatory = $true)][string]$Category,
    [Parameter(Mandatory = $true)][string]$Type
  )

  $file = Get-Item -LiteralPath $Path
  [pscustomobject]@{
    type     = $Type
    role     = $Role
    category = $Category
    path     = $Path.Replace('\', '/')
    size     = $file.Length
    mtime    = $file.LastWriteTimeUtc.ToString('o')
    checksum = (Get-FileHash -LiteralPath $Path -Algorithm SHA256).Hash
  }
}

$entries = @()

$imageDirs = @(
  @{ path = 'images/character/icon' ; role = 'icon'  ; category = 'character' },
  @{ path = 'images/character/cover'; role = 'cover' ; category = 'character' },
  @{ path = 'images/weapon/icon'    ; role = 'icon'  ; category = 'weapon'    },
  @{ path = 'images/weapon/cover'   ; role = 'cover' ; category = 'weapon'    }
)

foreach($dir in $imageDirs){
  Get-ChildItem -File $dir.path -Filter '*.png' |
    Sort-Object Name |
    ForEach-Object {
      $relativePath = $_.FullName.Substring($PWD.Path.Length + 1)
      $entries += New-ManifestEntry -Path $relativePath -Role $dir.role -Category $dir.category -Type 'image/png'
    }
}

$metaFiles = Get-ChildItem -File .\locale -Recurse -Filter '*.json' | Sort-Object FullName
foreach($file in $metaFiles){
  $relativePath = $file.FullName.Substring($PWD.Path.Length + 1)
  $category = [System.IO.Path]::GetFileNameWithoutExtension($file.Name)
  $entries += New-ManifestEntry -Path $relativePath -Role 'localization' -Category $category -Type 'application/json'
}

$metaRaw = ($metaFiles | ForEach-Object { Get-Content -Raw -Encoding UTF8 -LiteralPath $_.FullName }) -join ''
$metaHash = [BitConverter]::ToString([Security.Cryptography.SHA256]::Create().ComputeHash([Text.Encoding]::UTF8.GetBytes($metaRaw))).Replace('-', '')

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
