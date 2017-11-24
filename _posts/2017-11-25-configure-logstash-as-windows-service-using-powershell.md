---
title: 使用 Powershell 將 Logstash 設為 Windows 服務
layout: post
category: logstash powershell
tags: logstash windows service powershell
comments: true
---

```powershell
# Logstash 自動安裝程式
#
# 注意事項：
#
#  1. 僅支援 PowerShell 5.1 以上版本。
#  2. 本程式將以 Administrator 身份執行。
#  3. 請先安裝 Java 8 (或更高版本)。
#
# 參考：
#
#   1. [PowerShell](https://docs.microsoft.com/zh-tw/powershell/scripting/powershell-scripting?view=powershell-5.1)
#
# @since 2017-11-25 09:00:12
# @author Gary Liu<desp0916@gmail.com>
#

# ===================== 函式 =====================

# 檢查是否有安裝 Java？
function CheckJavaInstalled {
    Try {
        Start-Process java -ArgumentList "-version" -NoNewWindow -Wait -ErrorAction SilentlyContinue
        return $?
    } Catch {
        return $false
        Break
    }
}

# 取得目前系統中的 Java 版本
function GetJavaVersion {
    $p = Start-Process java -ArgumentList "-version" -NoNewWindow -RedirectStandardError .\javaver.txt -Wait
    $JavaVersion = ((Get-Content .\javaver.txt) -split '\n')[0]
    $x = del .\javaver.txt
    return $JavaVersion
}

# 判斷 Java 版本是否大於 8？
function CheckJava8OrAbove {
    param ([string] $javaVersion)
    $distribution,$string,$verStr=$javaVersion.Split(' ')
    $major,$minor,$rev=($verStr -replace '"', '').Split('.')
    $equalOrGreaterThanEight = $minor -ge 8
    return $equalOrGreaterThanEight
}

# 取得 script 目前所在的路徑
# https://stackoverflow.com/a/19542600
function GetScriptDirectory {
    $Invocation = (Get-Variable MyInvocation -Scope 1).Value;
    if ($Invocation.PSScriptRoot) {
        $Invocation.PSScriptRoot;
    } Elseif ($Invocation.MyCommand.Path) {
        Split-Path $Invocation.MyCommand.Path
    } else {
        $Invocation.InvocationName.Substring(0,$Invocation.InvocationName.LastIndexOf("\"));
    }
}

# 解壓縮 Unzip
# https://stackoverflow.com/questions/27768303/how-to-unzip-a-file-in-powershell
function Unzip {
    param([string]$zipfile, [string]$outpath)
    [System.IO.Compression.ZipFile]::ExtractToDirectory($zipfile, $outpath)
}

# 取得 Logstash 的安裝路徑
function GetLogstashHomePath($LogstashZipFilePath) {
    $ext = [IO.Path]::GetExtension($LogstashZipFilePath)
    $LogstashHomePath = $LogstashZipFilePath.Replace($ext, "")
    return $LogstashHomePath
}

# 建立本地使用者 aplogger
function CreateLocalUser($UserName, $Password) {

#$UserName = if(($result = Read-Host "請輸入 aplogger 的帳號名稱 [aplogger]") -eq ''){"aplogger"}else{$result}
#$UserName = Read-Host -Prompt "請輸入 aplogger 的帳號名稱"
#$Password = Read-Host -AsSecureString -Prompt "請輸入 aplogger 的密碼:"
#New-LocalUser -Name $UserName -Password $Password -FullName "AP Logger" -Description "AP Log 分析系統 agent"

    Try {
        $SecureStringPassword = ConvertTo-SecureString $Password -AsPlainText -Force
        New-LocalUser -Name $UserName -Password $SecureStringPassword -FullName "AP Logger" -Description "AP Logger" -ErrorAction SilentlyContinue
        return $true
    } Catch [Exception] {Microsoft.PowerShell.LocalAccounts
        Write-Host $_.Exception.Message -ForegroundColor Red
        return $false
        #Break
    }
}

# 將 aplogger 加入 Remote Desktop Users 群組
function AddToRDPGroup($UserName, $RDPGroup = "Remote Desktop Users") {
    Add-LocalGroupMember -Group $RDPGroup -Member $UserName
}

# 啟用 Windows Server 2012 R2 上的 Telnet Client
# https://stackoverflow.com/questions/7330187/how-to-find-the-windows-version-from-the-powershell-command-line
# https://msdn.microsoft.com/en-us/library/windows/desktop/ms724833(v=vs.85).aspx
function EnableTelnetClientOnWinServer {
    # Windows Server 2012 R2 的 ProductType == VER_NT_SERVER (3)
    $ProductType = (Get-CimInstance Win32_OperatingSystem).ProductType
    # Windows Server 2012 R2 的 Version == 6.3
    $Version = [System.Environment]::OSVersion.Version
    If ($ProductType -eq 3 -and $Version.Major -eq 6 -and $Version.Minor -eq 3) {
        Write-Host "啟用 Windows Server 2012 R2 上的 Telnet Client...."
        Import-Module servermanager
        Add-WindowsFeature telnet-client
        return $True
    }
}

# 將 logstash 的 bin 目錄設定到 PATH 環境變數之中
# https://docs.microsoft.com/zh-tw/powershell/module/microsoft.powershell.localaccounts/get-localuser?view=powershell-5.1
function AddLogstashBinToEnvPath($LogstashHomePath) {
    Write-Host "==> 檢查 $LogstashHomePath 是否已存在於使用者的 %PATH% 環境變數中？" -ForegroundColor Green
    $exits = $env:Path -split ";" | Foreach {$_.trim('\')} | Select-String -Pattern $([regex]::Escape($LogstashHomePath.trim('\')))
    If ($exits) {
        Write-Warning "$LogstashHomePath 已存在於使用者的 %PATH% 環境變數中。"
    } Else {
        Write-Host "將 $LogstashHomePath 加入使用者的 `$PATH 環境變數中..."
        # https://stackoverflow.com/questions/714877/setting-windows-powershell-path-variable
        [Environment]::SetEnvironmentVariable("Path", $env:Path + ";" + $LogstashHomePath + "\bin", [EnvironmentVariableTarget]::User)
    }
}

