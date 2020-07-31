# docker安装mysql 5.7

## 拉取镜像

```
docker pull mysql:5.7
```

## 新建mysql容器

```
docker run -itd --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7
```

设置root账户密码为123456，尽量设置复杂密码

## 配置远程访问

```
docker exec -it mysql bash 
```

- 进入 mysql 容器

```
mysql -uroot -p 
```

- 登录 mysql，执行后输入密码进入 mysql

```
GRANT ALL PRIVILEGES ON *.* TO 'username'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
```

- {username}：远程访问登录的用户名，不建议用 root;
- {password}：远程访问的登录密码;
- '%'：代表所有IP，如果可以尽量设置指定 IP 或 IP 段

```
FLUSH PRIVILEGES;
```

- 刷新权限

> note：mysql命令都需要 ; 结尾

## 测试

```
# 本机测试
mysql -h localhost -uroot -p

# 远程测试
mysql -h ip -u username -p password
# ip:使用pc公网ip
# username:上文创建账户
# password:上文创建账户密码
```

## 退出

```
exit;
```


> note：阿里/腾讯服务器安全组放开3306端口