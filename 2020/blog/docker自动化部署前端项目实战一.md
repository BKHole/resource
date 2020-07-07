# docker自动化部署前端项目实战一

本文适用于个人项目，如博客、静态文档，不涉及后台数据交互，以部署文档为例。

## 思路

利用服务器node脚本，监听github仓库webhook push事件触发post请求，自动拉取最新代码，再用docker接管项目编译、部署。

## 环境

本文使用云服务器搭建，环境版本：

- OS：CentOS Linux release 8.2.2004 
- docker：19.03.12
- node：14.5.0
- git：2.18.4

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
不出意外，出现==hello world==，docker安装成功

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

### 创建webhook

进入github项目仓库，按下图顺序操作

![](https://cdn.jsdelivr.net/gh/BKHole/resource@latest/2020/image/docksify.png)

验证webhook配置成功，点击红色感叹号右侧内容，出现如下请求信息
![](https://cdn.jsdelivr.net/gh/BKHole/resource@latest/2020/image/doc2.png)

## docker部署

### 创建Dockfile

在这里，将拉取的项目存放在app目录下，Dockerfile内容如下，放到服务器根目录（/root/Dockerfile）

```
FROM nginx
COPY /app/docsify /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 创建 http 服务器

创建index.js，放到服务器根目录（/root/index.js）

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
        
        // 创建 docker 镜像
        execSync(`docker build . -t ${data.repository.name}-image:latest`, {
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

解析，

**创建docker镜像**

```
docker build . -t docsify-image:latest 
```

- build：创建 docker 镜像
- .：使用当前目录下的 Dockerfile 文件，这里在根目录（/root/)执行
- -t：使用 tag 标记版本
- docsify-image:latest：创建名为 docsify-image 的镜像，并标记为 latest（最新）版本

**创建docker容器**

```
docker run -d -p 88:80 --name docsify-container docsify-image:latest
```

- run：创建并运行 docker 容器
- -d： 后台运行容器
88:80：将当前服务器的 88 端口（冒号前的 88），映射到容器的 80 端口（冒号后的 80）
- --name：给容器命名，便于之后定位容器
- docsify-image:latest：基于 docsify-image 最新版本的镜像创建容器

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

## 域名访问

在拥有域名的前提下，优先使用域名访问。为什么？域名当然比IP+端口号好记，且美观。

这里为了方便控制，使用[nginx-proxy](https://github.com/nginx-proxy/nginx-proxy)镜像来操作，如下操作docker会自动去镜像仓库拉取，建议服务器80端口给nginx使用，方便以后增加域名和访问端口监听。

```
docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro jwilder/nginx-proxy
```

然后绑定域名到新建容器，这里使用我的二级域名。

```
docker run -e VIRTUAL_HOST=libotao.nofoo.cn docsify-image
```

这里创建容器省略了容器名，

- -e：设置环境变量

这时，域名已经配置好了，访问[http://libotao.nofoo.cn](http://libotao.nofoo.cn)可以看到效果。

前面每次提交内容到github，服务器都会重新拉取最新代码，新建image，销毁container，新建container，访问内容才会更新，为了实现自动化，需要改造一下上面的index.js脚本,

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
        // 项目放在服务器app目录下 
        const projectDir = path.resolve(`./app/${data.repository.name}`)
        deleteFolderRecursive(projectDir)

        // 拉取仓库最新代码
        execSync(`git clone https://github.com/BKHole/${data.repository.name}.git ${projectDir}`, {
            stdio: 'inherit',
        })
        
        // 创建 docker 镜像
        execSync(`docker build . -t ${data.repository.name}-image:latest `, {
            stdio: 'inherit',
        })

        // 销毁 docker 容器
        execSync(`docker ps -a -f "name=^${data.repository.name}-container" --format="{{.Names}}" | xargs -r docker stop | xargs -r docker rm`, {
            stdio: 'inherit',
        })

        // 创建 docker 容器
        execSync(`docker run --name ${data.repository.name}-container -e VIRTUAL_HOST=libotao.nofoo.cn  ${data.repository.name}-image:latest`, {
            stdio: 'inherit',
        })

        console.log('deploy success')
        res.end('ok')
    }
}).listen(3000, () => {
    console.log('server is ready')
})

```

修改后覆盖之前存放的index.js，然后重启脚本。

```
pm2 restart index.js
```

配置完成后，以后每次提交github，都会自动更新，访问域名就会看到最新的内容。

![](https://cdn.jsdelivr.net/gh/BKHole/resource@latest/2020/image/20200707160021.png)

> note：本文中使用的端口号都需要在云服务器平台创建安全组策略，放开端口

## 参考

[docker + webhook 从零实现前端自动化部署](https://juejin.im/post/5ef4c7eff265da230b52dfc5)