# https://stackoverflow.com/questions/2602460/powershell-to-manipulate-host-file
# https://stackoverflow.com/questions/13279580/how-to-interpolate-variables-in-regular-expressions-in-powershell
# Uncomment lines with $HostName on them:
function UncommentSomeHost($HostName) {
    $hostsPath = "$env:windir\System32\drivers\etc\hosts"
    $hosts = Get-Content $hostsPath
    $hosts = $hosts | Foreach {if ($_ -match "^\s*#\s*(.*?\d{1,3}.*?$([regex]::Escape($HostName)).*)")
                               {$matches[1]} else {$_}}
    $hosts | Out-File $hostsPath -enc ascii
}

# Comment lines with $HostName on them:
function CommentSomeHost($HostName) {
    $hostsPath = "$env:windir\System32\drivers\etc\hosts"
    $hosts = Get-Content $hostsPath
    $hosts | Foreach {if ($_ -match "^\s*([^#].*?\d{1,3}.*?$([regex]::Escape($HostName)).*)")
                      {"# " + $matches[1]} else {$_}} |
             Out-File $hostsPath -enc ascii
}

# 取得本地使用者帳號
function GetLocalUser($UserName) {
   # Powershell 5.1:
   $User = Get-LocalUser -Name $UserName -ErrorAction SilentlyContinue
   # $User = Get-WmiObject -Class Win32_UserAccount -Filter "LocalAccount='True' and Name='$UserName'" -ErrorAction SilentlyContinue
   return $User
}

