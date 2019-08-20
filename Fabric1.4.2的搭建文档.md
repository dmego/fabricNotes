# 基于CentOS 7 搭建 Fabric1.4.2 区块链环境

## 1. 下载CentOS 7 镜像

- 下载地址：<http://isoredirect.centos.org/centos/7/isos/x86_64/>
- 网易源镜像地址：<http://mirrors.163.com/centos/7.6.1810/isos/x86_64/>

## 2. VMware 15 安装 CentOS Minimal

- 安装完成之后如果`ifconfig`命令无法使用，使用命令`yum -y install net-tools` 安装即可

### CentOS 换 yum 源

- 首先进入`/etc/yum.repos.d/`目录下，新建一个repo_bak目录，用于保存系统中原来的repo文件
  
    ```bash
    cd /etc/yum.repos.d/
    mkdir repo_bak
    mv *.repo repo_bak/
    ```

  - 在CentOS中配置使用网易和阿里的开源镜像
  
    ```bash
    curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    ```
  
  - 清除系统yum缓存并生成新的yum缓存
  
    ```bash
     yum clean all     # 清除系统所有的yum缓存
     yum makecache     # 生成yum缓存
    ```

  - 安装`git`、`pip`、`vim`、`wget`等
  
    ```bash
    yum install git
    yum install vim
    yum install wget
    yum -y install epel-release
    yum install python-pip
    pip install --upgrade pip
    ```
  
## 3. 安装Go语言环境

- 下载最新的GO安装包，具体的最新版本号可以从[Golang官网](https://golang.org/)上查看,如果被墙，可以去[Go语言中文网](https://studygolang.com/dl)下载

  ```bash
  cd ~
  wget https://studygolang.com/dl/golang/go1.11.5.linux-amd64.tar.gz
  tar -C /usr/local -xzf go1.11.5.linux-amd64.tar.gz #解压安装包到/usr/local目录下
  ```

- 打开`/etc/profile`文件

  ```bash
  vim /etc/profile
  ```

- 在最后添加如下四个环境变量

  ```bash
  export  PATH=$PATH:/usr/local/go/bin
  export  GOROOT=/usr/local/go
  export  GOPATH=/opt/gopath
  export  PATH=$PATH:/opt/gopath/bin
  ```

- `source /etc/profile` 使环境变量生效, 用`go version`验证一下`Go`是否安装成功

- 创建`Go`工作目录

  ``` bash
  cd ~
  mkdir -p /opt/gopath/src/github.com/hyperledger/fabric
  ```

## 4. Docker 安装(*号步骤可以不用操作)

- 卸载旧版本`Docker`

  ```bash
  sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
  ```

- 安装`Docker CE`

  - 安装所需的包。`yum-utils`提供了`yum-config-manager` 效用，并`device-mapper-persistent-data`和`lvm2`由需要`devicemapper`存储驱动程序

    ```bash
    sudo yum install -y yum-utils \
    device-mapper-persistent-data \
    lvm2
    ```

  - 使用以下命令设置**稳定**存储库

    ```bash
     sudo yum-config-manager \
        --add-repo \
        https://download.docker.com/linux/centos/docker-ce.repo
    ```

  - 安装最新版本的Docker CE和containerd
  
    ```bash
    sudo yum install docker-ce docker-ce-cli containerd.io
    ```
  
  - 启动`Docker`
  
    ```bash
    sudo systemctl start docker
    ```
  
  - (*)通过运行`hello-world` 映像验证是否正确安装了`Docker CE`

    ```bash
    docker run hello-world
    ```

  - 或者运行以下命令查看`Docker`版本信息
  
    ```bash
    docker version
    ```
  
  - 添加阿里云的`Docker Hub`镜像
  
    ```bash
    sudo mkdir -p /etc/docker
    sudo tee /etc/docker/daemon.json <<-'EOF'
    {
        "registry-mirrors": ["https://6e0d9uoa.mirror.aliyuncs.com"]
    }
    EOF
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```
  
- 安装`Docker-Compose`

  - 安装`python-pip`

    ```bash
    sudo yum install python-pip -y
    ```

  - 安装`Docker-Compose`

    ```bash
    pip install docker-compose
    ```

  - 查看是否安装成功

    ```bash
    docker-compose version
    ```

## 5. 安装Node.js 环境

- 安装`nvm`

  ```bash
  wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
  ```

- 添加环境变量

  ```bash
  vim ~/.profile
  #下面内容添加到文件中
  export NVM_DIR="${XDG_CONFIG_HOME/:-$HOME/.}nvm"
  [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm
  ```

- 安装 node V8

  ```bash
  nvm install v8.15.1
  ```

- 查看安装是否成功

  ```bash
  node -v
  npm -v
  ```

- npm 设置淘宝源

  ```bash
  npm config set registry https://registry.npm.taobao.org --global
  npm config set disturl https://npm.taobao.org/dist --global
  ```

## 6. 下载Fabric源码，安装镜像

- 克隆源码

  ```bash
  cd /opt/gopath/src/github.com/hyperledger
  git clone https://github.com/hyperledger/fabric.git
  git checkout v1.4.2 #将版本切换到1.4.2
  ```

- 完成后进入 `fabric/scripts` 文件夹,执行 `bootstrap.sh` 脚本，自动进行**克隆fabric-samples并检出适当的版本**、**安装二进制可执行文件和配置文件**、 **下载fabric相关镜像**的操作

  ```bash
  cd fabric/scripts
  ./bootstrap.sh
  ```

- 如果下载二进制文件非常慢，可以下载到本地，然后解压拷贝到`/opt/gopath/bin`目录下面或者是`/usr/local/bin`目录下面，下面是下载地址

  - [hyperledger-fabric-linux-amd64-1.4.2.tar.gz](https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.4.2/hyperledger-fabric-linux-amd64-1.4.2.tar.gz)

  - [hyperledger-fabric-ca-linux-amd64-1.4.2.tar.gz](https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric-ca/hyperledger-fabric-ca/linux-amd64-1.4.2/hyperledger-fabric-ca-linux-amd64-1.4.2.tar.gz)

  - 检查`/opt/gopath/bin`目录下面（或者`/usr/local/bin`）之前拷贝过来的文件权限，如果没有可执行权限，按照下面模式，更改`bin`目录下面所有拷贝过来的二进制文件权限

    ```bash
    chmod 775 configtxgen #提升权限
    ```
  
  - **注**：这样需要编辑`bootstrap.sh`注释掉最后调用的`binariesInstall`方法
  
    ```bash
    #........
    #...........
    if [ "$BINARIES" == "true" ]; then
      echo
      echo "Installing Hyperledger Fabric binaries"
      echo
      # binariesInstall #---注释掉这里
    fi
    #...........
    #........
    ```
  
  - 再次执行`bootstrap.sh`文件下载`docker`镜像
  
    ```bash
    ./bootstrap.sh
    ```

## 7. 运行 first-network 的例子

- 启动和关闭网络

  ```bash
  cd fabric/scripts/fabric-samples/first-network
  ./byfn.sh up # 启动网络
  ./byfn.sh down #关闭网络
  ```
