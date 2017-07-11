
----------
拉取mongo镜像：

启动容器：  


初始化admin：  

```sh
docker run --name txtMongo -p 270127:27017 -v /var/mongo_backup:/data/db -d 71c --auth
```

```js
use admin
db.createUser(
  {
    user: "txtAdmin",
    pwd: "PivosTxtech07031036",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
```

（1）重新连接MongoDB数据库

退出容器，重新用下面命令进入容器即可：

docker exec -it mongodb_mongo_1 mongo admin

#----------
# Result：
#----------
MongoDB shell version: 3.2.12
connecting to: admin

（2）授权登录admin

```js
db.auth('txtAdmin', 'PivosTxtech07031036')
```

（3）创建访问指定数据库的用户

*Step1: switch to the specified database*:

```js
use octblog
```

*Step2: create a user*  

```js
db.createUser(
  {
    user: "txt",
    pwd: "itxtmcc",
    roles: [ { role: "readWrite", db: "txt-lsp-agent-mcc" },
             { role: "readWrite", db: "txt-lsp-agent-mcc-v2" } ]
  }
)
```

```
mongodump --collection employee --db mongodevdb --username mongodevdb --password YourSecretPwd --out /dbbackup
```

Failed: error connecting to db server: server returned error on SASL authentication step: Authentication failed. 

solve: 

--authenticationDatabase admin


使用指定用户名和备份路径：  

```
docker exec -it d08 mongodump --db txt-lsp-agent-mcc --username txtAdmin --password PivosTxtech07031036 --out /data/db/backup --authenticationDatabase admin
```

## 将容器中的mongo数据库备份出来

```sh
#!/bin/bash

dbNames="txt-lsp-agent-mcc 
txt-lsp-agent-api
txt-lsp-agent-mcc-v2
txt-lsp-agent-weixin"

for i in $dbNames
do
  docker exec -it d08 mongodump --db $i --username txtAdmin --password PivosTxtech07031036 --out /data/db/backup --authenticationDatabase admin
done

echo; echo "has backup mongodb database in container to /var/mongo_backup/"
exit 0
```
