# 禁用某个 Windows 更新包

## 1. 安装 PSWindowsUpdate

```powershell
Install-PackageProvider -Name NuGet -Force
Install-Module PSWindowsUpdate -Force
```

## 2. 隐藏更新（示例：KB5071547）

```powershell
Hide-WindowsUpdate -KBArticleID KB5071547 -Confirm:$false
```

## 3. 确认是否已隐藏

```powershell
Get-WindowsUpdate -MicrosoftUpdate
```
