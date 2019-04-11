## [原创] centos7 eduroam freeradius 安装记录

本文原创：**中国科学技术大学 张焕杰**

修改时间：2019.04.11

测试工具：https://eduroam.ustc.edu.cn/

[Protocol and Password Compatibility](http://deployingradius.com/documents/protocols/compatibility.html)

注意：本文仅仅设置用户认证部分，可以完成本地用户在其他地方上网的认证，未包含无线控制器有关设置(clients.conf增加即可)。

## 0. 基础

1. 有用户名和明文密码，以便radius使用（加密的密码未测试过)

2. 准备一个IP地址，将来用作radius服务器。

## 1. 联系eduroam认证中心

CERNET用户可以联系eduroam@CERNET http://www.eduroam.edu.cn
其他用户请联系 eduroam 中国无线网漫游交换中心 http://eduroam.cstnet.cn

填写申请表，发送邮件，等待对方的电话联系，并获取到相关的key.

## 2. 安装radius服务器

2.1 虚拟机安装CentOS 7

设置好主机名、IP地址等信息。

2.2 安装如下包

```
yum install -y mariadb mariadb-server freeradius-mysql freeradius freeradius-utils git
```

## 3. 启动mysqld
```
systemctl start mariadb   #启动mariadb
systemctl enable mariadb 
```

## 4. 初始化git，跟踪修改
```
cd /etc/raddb
git init 
git add *
git commit -m init
```

## 5. 创建数据库

参考 /etc/raddb/mods-config/sql/main/mysql/schema.sql setup.sql

必要时修改setup.sql中的密码 

```
echo "create database radius" | mysql
mysql radius < /etc/raddb/mods-config/sql/main/mysql/schema.sql
mysql radius < /etc/raddb/mods-config/sql/main/mysql/setup.sql
```

## 6. 设置包过滤规则

下面的2个IP是上游radius服务器（北大维护）IP地址。

如果自己的无线控制器AC也要连接radius服务器认证，需要把自己的无线控制器放开。

```
firewall-cmd --permanent --remove-service=ssh
firewall-cmd --permanent --zone=public --add-rich-rule="rule family="ipv4" source address="x.x.x.0/24" service name="ssh" accept"
firewall-cmd --permanent --zone=public --add-rich-rule="rule family="ipv4" source address="162.105.129.2" service name="radius" accept"
firewall-cmd --permanent --zone=public --add-rich-rule="rule family="ipv4" source address="121.194.2.97" service name="radius" accept
firewall-cmd --reload
```

## 7. 修改配置

7.1 ca有关配置

生成一个有效期足够长的证书，以免使用中更换。

```
cd /etc/raddb/certs
make destroycerts
vi ca.cnf

#修改default_days 7200
#修改[certificate_authority]

vi server.cnf

#修改default_days 7200
#修改[certificate_authority]

openssl dhparam -out dh 1024
make ca.pem
make ca.der
make server.pem

chgrp radiusd *
```

如果想查看生成的证书，可以执行
```
openssl x509 -in ca.pem -text | more
openssl x509 -in server.pem -text | more
```

7.2 clients.conf & proxy.conf

/etc/raddb/clients.conf增加如下:

注意：我这里仅仅作为漫游认证，并未增加本地无线控制器的信息。正常还需要增加无线控制器的信息

```
client flr.edu.cn {
        ipaddr          = x.x.x.x
        proto           = *
        secret          = yyyyyyyyyyyyyyyy 
        require_message_authenticator = no
}
client backup.flr.edu.cn {
        ipaddr          = x.x.x.x
        proto           = *
        secret          = yyyyyyyyyyyyyyyy 
        require_message_authenticator = no
}
```

/etc/raddb/proxy.conf 
```
realm fsyy.ustc.edu.cn {
}

home_server flr.edu.cn {
        type = auth
        ipaddr = x.x.x.x      
        port = 1812
        secret = ****************
        response_window = 20
        zombie_period = 40
        status_check = status-server
        check_interval = 120
        num_answers_to_alive = 3
        max_outstanding = 65536
}

home_server backup.flr.edu.cn {
        type = auth
        ipaddr = x.x.x.x      
        port = 1812
        secret = ****************
        response_window = 20
        zombie_period = 40
        status_check = status-server
        check_interval = 120
        num_answers_to_alive = 3
        max_outstanding = 65536
}

home_server_pool edu.cn-Failover {
        type = fail-over
        home_server = flr.edu.cn
        home_server = backup.flr.edu.cn
}

realm DEFAULT {
        auth_pool       = edu.cn-Failover
        nostrip
}
```
7.3 其他

/etc/raddb/mods-available/sql 
```
driver = "rlm_sql_mysql"

dialect = "mysql"
 
# Connection info:
#
server = "localhost"
port = 3306
login = "radius"
password = "radpass"
```
/mods-config/sql/main/mysql/queries.conf
```
sql_user_name = "%{%{Stripped-User-Name}:-%{%{User-Name}:-DEFAULT}}"
```

sites-available/default
将authorize中
```
-sql 
改为
sql
```

## 8. 启用sql模块

```
cd /etc/raddb/mods-enabled
ln -s ../mods-available/sql .
```


## 9. 测试用户
```
mysql radius
INSERT INTO radcheck VALUES (1,'test','Cleartext-Password',':=','test123');
```
建立一个用户test，密码test123

## 10. 测试
1. 在radius服务器上执行
radiusd -X

2. 访问 http://eduroam.ustc.edu.cn

输入 test@fsyy.ustc.edu.cn test 测试正常

如果测试通过，说明本地用户已经可以在其他地方登录。

## 11. 加入启动过程

```
systemctl start radiusd
systemctl enable radiusd
```
不过我测试重启时，可能因为mariadb启动有点慢，自动启动会出错。

***
欢迎 [加入我们整理资料](https://github.com/bg6cq/ITTS)
