# Windows Gold Disk Building

<details><summary>References</summary>https://www.elevenforum.com/tutorials/</br>
https://www.winos.me/</br>
https://github.com/Chuyu-Team/Dism-Multi-language</br>
https://www.elevenforum.com/tutorials/?prefix_id=7</br>
https://www.elevenforum.com/tutorials/?prefix_id=12</br>
https://www.tenforums.com/tutorials/id-Installation_Upgrade/</br>
https://www.tenforums.com/tutorials/id-Virtualization/</br>
nsnfrm topic/249660-disable-windows-10-telemetry-and-data-collection-collection-of-methods-tools</br>
https://devblogs.microsoft.com/scripting/automatically-enable-and-disable-trace-logs-using-powershell/</br>
https://duckduckgo.com/?q=windows+11+disable+logging+tracing&ia=web</br>
</details>


## 1. Set up an environment to perform the modifications.
<details><summary>References</summary>DevOps practices</details>

```powershell
# placeholder
# preferably download distribution files inside a VM
# bring up a VM
# automate the ISO build/ISO slim
```


### 1.2. Slim down the ISO
```powershell
# placeholder
```
### References:
NTLite Windows11 Tuning PreSetupStage xml

## 2. Cloud


## 3. OS settings
Pre-configure OS settings  
+ Slimdown for all use cases  
+ Improve speed performance latency  
+ Improve reliability and reduce infosec risk  
+ Reduce energy footprint  
+ Empower the user correctly  
+ Reduce maintenaince risk and cost  

### 3.1 Power Management
<details><summary>References</summary>https://www.softpedia.com/get/System/Launchers-Shutdown-Tools/Power-Plan-Assistant.shtml<br/>
https://gist.github.com/raspi/203aef3694e34fefebf772c78c37ec2c#file-enable-all-advanced-power-settings-ps1-L5<br/>
https://gist.github.com/Nt-gm79sp/1f8ea2c2869b988e88b4fbc183731693<br/>
https://www.tenforums.com/performance-maintenance/149514-list-hidden-power-plan-attributes-maximize-cpu-performance.html<br/>
https://www.tenforums.com/tutorials/107613-add-remove-ultimate-performance-power-plan-windows-10-a.html<br/>
https://forums.guru3d.com/threads/windows-power-plan-settings-explorer-utility.416058<br/>
https://www.notebookcheck.net/Useful-Life-Hack-How-to-Disable-Modern-Standby-Connected-Standby.453125.0.html<br/>
https://www.dell.com/community/XPS/How-to-disable-modern-standby-in-Windows-21H1/td-p/7996308<br/>
</details>

```powershell

# get rid of hibernation
powercfg -h off

# use normal standby and not modern standby

powercfg /setdcvalueindex scheme_current sub_none F15576E8-98B7-4186-B944-EAFA664402D9 0
powercfg /setacvalueindex scheme_current sub_none F15576E8-98B7-4186-B944-EAFA664402D9 0
REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Power\PowerSettings\F15576E8-98B7-4186-B944-EAFA664402D9 /v Attributes /t REG_DWORD /d 2 /f

Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Power" -Name "CsEnabled" -Value 0 -ErrorAction SilentlyContinue

# Coalescing IO - will introduce IO latency to save power

```



### 3.2 Disk Encryption
Delay encryption and present user choice:

```powershell
1. `fsutil behavior set disableencryption 1`: Disable encryption on the file system.
2. `cipher /d /s:C:\`: Decrypt all encrypted files on the C drive. Note that this command only works for files encrypted with the Encrypting File System (EFS). You should be logged in as the user who encrypted the files or an administrator who has the EFS recovery agent certificate. Otherwise, the command will fail, and the files will remain encrypted.
3. `reg add "HKLM\Software\Policies\Microsoft\Windows\EnhancedStorageDevices" /v "TCGSecurityActivationDisabled" /t REG_DWORD /d "1" /f`: Disable the Trusted Platform Module (TPM) security activation to prevent automatic encryption of new storage devices.
4. `sc config BDESVC start= disabled`: Disable the BitLocker Drive Encryption Service, which is responsible for managing BitLocker operations.
5. `sc config "EFS" start= disabled`: Disable the Encrypting File System (EFS) service, which manages EFS operations.

fsutil behavior set disableencryption 1
cipher /d /s:C:
reg add "HKLM\Software\Policies\Microsoft\Windows\EnhancedStorageDevices" /v "TCGSecurityActivationDisabled" /t REG_DWORD /d "1" /f
sc config BDESVC start= disabled
sc config "EFS" start= disabled

# Add dekstop icon to start Encryption upon user decision

```

