
# Custom đường dẫn trong powershell

Bash (PowerShell) 
```
notepad $profile
```
## Định dạng theo chuẩn Linux: user@hostname:~/path$

```
function prompt {
    $user = $env:USERNAME

    $computer = $env:COMPUTERNAME.ToLower()

    $path = $pwd.Path.Replace($HOME, "~")

    Write-Host -NoNewline "$user@$computer" -ForegroundColor Green

    Write-Host -NoNewline ":" -ForegroundColor White

    Write-Host -NoNewline "$path" -ForegroundColor Blue

    return "$ "
}
```
# Chia 4 màn hình powershell
```
function quad {
    wt -M split-pane -H `; split-pane -V `; focus-pane -t 0 `; split-pane -V
}
```
