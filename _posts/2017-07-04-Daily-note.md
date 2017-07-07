
```xml
<disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/srv/storage/images/kerio_control.img'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </disk>
```

to

```xml
<disk type='file' device='disk'>
      <driver name='qemu' type='raw'/>
      <source file='/srv/storage/images/kerio_control.img'/>
      <target dev='hda' bus='ide'/>
      <address type='drive' controller='0' bus='0' unit='0'/>
</disk>
```

```xml
<interface type='bridge'>
  <mac address='52:54:00:11:c4:3b'/>
  <source bridge='br0'/>
  <target dev='vnet2'/>
  <model type='virtio'/>
  <alias name='net0'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```

改变网卡为

```xml
<interface type='bridge'>
      <mac address='52:54:00:11:c4:3b'/>
      <source bridge='br0'/>
      <target dev='vnet2'/>
      <model type='e1000'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```
-----------
生成ssh私钥登录：

```shell
ssh-keygen -t rsa
```

```
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/ml/.ssh/id_rsa): /Users/ml/.ssh/txech_vm
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/ml/.ssh/txech_vm.
Your public key has been saved in /Users/ml/.ssh/txech_vm.pub.
The key fingerprint is:
SHA256:48jFQSCj5YxPr94bglzsYSBWtzcSW8qPoRuvSZCHbx8 ml@mldeMacBook-Pro.local
The key's randomart image is:
+---[RSA 2048]----+
|   .+o.o.        |
|  .*oo*.         |
|..+ +B o.        |
|..o=..*...       |
| + +*...S        |
| .+=++ + .       |
|  o=+E+ .        |
|  o.+o..         |
|   o..o.         |
+----[SHA256]-----+
```

txech_vm.rsa (密码为空)

建立普通用户，并赋予root执行权限：  

```
useradd txtech -s /bin/bash -p 432980san=txt -m -G sudo
```


- sshd_config file

```shell
# Package generated configuration file
# See the sshd_config(5) manpage for details

# What ports, IPs and protocols we listen for
Port 22
# Use these options to restrict which interfaces/protocols sshd will bind to
#ListenAddress ::
#ListenAddress 0.0.0.0
Protocol 2
# HostKeys for protocol version 2
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_dsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
#HostKey /etc/ssh/ssh_host_ed25519_key
#Privilege Separation is turned on for security
UsePrivilegeSeparation yes

# Lifetime and size of ephemeral version 1 server key
KeyRegenerationInterval 3600
ServerKeyBits 1024

# Logging
LogLevel INFO

# Authentication:
LoginGraceTime 120
StrictModes yes

RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile  %h/.ssh/authorized_keys

# Don't read the user's ~/.rhosts and ~/.shosts files
IgnoreRhosts yes
# For this to work you will also need host keys in /etc/ssh_known_hosts
RhostsRSAAuthentication no
# similar for protocol version 2
HostbasedAuthentication no
# Uncomment if you don't trust ~/.ssh/known_hosts for RhostsRSAAuthentication
#IgnoreUserKnownHosts yes

# To enable empty passwords, change to yes (NOT RECOMMENDED)
PermitEmptyPasswords no

# Change to yes to enable challenge-response passwords (beware issues with
# some PAM modules and threads)
ChallengeResponseAuthentication no

# Change to no to disable tunnelled clear text passwords

# Kerberos options
#KerberosAuthentication no
#KerberosGetAFSToken no
#KerberosOrLocalPasswd yes
#KerberosTicketCleanup yes

# GSSAPI options
#GSSAPIAuthentication no
#GSSAPICleanupCredentials yes

X11Forwarding yes
X11DisplayOffset 10
PrintMotd no
PrintLastLog yes
TCPKeepAlive yes
#UseLogin no

#MaxStartups 10:30:60
#Banner /etc/issue.net

# Allow client to pass locale environment variables
AcceptEnv LANG LC_*

Subsystem sftp /usr/lib/openssh/sftp-server

# Set this to 'yes' to enable PAM authentication, account processing,
# and session processing. If this is enabled, PAM authentication will
# be allowed through the ChallengeResponseAuthentication and
# PAM authentication via ChallengeResponseAuthentication may bypass
# If you just want the PAM account and session checks to run without
# and ChallengeResponseAuthentication to 'no'.
UsePAM yes
Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,blowfish-cbc,aes128-cbc,3des-cbc,cast128-cbc,arcfour,aes192-cbc,aes256-cbc
UseDNS no
AddressFamily inet
PermitRootLogin yes 
SyslogFacility AUTHPRIV
PasswordAuthentication no 
```


---------------
mongodb 使用认证：  

```js
var opt = {
        user: 'mcc',
        pass: 'i+mcc=txtmcc',
        auth: {
            authdb: 'admin'
        }
    };


let connectDatabase = () => {
  mongoose.connect('mongodb://127.0.0.1/txt-lsp-agent-mcc-v2')
  let db = mongoose.connection
  db.on('error', console.error.bind(console, 'connection error:'))
  db.once('open', () => {
    console.log('Connected to Database.')
  })
}
```

------------
git reset [--soft | --mixed | --hard


上面常见三种类型



--mixed

会保留源码,只是将git commit和index 信息回退到了某个版本.

git reset 默认是 --mixed 模式 
git reset --mixed  等价于  git reset


--soft

保留源码,只回退到commit 信息到某个版本.不涉及index的回退,如果还需要提交,直接commit即可.



--hard

源码也会回退到某个版本,commit和index 都回回退到某个版本.(注意,这种方式是改变本地代码仓库源码)

当然有人在push代码以后,也使用 reset --hard <commit...> 回退代码到某个版本之前,但是这样会有一个问题,你线上的代码没有变,线上commit,index都没有变,当你把本地代码修改完提交的时候你会发现权是冲突.....

