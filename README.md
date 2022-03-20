# docker配置记录

为了天池的复赛现学的docker，做了一点踩坑记录，感觉docker里面的坑很多，以后还要慢慢学

由于服务器的网络实在是太差了，pull一个最基本的镜像都会失败，所以我在自己的电脑上安装了docker进行测试。（windows 10 家庭版）

1. 启动Hyper-V，由于我的电脑里不知道为什么找不到这个选项，所以需要创建一个.cmd文件，内容为：

   ```txt
   pushd "%~dp0"
    
   dir /b %SystemRoot%\servicing\Packages\*Hyper-V*.mum >hyper-v.txt
    
   for /f %%i in ('findstr /i . hyper-v.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\servicing\Packages\%%i"
    
   del hyper-v.txt
    
   Dism /online /enable-feature /featurename:Microsoft-Hyper-V-All /LimitAccess /ALL
   ```

   然后以管理员身份运行，运行完之后会让你重启一下，然后就能打开Hyper-V了。

2. 然后启动docker，如果是第一次用docker的话需要去注册一下账号然后登录，这步比较简单，有手就行。

3. 简单测试一下docker是不是能正常运行，`docker run hello-world`

4. 喝口水让电脑歇会儿

## ubuntu镜像

### 方法一：

打算以Ubuntu镜像作为base，然后结合conda pack打包我服务器上的运行环境

1. 安装Ubuntu镜像，可以去hub.docker.com找合适的版本，`docker pull ubuntu:20.04`

2. `docker run -it ubuntu:20.04`查看这个镜像是否可以运行。（cmd界面快捷键Ctrl + A，Ctrl＋D可以退出这个容器）

3. 打包我在服务器上的anaconda环境`conda pack mmcv_lyx -o tianchi.tar.gz --ignore-editable-packages`，下载这个压缩包到自己的电脑上

4. 编写`Dockerfile`文件：

   ```dockerfile
   FROM ubuntu:20.04
   MAINTAINER liyingxuan 913797866@qq.com
   
   RUN mkdir /py37/
   
   ADD tianchi.tar.gz /py37/
   
   ADD . /
   WORKDIR /
   
   CMD ["sh", "run.sh"]
   ```

   其中`From`表示在哪个image的基础上build这个image，`MAINTAINER`表示这个image创建者的信息，后面是一些命令，这个是比较基础的，一般会把要运行的命令写在`run.sh`脚本里面。

   其中`run.sh`里面是我的测试脚本

   ```bash
   /py37/bin/python hello_world.py
   ```

   but这个方法真的太太暴力了，而且有些库比如opencv这样弄还是会报错的，而且整个镜像非常大，还是推荐方法2

3. 编译docker环境（一定不要忘记了最后的点！泪目了，，，）

   ```bash
   build -t tianchi .
   ```

6. 运行

   ```dockerfile
   docker run tianchi
   ```

### 方法二：

编写`Dockerfile`文件：

```dockerfile
FROM ubuntu:20.04
MAINTAINER liyingxuan 913797866@qq.com

RUN  apt-get update
RUN  apt-get upgrade -y

# Install python3
RUN  apt-get install -y python3

ADD . /
WORKDIR /

# Install pip
RUN apt-get install -y python3-pip
RUN pip install --upgrade pip
RUN chmod +x requirements.txt
RUN pip install -r requirements.txt

CMD ["sh", "run.sh"]
```

将需要安装的库写在`requirements.txt`中，其余操作和方法1一样。

##### 一些调试环境的技巧

先不要写`CMD`的命令，先弄好环境的配置，然后进入这个容器，直接python调试，调试通过以后，再加上这个命令重新编译

## 删除一些没用的images

1. 删除正在使用该image的container，`docker ps -a`可以查看containers和对应的images

   ```bash
   docker rm <containerID>
   ```

   懒人方法，一条命令删除所有containers，这个命令要用powershell执行，用cmd会报错（试了好久才发现是这个问题，麻了）

   ```bash
   docker rm $(docker ps -a -q)
   ```

2. 删除image，`docker images`查看images和对应的ID 

   ```bash
   docker rmi <imageID>
   ```

   也可以批量删除所有没有tag的image

   ```bash
   docker images|grep none|awk '{print $3}'|xargs docker rmi
   ```

## 常见问题

1. 随便搞了几个镜像我硬盘满了。。。。

   ```bash
   docker container prune
   docker volume prune
   docker builder prune
   docker system prune
   ```

   如果还是占用太多的话，打开docker desktop -> debug(右上角一个小虫的标志)->clean/Purge data，选中所有，都删掉（这样image应该不能用了，就是删得真的很干净） 

   也可以从C盘搬到其他盘去，但是具体太麻烦了先将就着用

2. 不能解决的error

   重启docker就完事了，重启docker不行的话就重启电脑吧。



## 参考教程：

- docker安装和简单使用：https://blog.csdn.net/hunan961/article/details/79484098
- 清理垃圾镜像：http://www.dockone.io/article/3056

