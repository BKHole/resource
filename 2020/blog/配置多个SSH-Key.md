# 配置多平台SSH-Key

## 进入系统ssh目录

```
cd ~/.ssh
```

## 生成github密钥

```
ssh-keygen -t rsa -C "your_mail@example.com" -f github_rsa
```

回车确认，直到完成。其他账号操作类似

## 生成gitlab密钥

```
ssh-keygen -t rsa -C "your_mail@example.com" -f gitlab_rsa
```

## 生成gitee密钥

```
ssh-keygen -t rsa -C "your_mail@example.com" -f gitee_rsa
```

## 查看公钥

```
// 查看github公钥
cat github_rsa.pub
// 查看gitlab公钥
cat gitlab_rsa.pub
// 查看gitee公钥
cat github_rsa.pub
```

将得到的公钥分别复制到对应账号SSH设置中

## 添加config文件（若无）

```
touch config //mac linux，windows推荐使用git bash
```

## 为config文件添加如下内容

```
# github.com
Host github.com
HostName github.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/github_rsa

# gitlab.company.com
Host gitlab.company.com # 此域名端口默认22，如果为122端口，需设置gitlab.company.com:122
HostName gitlab.company.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/gitlab_rsa

# gitee.com
Host gitee.com
HostName gitee.com
PreferredAuthentications publickey
IdentityFile ~/.ssh/gitee_rsa
```

## 测试

```
// test github ssh-key
ssh -T git@github.com
// test gitlab ssh-key
ssh -T git@gitlab.company.com
// test gitee ssh-key
ssh -T git@gitee.com
```

## 成功

输出内容如下时，配置成功

> Hi YK-Unit! You've successfully authenticated, but GitHub does not provide shell access.

此输出内容为测试github ssh-key，其他账号测试内容类似

## 提醒

Windows系统命令终端推荐使用git bash。