## 3.3 IO Optimization
### 3.3.1 Eliminate everything log, performance counter, record keeping, temp files related
<details><summary>References</summary>https://yandex.com/search/?text=CrashControl+EnableLogFile&lr=10379 :: this search engine returns better results.</details>

```powershell


# Write Cache
# Ensure the script is running with administrative privileges
if (-NOT ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Error "This script needs to be run as an Administrator. Exiting..."
    exit
}

# Maximize write cache via registry (this sets the LargeSystemCache to 1, which maximizes cache)
$registryPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management"
Set-ItemProperty -Path $registryPath -Name "LargeSystemCache" -Value 1

Write-Host "Maximized write cache via registry."





- symlink logs and tempfiles to > NUL
(example)
@echo off

:: Disable unnecessary event logging
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v "EnableLogFile" /t REG_DWORD /d "0" /f

:: Disable automatic memory dump creation
reg add "HKLM\SYSTEM\CurrentControlSet\Control\CrashControl" /v "CrashDumpEnabled" /t REG_DWORD /d "0" /f

:: Disable DumpStack.log and DumpStack.log.tmp creation
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Reliability" /v "StackTraceDatabaseLogEnable" /t REG_DWORD /d "0" /f

:: Disable Windows Error Reporting
reg add "HKLM\SOFTWARE\Microsoft\Windows\Windows Error Reporting" /v "Disabled" /t REG_DWORD /d "1" /f


:: Delete existing DumpStack.log and DumpStack.log.tmp files
del /f /q C:\DumpStack.log
del /f /q C:\DumpStack.log.tmp

:: Create a RAM drive (adjust drive letter and size as needed)
imdisk -a -s 512M -m R: -p "/fs:ntfs /q /y"

:: preferably the winpe ramdrive will be more useful

:: Redirect event log files to the RAM drive (replace R: with the desired drive letter)
wevtutil el > event_logs.txt
for /f "tokens=*" %%A in (event_logs.txt) do (
    wevtutil sl %%A /lfn:"R:\%%A.evtx"
)

:: Clean up
del /f /q event_logs.txt

```


### 3.3.2 Add a button to reverse the above as needed

## 3.4 Drivers

Prevent out-of-date drivers from MS update

``` powershell

reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" /v "ExcludeWUDriversInQualityUpdate" /t REG_DWORD /d "1" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\DeviceInstall\Settings" /v "DeviceInstallDisabled" /t REG_DWORD /d "1" /f

```

## 3.5 Updates
<details><summary>References</summary>https://techcommunity.microsoft.com/t5/windows-it-pro-blog/the-windows-update-policies-you-should-set-and-why/ba-p/3270914</details>

``` powershell
:: Set Windows Update policy to receive stable updates only
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "DeferFeatureUpdates" /t REG_DWORD /d "0" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "DeferFeatureUpdatesPeriodInDays" /t REG_DWORD /d "0" /f

:: Set Windows Update to check for updates frequently
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "ScheduledInstallDay" /t REG_DWORD /d "0" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "ScheduledInstallTime" /t REG_DWORD /d "1" /f


:: other Microsoft product updates through Windows Update
(example compatible with Windows 8, 8.1, 10, and 11.)
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update" /v "IncludeRecommendedUpdates" /t REG_DWORD /d "1" /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Services\7971f918-a847-4430-9279-4a52d1efe18d" /v "RegisteredWithAU" /t REG_DWORD /d "1" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" /v "DoNotConnectToWindowsUpdateInternetLocations" /t REG_DWORD /d "0" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "IncludeRecommendedUpdates" /t REG_DWORD /d "1" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "Include_WSUS31" /t REG_DWORD /d "1" /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v "Include_MSUpdate" /t REG_DWORD /d "1" /f
```



## 3.6 Network
### 3.6.1. Turn off unused network protocols with a scheduled task
  client for ms net  
  file and pr sharing  
  register w dns  
  netbios  
  wi fi wake  
  eth fc  
  bluetooth

