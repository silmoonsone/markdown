禁用某个Windows更新包



- 安装PSWindowsUpdate

```powershell
Install-PackageProvider -Name NuGet -Force
Install-Module PSWindowsUpdate -Force
```

- 隐藏更新，如隐藏KB5071547

```powershell
Hide-WindowsUpdate -KBArticleID KB5071547 -Confirm:$false
```

- 确认是否已经隐藏

```powershell
Get-WindowsUpdate -MicrosoftUpdate
```
