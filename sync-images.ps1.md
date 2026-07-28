---
tags:
  - tool
  - 计算机网络
---

# Obsidian 图片同步脚本

将 Obsidian `![[...]]` 图片引用转为标准 Markdown，并把图片复制到笔记目录的 `attachments/` 子文件夹中。处理后笔记文件夹即自包含，可直接推 GitHub，Obsidian 和 GitHub 均能正常显示图片。

## 用法

```powershell
# 默认处理当前 study/计算机网络 目录
.\sync-images.ps1.md

# 或指定其他笔记目录
.\sync-images.ps1.md -TargetDir "E:\Obsidian\zeluo2025\zeluo2025\study\数据结构"
```

## 脚本

```powershell
<#
.SYNOPSIS
    将 Obsidian 笔记中的 ![[Pasted image ...]] 转为标准 Markdown 图片语法，
    并同步图片到笔记目录的 attachments/ 子文件夹。

.DESCRIPTION
    1. 扫描指定目录下所有 .md 文件中的 ![[Pasted image ...]] 引用
    2. 从 Obsidian 全局附件目录复制图片到 ./attachments/
    3. 将 wikilink 语法替换为 ![...](attachments/...) 标准 Markdown

.PARAMETER TargetDir
    要处理的笔记目录路径。默认为当前脚本所在目录。

.PARAMETER AttachmentSource
    Obsidian 全局附件目录。默认为 "$VaultRoot/00 attachment"。

.EXAMPLE
    .\sync-images.ps1.md

.EXAMPLE
    .\sync-images.ps1.md -TargetDir "E:\Obsidian\zeluo2025\zeluo2025\study\数据结构"
#>

param(
    [string]$TargetDir = $PSScriptRoot,
    [string]$AttachmentSource = ""
)

$ErrorActionPreference = "Stop"

# ---- 自动推断 Obsidian vault 根目录和附件源 ----
if (-not $AttachmentSource) {
    # 从 TargetDir 向上找 .obsidian 目录来确定 vault 根
    $path = $TargetDir
    while ($path -and (Split-Path $path -Leaf) -ne "") {
        if (Test-Path (Join-Path $path ".obsidian")) {
            $vaultRoot = $path
            break
        }
        $path = Split-Path $path -Parent
    }
    if (-not $vaultRoot) {
        Write-Error "未找到 .obsidian 配置目录，无法确定 Vault 根目录。"
        exit 1
    }
    # 读取 Obsidian 配置中的附件目录设置
    $appJson = Join-Path $vaultRoot ".obsidian\app.json"
    if (Test-Path $appJson) {
        $config = Get-Content $appJson -Raw | ConvertFrom-Json
        $attachFolder = $config.attachmentFolderPath
        if ($attachFolder) {
            $AttachmentSource = Join-Path $vaultRoot $attachFolder
        }
    }
    if (-not $AttachmentSource) {
        $AttachmentSource = Join-Path $vaultRoot "00 attachment"
    }
}

Write-Host "Target dir : $TargetDir"
Write-Host "Image source: $AttachmentSource"
Write-Host ""

$attachDst = Join-Path $TargetDir "attachments"

# ---- 1. 创建目标 attachments 目录 ----
New-Item -ItemType Directory -Path $attachDst -Force | Out-Null

# ---- 2. 扫描所有 .md 文件中的图片引用 ----
$mdFiles = Get-ChildItem -Path $TargetDir -Filter "*.md"
$imageRefs = @{}  # filename -> original full reference (with possible |size)

foreach ($file in $mdFiles) {
    $content = Get-Content $file.FullName -Raw
    $matches = [regex]::Matches($content, '!\[\[(Pasted image [^\]]+)\]\]')
    foreach ($m in $matches) {
        $fullRef = $m.Groups[1].Value
        $filename = ($fullRef -split '\|')[0]
        if (-not $imageRefs.ContainsKey($filename)) {
            $imageRefs[$filename] = @{
                FullRef = $fullRef
                Files    = @()
            }
        }
        $imageRefs[$filename].Files += $file.Name
    }
}

Write-Host "找到 $($imageRefs.Count) 张引用图片"

# ---- 3. 复制图片 ----
$copied = 0
$missing = @()
foreach ($img in $imageRefs.Keys) {
    $src = Join-Path $AttachmentSource $img
    $dst = Join-Path $attachDst $img
    if (Test-Path $src) {
        Copy-Item $src $dst -Force
        $copied++
    } else {
        $missing += $img
    }
}
Write-Host "已复制: $copied 张"
if ($missing.Count -gt 0) {
    Write-Host "未找到: $($missing.Count) 张" -ForegroundColor Yellow
    $missing | ForEach-Object { Write-Host "  - $_" -ForegroundColor Yellow }
}

# ---- 4. 替换 wikilink 为标准 Markdown ----
foreach ($file in $mdFiles) {
    $content = Get-Content $file.FullName -Raw
    $newContent = [regex]::Replace($content,
        '!\[\[(Pasted image [^\]]+?)\]\]',
        {
            param($m)
            $ref = $m.Groups[1].Value
            $filename = ($ref -split '\|')[0]
            "![$filename](attachments/$filename)"
        }
    )
    Set-Content $file.FullName -Value $newContent -NoNewline
    Write-Host "已更新: $($file.Name)"
}

Write-Host ""
Write-Host "完成！现在 `$TargetDir` 已自包含，可直接推 GitHub。" -ForegroundColor Green
```

## 说明

- 脚本会自动读取 `.obsidian/app.json` 中的 `attachmentFolderPath` 配置，找到图片源目录
- 只处理 `Pasted image` 开头的图片引用，不会影响其他 wikilinks
- Obsidian 的尺寸限定符（`|697`）会被移除，不影响图片显示
- 脚本可重复执行，已复制的图片会覆盖更新