```powershell
Register-ScheduledTask -TaskName "DisableNetworkBindings" -Trigger (New-ScheduledTaskTrigger -OnEventID 4004 -User "NT AUTHORITY\SYSTEM") -Action (New-ScheduledTaskAction -Execute "Powershell.exe" -Argument "Disable-NetAdapterBinding -Name * -ComponentID ms_msclient, ms_server, ms_serverdriver, ms_tcpip6, ms_wuguid, ms_wusnmp, ms_lltdio, ms_rspndr, ms_nwifi, ms_msclientio, ms_ndisuio, ms_rdma_ndk, ms_rdma_rspndr, ms_rdma_tcp, ms_rdma_udp, ms_tcpip -PassThru | Disable-NetAdapterBinding -Name * -ComponentID ms_netbt, ms_lldp, ms_wfplwf, ms_wfpcpl, ms_pacer | Set-NetAdapterAdvancedProperty -Name * -DisplayName 'Flow Control' -DisplayValue 'Disabled'") -Settings (New-ScheduledTaskSettingsSet -Priority 4 -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1)) -Force

reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Dnscache\Parameters" /v GlobalQueryBlockList /t REG_MULTI_SZ /d "local,localhost,localdomain,local.,*\nmdns,*.local" /f

netsh advfirewall firewall add rule name="Block UDP 5353" dir=in action=block protocol=UDP localport=5353
netsh advfirewall firewall add rule name="Block UDP 5353" dir=out action=block protocol=UDP localport=5353
netsh advfirewall firewall add rule name="Block UDP 1900" dir=in action=block protocol=UDP localport=1900
netsh advfirewall firewall add rule name="Block UDP 1900" dir=out action=block protocol=UDP localport=1900

netsh advfirewall firewall add rule name="Block IGMP" dir=in action=block protocol=IGMP
netsh advfirewall firewall add rule name="Block IGMP" dir=out action=block protocol=IGMP

sc config Bonjour Service start=disabled :: make this a scheduledtask
```


### Add button to enable per user need (the igmp upnp mdns and ssdp are used for multimedia stuff)
 - placeholder

## Turn off IPv6
(Prevent ipv6:: binding)
```powershell
netsh int ipv6 isatap set state disabled #set-Net6to4Configuration
netsh interface ipv6 set global randomizeidentifiers=disabled
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" /v "DisabledComponents" /t REG_DWORD /d "ffffffff" /f
netsh interface ipv6 set teredo disabled
netsh interface ipv6 set 6to4 disabled
netsh interface ipv6 set isatap disabled
netsh interface ipv6 set interface "Loopback Pseudo-Interface 1" routerdiscovery=disabled
netsh interface ipv6 set interface "Loopback Pseudo-Interface 1" dadtransmits=0 store=active
netsh interface ipv6 set interface "Loopback Pseudo-Interface 1" routeradvertise=disabled
netsh advfirewall firewall add rule name="Block all IPv6 traffic" protocol=icmpv6:255,any dir=in action=block
netsh advfirewall firewall add rule name="Block all IPv6 traffic" protocol=icmpv6:255,any dir=out action=block
netsh advfirewall firewall add rule name="Block all IPv6 TCP/UDP traffic" protocol=TCPv6,UDPv6 dir=in action=block
netsh advfirewall firewall add rule name="Block all IPv6 TCP/UDP traffic" protocol=TCPv6,UDPv6 dir=out action=block
(edge=yes)
netsh advfirewall firewall add rule name="Block all IPv6 traffic" protocol=any dir=in action=block edge=yes profile=any interface=any
netsh advfirewall firewall add rule name="Block all IPv6 traffic" protocol=any dir=out action=block edge=yes profile=any interface=any
```

### Firewall default disallow
fw dis inbound out- allj, cast teredo v6 cortana mDNS Narrator network discovery remote assist start wi-fi direct windows calc windows search wireless display
```powershell
netsh advfirewall set allprofiles firewallpolicy blockinbound,allowoutbound
```
## remove Wireless Display

### disallow allow remote assist because it's laggy
```powershell
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Remote Assistance" /v "fAllowToGetHelp" /t REG_DWORD /d "0" /f
```

### SMB tuning
disable default admin and disk share server  
restrict access over anonymous connections  
prevent joining homegroup  
hide computer from browser list  
prevent network auto discovery  
hide entire network in network neighborhood

### Hardening and tuning
dis UAC pw  
act  
uninst add remove onedrive  
dis msteams startup

## Edge
edge start withot data  
  privacy statement reject all  
  multilingual text suggestions

## Regional
add kbd remove kbd  
add language basic typing ocr (not lang pack)  
first day of week

## Tooling
add latest ps  
win event colletor service?  
office  
winpe setup  
run apps in containers  
11 KB5010474  
11 KB2267602  
11 KB4052623  
upd ms store apps  
prevent system volume information folder creation  
storage spaces not working  
+zip fldr  
-wrk fldr  
+rmdks crp  

MS Store:  
- turn off autoplay videos

## Graphics
Turn off GUI fx  
google.com/search?q=UserPreferencesMask+value+in+the+Registry+to+enable+the+Classic+graphics+mode ?

## Eliminate Smooth scrolling
@echo off
reg add "HKEY_CURRENT_USER\Control Panel\Desktop" /v "SmoothScroll" /t REG_SZ /d "0" /f
reg add "HKEY_CURRENT_USER\Control Panel\Desktop" /v "MouseWheelRouting" /t REG_SZ /d "0" /f
echo Smooth scrolling has been disabled. Please restart your computer for the changes to take effect.
pause



