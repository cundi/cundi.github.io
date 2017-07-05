
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


