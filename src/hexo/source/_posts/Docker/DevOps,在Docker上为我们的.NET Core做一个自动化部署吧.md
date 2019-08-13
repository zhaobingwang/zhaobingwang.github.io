---
title: 在Docker上为我们的.NET Core程序做一个自动化部署吧
categories: Docker
comments: true
date: 2019/05/21 22:54:17
updated: 2019/05/21 22:54:17
tags:
    - Docker
    - 自动化
    - 软件工程
---

# 环境
>CentOS : 7.6.1810 (Core) 
Jenkins : 2.177
Docker : 18.09。6
.NET Core : 2.2

# 流程
当吧GitHub中的`dev`分支合并到`master`后，jenkins将被触发，自动拉取代码到所在服务器上，然后通过下文中的`shell`脚本进行自动构建部署。

# jenkins配置
基本配置可参见之前的文章：[如何：将Github项目持续集成部署到Nuget](https://blog.csdn.net/zhaobw831/article/details/85342075)。
本文配置git为代码托管平台，指定构建分支为`master`，使用GitHub触发器，具体配置可参见上文中所指出的文章。

# docker脚本
```
FROM microsoft/dotnet:2.2-aspnetcore-runtime
WORKDIR /app
COPY . .
EXPOSE 80
ENTRYPOINT ["dotnet", "CSTree.Web.dll"]

```

# shell脚本
脚本说明：
>此脚本用于在GitHub构建的触发后，自动编译`.NET Core`项目，然后打包成`docker`镜像，创建容器（将容器的80端口绑定到主机的8801端口），最后通过`Nginx`做反向代理。

*`-v /etc/localtime:/etc/localtime`用于挂载主机localtime到容器内以保持容器与主机时间同步。（docker默认使用UTC时间，与北京时间相差8小时）*
```shell
# 编译项目
#echo "begin www.cstree.cn build..."
dotnet build cstree.sln -c Release
#echo "build www.cstree.cn success"

echo "begin www.cstree.cn pack..."
# 打包项目 cstree 并输出到临时存放目录
echo "pack www.cstree.cn ..."
sudo rm -rf /publish/docker/temp.www.cstree.cn
sudo mkdir /publish/docker/temp.www.cstree.cn
sudo dotnet publish src/CSTree.Web/CSTree.Web.csproj -c Release -o /publish/docker/temp.www.cstree.cn
sudo rm -f /publish/docker/temp.www.cstree.cn/appsettings.json
sudo rm -f /publish/docker/temp.www.cstree.cn/appsettings.Development.json
sudo rm -f /publish/docker/temp.www.cstree.cn/nlog.config
sudo cp -Rf /publish/docker/temp.www.cstree.cn/* /publish/docker/www.cstree.cn
echo "pack www.cstree.cn success"

echo "begin build docker..."
cd /publish/docker/www.cstree.cn
sudo docker stop cstree01 || true && sudo docker rm cstree01 || true

result=$( sudo docker images -q cstree:latest )

if [[ -n "$result" ]]; then
  echo "Container exists"
  sudo docker rmi cstree:latest
else
  echo "No such container"
fi

sudo docker build -t cstree:latest .
sudo docker run -v /etc/localtime:/etc/localtime --name=cstree01 -p 8801:80 -d  cstree:latest
echo "build docker success..."
```

Nginx配置
```txt
server {
    listen       80 default_server;
    listen       [::]:80 default_server;
    server_name  _;
    root         /usr/share/nginx/html;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
		proxy_pass http://127.0.0.1:8801/;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection keep-alive;
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
    }
    error_page 404 /404.html;
        location = /40x.html {
    }
    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
```
