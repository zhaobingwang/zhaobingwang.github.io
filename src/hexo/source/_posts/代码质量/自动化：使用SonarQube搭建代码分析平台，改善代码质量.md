---
title: 使用SonarQube搭建代码质量管理平台
categories: 代码质量管理
comments: true
date: 2019/08/04 00:52:46
updated: 2019/08/04 00:52:46
tags:
    - SonarQube
    - 自动化
    - 代码质量管理
    - 软件工程
---

# SonarQube简介
[SonarQube](https://www.sonarqube.org/)是一种自动代码审查工具,用于检测代码中的`错误`，`漏洞`及`代码气味`([
code smells](https://en.wikipedia.org/wiki/Code_smell)，这个翻译有点奇怪?)，它可以与现有工作流集成（比如jenkins）,以便在项目分支和拉取请求中实现持续的代码检查。

# 开始搭建吧
*本文使用docker搭建，如果直接在宿主机上搭建，需要`jdk`环境，*
- 创建一个PostgreSQL
```shell
docker run --name pg.sq -e POSTGRES_USER=sonar -e POSTGRES_PASSWORD=sonar -d postgres
```
- 创建SonarQube平台
```shell
docker run --name sq --link pg.sq -e sonar.jdbc.username=sonar -e sonar.jdbc.password=sonar -e sonar.jdbc.url=jdbc:postgresql://pg.sq:5432/sonar -p 9000:9000 -d sonarqube
```
待容器安装完成后，本机打开：http://127.0.0.1:9000/访问SonarQube服务
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190803235901613.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9idzgzMQ==,size_16,color_FFFFFF,t_70)
可以使用默认账号/密码登录：admin/admin。
然后我们可以在`Projects`功能菜单下创建一个项目，本文创建的项目为`SonarQubeDemo`：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190803235953895.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9idzgzMQ==,size_16,color_FFFFFF,t_70)
# 使用SonarScanner
## 下载SonarScanner
我们还需要下载 [SonarScanner for MSBuild](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/) 

本文使用的是.NET Core/C#代码，所以下载的是.NET平台下`SonarScanner`

## 配置SonarQube.Analysis.xml
添加以下配置项
```xml
  <Property Name="sonar.host.url">http://localhost:9000</Property>
  <Property Name="sonar.login">admin</Property>
  <Property Name="sonar.password">admin</Property>
```
然后执行以下命令
```shell
dotnet <path to SonarScanner.MSBuild.dll> begin /k:"project-key" 
dotnet build <path to solution.sln>
dotnet <path to SonarScanner.MSBuild.dll> end 
```
本示例源码为一个.NET Core/C# ConsoleApp，其中代码为：
```csharp
using System;
namespace SonarQubeDemo
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World!");
        }
    }
}

```
查看SonarQube
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190804002124434.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9idzgzMQ==,size_16,color_FFFFFF,t_70)

# 我们来制造一些有问题的代码
我们来创建一个空引用的代码
```csharp
using System;

namespace SonarQubeDemo
{
    class Program
    {
        static void Main(string[] args)
        {
            User user = null;
            Console.WriteLine(user.ID);
        }
    }
    class User
    {
        public int ID { get; set; }
    }
}

```
在执行完上文中`SonarScanner`的命令后：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190804004636174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9idzgzMQ==,size_16,color_FFFFFF,t_70)
产生了一个bug，点击bug查看详细信息：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190804004809650.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9idzgzMQ==,size_16,color_FFFFFF,t_70)
至此，SonarQube的开篇就结束了。
