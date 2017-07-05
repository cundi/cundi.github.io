
# Bugzilla
准备数据库，and expose docker port internally:  

```shell
sudo docker run -d --name bugzilla_db  -e MYSQL_ROOT_PASSWORD=root -p 127.0.0.1:3307:3306 411
```

进入docker建立数据库

从docker hub上拉取镜像，并容器互联：  

```
sudo docker run -d -p 9081:80 --name bugzilla -v /tmp:/etc/msmtprc:ro --link bugzilla_db:db -e MYSQL_HOST=172.17.0.9 -e MYSQL_PORT=3306 -e MYSQL_DB=bugzilla -e MYSQL_USEER=root -e MYSQL_PWD=root achild/bugzilla
```

不过，启动时指定的参数好像没有写入到配置文件，遂进入容器改动配置：  


bugzilla的`/var/www/bugzilla-4.4.8/localconfig`.  

```shell
# If you do not have access to the group your scripts will run under,
# set this to "". If you do set this to "", then your Bugzilla installation
# will be _VERY_ insecure, because some files will be world readable/writable,
# and so anyone who can get local access to your machine can do whatever they
# want. You should only have this set to "" if this is a testing installation
# and you cannot set this up any other way. YOU HAVE BEEN WARNED!
#
# If you set this to anything other than "", you will need to run checksetup.pl
# as root or as a user who is a member of the specified group.
$webservergroup = 'www-data';

# Set this to 1 if Bugzilla runs in an Apache SuexecUserGroup environment.
#
# If your web server runs control panel software (cPanel, Plesk or similar),
# or if your Bugzilla is to run in a shared hosting environment, then you are
# almost certainly in an Apache SuexecUserGroup environment.
#
# If this is a Windows box, ignore this setting, as it does nothing.
#
# If set to 0, checksetup.pl will set file permissions appropriately for
# a normal webserver environment.
#
# If set to 1, checksetup.pl will set file permissions so that Bugzilla
# works in a SuexecUserGroup environment.
$use_suexec = 0;

# What SQL database to use. Default is mysql. List of supported databases
# can be obtained by listing Bugzilla/DB directory - every module corresponds
# to one supported database and the name of the module (before ".pm")
# corresponds to a valid value for this variable.
$db_driver = 'mysql';

# The DNS name or IP address of the host that the database server runs on.
$db_host = '172.17.0.9';

# The name of the database. For Oracle, this is the database's SID. For
# SQLite, this is a name (or path) for the DB file.
$db_name = 'bugzilla';

# Who we connect to the database as.
$db_user = 'root';

# Enter your database password here. It's normally advisable to specify
# a password for your bugzilla database user.
# If you use apostrophe (') or a backslash (\) in your password, you'll
# need to escape it by preceding it with a '\' character. (\') or (\)
# (It is far simpler to just not use those characters.)
$db_pass = 'root';

# Sometimes the database server is running on a non-standard port. If that's
# the case for your database server, set this to the port number that your
# database server is running on. Setting this to 0 means "use the default
# port for my database server."
$db_port = 3306;

# MySQL Only: Enter a path to the unix socket for MySQL. If this is
# blank, then MySQL's compiled-in default will be used. You probably
# want that.
$db_sock = '';
```

然后执行perl脚本，即可完成数据库表创建：  

```
./checksetup.pl
```

配置SMTP邮件（smtp.exmail.qq.com）通知时要注意，不勾选邮件队列（use_mailer_queue：off）和smtp_ssl（off），否则收不到邮件。  

------------

# WebvirtMgr

Ubutut物理机配置网桥：  

```shell
 cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp1s0
iface enp1s0 inet manual

# br0 for kvm
auto br0
iface br0 inet dhcp
        hwaddress ether 00:11:32:4F:B2:54
        bridge_ports enp1s0
        bridge_maxwait 0
        bridge_fd 0
        bridge_stp off
```

kvm的xml描述文件：  

```xml
<domain type='kvm' id='36'>
  <name>DSM5.2</name>
  <uuid>322f8c59-33a6-4a80-72da-60b0fd2cc73a</uuid>
  <description>None</description>
  <memory unit='KiB'>2097152</memory>
  <currentMemory unit='KiB'>2097152</currentMemory>
  <vcpu placement='static'>1</vcpu>
  <resource>
    <partition>/machine</partition>
  </resource>
  <os>
    <type arch='x86_64' machine='pc-i440fx-xenial'>hvm</type>
    <boot dev='cdrom'/>
    <boot dev='hd'/>
    <bootmenu enable='yes'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <pae/>
  </features>
  <cpu mode='host-model'>
    <model fallback='allow'/>
  </cpu>
  <clock offset='utc'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <devices>
    <emulator>/usr/bin/kvm-spice</emulator>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/www/webvirtmgr/images/XPEnoboot_DS3615xs_5.2-5967.1.iso'/>
      <backingStore/>
      <target dev='hda' bus='ide'/>
      <readonly/>
      <alias name='ide0-1-0'/>
      <address type='drive' controller='0' bus='1' target='0' unit='0'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/idsm.img'/>
      <backingStore/>
      <target dev='hdb' bus='ide'/>
      <alias name='ide0-0-1'/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <controller type='usb' index='0'>
      <alias name='usb'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'>
      <alias name='pci.0'/>
    </controller>
    <controller type='ide' index='0'>
      <alias name='ide'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <controller type='sata' index='0'>
      <alias name='sata0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </controller>
    <interface type='bridge'>
      <mac address='52:54:00:d3:95:90'/>
      <source network='netbr0' bridge='br0'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <serial type='pty'>
      <source path='/dev/pts/1'/>
      <target port='0'/>
      <alias name='serial0'/>
    </serial>
    <console type='pty' tty='/dev/pts/1'>
      <source path='/dev/pts/1'/>
      <target type='serial' port='0'/>
      <alias name='serial0'/>
    </console>
    <input type='tablet' bus='usb'>
      <alias name='input0'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='vnc' port='5900' autoport='yes' listen='0.0.0.0' passwd='ZzwZ5tWjoTt2'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='cirrus' vram='16384' heads='1'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
    </memballoon>
  </devices>
  <seclabel type='dynamic' model='apparmor' relabel='yes'>
    <label>libvirt-322f8c59-33a6-4a80-72da-60b0fd2cc73a</label>
    <imagelabel>libvirt-322f8c59-33a6-4a80-72da-60b0fd2cc73a</imagelabel>
  </seclabel>
</domain>
```

