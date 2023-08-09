通过docker容器部署Dgraph

# 一、安装docker

1、使用 brew 安装

```
brew install --cask --appdir=/Applications docker
```

2、手动下载安装

```
https://www.docker.com/get-started/
```

# 二、获取 dgraph 镜像

如果镜像下载慢可以配置镜像，网易镜像如下：

```
http://hub-mirror.c.163.com
```

1、获取dgraph核心组件镜像

```
docker pull dgraph/standalone
```

2、获取dgraph本地可视化终端镜像（没有该镜像无法访问本地UI）

```
docker pull dgraph/ratel
```

# 三、启动docker容器

```
docker run -it -p "8080:8080" -p "9080:9080" -v ~/dgraph:/dgraph "dgraph/standalone:latest" #启动核心组件
```

```
docker run -it -p "8000:8000" "dgraph/ratel:latest" #启动可视化终端
```

# 四、访问可视化终端

访问地址：

⚠️8000端口为启动docker可视化终端容器时映射的端口

```
http://localhost:8000
```