# 修改 AP Log 目錄的存取權限
# https://technet.microsoft.com/zh-tw/library/ff730951.aspx
# https://msdn.microsoft.com/zh-tw/library/system.security.accesscontrol.filesystemrights(v=vs.110).aspx
function ChangeApLogPathACL($Username, $ApLogPath) {
    $colRights = [System.Security.AccessControl.FileSystemRights]"ReadAndExecute, ListDirectory"
    $InheritanceFlag = [System.Security.AccessControl.InheritanceFlags]::ObjectInherit
    $PropagationFlag = [System.Security.AccessControl.PropagationFlags]::InheritOnly
    $objType =[System.Security.AccessControl.AccessControlType]::Allow
    $objUser = New-Object System.Security.Principal.NTAccount($Username)
    $objACE = New-Object Security.AccessControl.FileSystemAccessRule "$env:COMPUTERNAME\$UserName", 'ReadAndExecute, ListDirectory', 'ContainerInherit, ObjectInherit', 'InheritOnly', 'Allow'

    $objACL = Get-ACL $ApLogPath
    $objACL.AddAccessRule($objACE)

    Set-ACL $ApLogPath $objACL
}

# ===================== 主程式開始 =====================

# 重要設定
$UserName = "aplogger"
$Password = "Pa$$w0rd"
#$WorkingPath = GetScriptDirectory
$WorkingPath = $PSScriptRoot
$ApLogPath = "C:\log"
$SystemHostsPath = "$env:windir\System32\drivers\etc\hosts"
$APLogHostFile = "hosts-staging.txt"
$APLogHostFilePath = "$($WorkingPath)\$($APLogHostFile)"
$LogstashZipFile = "logstash-5.6.4.zip"
$LogstashZipFilePath = "$($WorkingPath)\$($LogstashZipFile)"
$LogstashHomePath = GetLogstashHomePath $LogstashZipFilePath
$ServiceName = "Logstash"
$RDPGroup = "Remote Desktop Users"
$PSVersion = $PSVersionTable.PSVersion

# 檢查 Powershell 版本

Write-Host "==> 檢查 Powershell 是否為 5.1 以上版本？" -ForegroundColor Green

$PSVersion | Format-Table

If (-Not ($PSVersion.Major -ge 5 -and $PSVersion.Minor -ge 1)) {
    Write-Warning "本程式僅支援 Powershell 5.1 以上版本，本程式即將於 10 秒後結束。"
    Write-Warning "建議升級至 Windows Management Framework 5.1 或以上版本:"
    Write-Warning "https://docs.microsoft.com/zh-tw/powershell/wmf/5.1/install-configure"
    Start-Sleep -s 10
    exit
}

Add-Type -AssemblyName System.IO.Compression.FileSystem
Set-location $WorkingPath

# 本 Script 必須以 Administrator 的身份執行！
# https://stackoverflow.com/questions/7690994/powershell-running-a-command-as-administrator
Write-Host "==> 以 Administrator 身份執行...." -ForegroundColor Green

