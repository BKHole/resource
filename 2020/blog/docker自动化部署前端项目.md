# docker自动化部署前端项目

本文适用于个人项目，如博客、静态文档。

## 原理

利用服务器node脚本，监听github仓库webhook push事件触发post请求，自动拉取最新代码，再用docker接管项目编译、部署。

## 安装环境

云服务器如果没有安装以下环境，需要安装。

- docker
- node
- pm2
- git

### docker

```
# Step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils
# Step 2: 添加软件源信息，使用阿里云镜像
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 安装 docker-ce
sudo yum install docker-ce docker-ce-cli containerd.io
# Step 4: 开启 docker服务
sudo systemctl start docker
# Step 5: 运行 hello-world 项目
sudo docker run hello-world
```

### git

从代码仓库拉取最新代码
```
yum install git
```

### node

创建js脚本。使用nvm管理node版本，先安装nvm

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
```

将nvm设置环境变量

```
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion

```

通过 nvm 安装最新版 node

```
nvm install node
```

### PM2

安装pm2，服务器后台运行js脚本

```
npm i pm2 -g
```

## webhook

github 的 webhook 会在当前仓库触发某些事件时，发送一个 post 形式的 http 请求

## docker部署

### 创建Dockfile

Dockerfile

```
FROM nginx
COPY /app/docsify /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 创建.dockerignore

.dockerignore

```
# .dockerignore
node_modules
```

### 创建 http 服务器

创建index.js，放到服务器根目录（/root/index.js)

```
const http = require("http")
const { execSync } = require("child_process")
const fs = require("fs")
const path = require("path")

// 递归删除目录
function deleteFolderRecursive(path) {
    if (fs.existsSync(path)) {
        fs.readdirSync(path).forEach(function (file) {
            const curPath = path + "/" + file;
            if (fs.statSync(curPath).isDirectory()) { // recurse
                deleteFolderRecursive(curPath);
            } else { // delete file
                fs.unlinkSync(curPath);
            }
        });
        fs.rmdirSync(path);
    }
}

const resolvePost = req =>
    new Promise(resolve => {
        let chunk = "";
        req.on("data", data => {
            chunk += data;
        });
        req.on("end", () => {
            resolve(JSON.parse(chunk));
        });
    });

http.createServer(async (req, res) => {
    console.log('receive request')
    console.log(req.url)
    if (req.method === 'POST' && req.url === '/') {
        const data = await resolvePost(req);
        const projectDir = path.resolve(`./app/${data.repository.name}`)
        deleteFolderRecursive(projectDir)

        // 拉取仓库最新代码
        execSync(`git clone https://github.com/BKHole/${data.repository.name}.git ${projectDir}`, {
            stdio: 'inherit',
        })
        console.log('projectDir=', projectDir);
        // 复制 Dockerfile 到项目目录
        // fs.copyFileSync(path.resolve(`./Dockerfile`), path.resolve(projectDir, './Dockerfile'))

        // 复制 .dockerignore 到项目目录
        // fs.copyFileSync(path.resolve(__dirname, `./.dockerignore`), path.resolve(projectDir, './.dockerignore'))
        console.log('__dirname',__dirname);
        // 创建 docker 镜像
        execSync(`docker build . -t ${data.repository.name}-image:latest `, {
            stdio: 'inherit',
        })

        // 销毁 docker 容器
        execSync(`docker ps -a -f "name=^${data.repository.name}-container" --format="{{.Names}}" | xargs -r docker stop | xargs -r docker rm`, {
            stdio: 'inherit',
        })

        // 创建 docker 容器
        execSync(`docker run -d -p 88:80 --name ${data.repository.name}-container  ${data.repository.name}-image:latest`, {
            stdio: 'inherit',
        })
       
        console.log('deploy success')
        res.end('ok')
    }
}).listen(3000, () => {
    console.log('server is ready')
})

```

### 运行node脚本


```
pm2 start index.js
```

### test

服务器运行pm2 logs查看index.js打印日志

```
pm2 logs
```

本地仓库修改文件内容，提交远程仓库，日志出现deploy success,自动化部署成功。
![img](https://cdn.jsdelivr.net/gh/BKHole/resource@latest/2020/image/1730312b116d8124)

访问[http://47.108.82.91:88](http://47.108.82.91:88)，记得在云服务器上放开访问端口号