If (-Not ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {  
    $arguments = "& '" + $MyInvocation.mycommand.definition + "'"
    Start-Process powershell -Verb runAs -ArgumentList $arguments
    exit
}

EnableTelnetClientOnWinServer

Write-Host "==> 檢查本機帳號 $UserName 是否已存在？" -ForegroundColor Green

If (GetLocalUser($UserName)) {
    Write-Warning "本機帳號 $UserName 已存在。"
} Else {
    Write-Host "本機帳號 $UserName 不存在！建立本機帳號 $UserName..."
    CreateLocalUser $UserName $Password
}

Write-Host "==> 檢查本機帳號 $UserName 是否已加入 $RDPGroup 群組中？" -ForegroundColor Green

If ((Get-LocalGroupMember -Group $RDPGroup -Member $UserName -ErrorAction SilentlyContinue)) {
    Write-Warning "aplogger 帳號已加入 $RDPGroup 群組中。"
} Else {
    Write-Host "將 aplogger 加入 $RDPGroup 群組中..."
    AddToRDPGroup $UserName
}

# 變更 AP log 路徑的存取權限 (讓 aplogger 也能讀取)
Write-Host "===> 變更 AP log 路徑（$ApLogPath）的存取權限..." -ForegroundColor Green
ChangeApLogPathACL $UserName $ApLogPath
Get-Acl $ApLogPath | Format-List

# 更新系統 hosts 檔（加入 AP Log 分析系統的 hosts）
# TODO:
#   1. 檢查是否系統 hosts 是否已經有加入過了？
#   2. 檢查 $Hosts 是否存在？
Write-Host "==> 修改系統 hosts 檔（$SystemHostsPath）..." -ForegroundColor Green
Add-Content -Path $SystemHostsPath -Value "`n`n", (Get-Content -Path $APLogHostFilePath)

Write-Host "==> 檢查是否已安裝 Java？" -ForegroundColor Green

If (CheckJavaInstalled) {

    Write-Warning "Java 已經安裝"

    # 檢查是否為 Java 8 以上版本？
    Write-Host "==> 檢查是否為 Java 8 以上版本？" -ForegroundColor Green
    $JavaVersion = GetJavaVersion
    Write-Host $JavaVersion

    If (CheckJava8OrAbove($javaVersion)) {

        # 檢查 $LogstashHomePath 目錄是否已經存在？
        Write-Host "==> 檢查 $LogstashHomePath 目錄是否已經存在？" -ForegroundColor Green

        If (Test-Path $LogstashHomePath) {
            Write-Warning "$LogstashHomePath 目錄已存在。"
        } Else {
            Write-Host "開始解壓縮 Logstash，請稍候...."
            Unzip $LogstashZipFilePath $WorkingPath
        }

        # 設定 %PATH%
        AddLogstashBinToEnvPath $LogstashHomePath

        # 將 Logstash 設定為服務
        Write-Host "==> 檢查是 $ServiceName 是否服務已存在？" -ForegroundColor Green

        $Service = Get-Service -Name $ServiceName -ErrorAction SilentlyContinue

        If ($Service) {
            Write-Warning "$ServiceName 服務已存在。"
        } Else {
            # https://blogs.msdn.microsoft.com/koteshb/2010/02/12/powershell-how-to-create-a-pscredential-object/
            # https://blogs.technet.microsoft.com/gary/2009/07/23/creating-a-ps-credential-from-a-clear-text-password-in-powershell/
            $BinaryPath = $LogstashHomePath + "\bin\logstash.bat -f " + $WorkingPath + "\logstash.conf"
            $secpasswd = ConvertTo-SecureString $Password -AsPlainText -Force
            $Credential = New-Object System.Management.Automation.PSCredential("$env:COMPUTERNAME\$UserName", $secpasswd)
            New-Service -Name $ServiceName -DisplayName $ServiceName -Description "$ServiceName service." -BinaryPathName $BinaryPath -Credential $Credential
        }

        $Service = Get-Service -Name $ServiceName -ErrorAction SilentlyContinue

        # 啟動服務
        Write-Host "==> 啟動 $ServiceName 服務..." -ForegroundColor Green
        $ConfirmStart = Start-Service -InputObject $Service -Confirm -PassThru | Format-Table

        # 查看 Logstash 的 logs
        If ($ConfirmStart) {
            Write-Host "==> 查看 Logstash 的 logs" -ForegroundColor Green
            Get-Content -Path $($LogstashHomePath + "\logs\logstash-plain.log") -Wait
        }

    } Else {
        Write-Error "請安裝 Java 8 以上的版本！"
    }

} Else {
    Write-Error "尚未安裝 Java！請先安裝 Java 8 或以上版本。"
}

Write-Warning "==> 本程式即將於 60 秒後結束..."
Start-Sleep 60

```