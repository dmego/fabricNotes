# 基于Fabric 1.4.2 搭建 BlockChain-Explorer(cii-badge分支)

## 1.环境依赖说明

- [nodejs](https://nodejs.org/dist/) 8.11.x (对于版本9.x还不支持)
- [PostgreSQL]([https://www.postgresql.org](https://www.postgresql.org/)) 9.5 以上
- [JQ](https://stedolan.github.io/jq/)，jq 是一个轻量级且灵活的命令行JSON处理器
- Git
- gcc-c++

已经验证的Docker环境支持版本

- [Docker CE 18.09.2 or later](https://hub.docker.com/search/?type=edition&offering=community&operating_system=linux)
- [Docker Compose 1.14.0](https://docs.docker.com/compose)

> **注**：测试环境为`CentOS Linux release 7.4.1708 (Core)`

## 2. Git 等依赖软件安装

- 安装

  ```bash
  yum install -y git
  yum install -y gcc-c++
  ```

- 测试

  ```bash
  git --version
  gcc --version
  g++ --version
  ```

## 3. Node.js 环境安装

- 下载解压

  ```bash
  wget https://nodejs.org/dist/v8.11.4/node-v8.11.4-linux-x64.tar.xz
  tar -xvf node-v8.11.4-linux-x64.tar.xz 
  ```

- 建立软链接

  ```bash
  ln -s /root/node-v8.11.4-linux-x64/bin/node /usr/local/bin/node  
  ln -s /root/node-v8.11.4-linux-x64/bin/npm /usr/local/bin/npm
  ```

- 测试

  ```bash
  node -v # v8.11.4
  npm -v # 5.6.0
  ```

- npm 设置淘宝源

  ```bash
  npm config set registry https://registry.npm.taobao.org --global
  npm config set disturl https://npm.taobao.org/dist --global
  ```
  
  > **注**：如果用`nvm`安装，在执行初始化数据库步骤时可能会出现问题

## 4.PostgreSQL 安装(*号步骤可以不用操作)

- 添加 RPM

  ```bash
  yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
  ```

  > **注**：如果无效，建议去[官方下载地址](https://www.postgresql.org/download/linux/)查看

- 安装PostgreSQL 9.5

  ```bash
   yum install postgresql95-server postgresql95-contrib -y
  ```

- 初始化数据库

  ```bash
  /usr/pgsql-9.5/bin/postgresql95-setup initdb
  ```

- 设置开机自启并启动服务

  ```bash
  systemctl enable postgresql-9.5.service # 设置开机自启
  systemctl start postgresql-9.5.service # 启动服务
  ```

> **注**：PostgreSQL 安装完成后，会建立一下`postgres`用户，用于执行PostgreSQL，数据库中也会建立一个`postgres`用户，默认密码为自动生成，需要在系统中改一下。

- 修改用户密码

  ```bash
  su - postgres  # 切换用户，执行后提示符会变为 '-bash-4.2$'
  psql -U postgres # 登录数据库，执行后提示符变为 'postgres=#'
  ALTER USER postgres WITH PASSWORD 'postgres';  #设置postgres用户密码（！！！这个密码要注意下不能包含@符号）
  \q  #退出数据库
  exit # 退出 postgres 用户
  ```

- (*)开启远程访问

  ```bash
  vi /var/lib/pgsql/9.5/data/postgresql.conf
  # 修改 #listen_addresses = 'localhost' 为 listen_addresses='*',表示任何人都可以访问
  # 当然，此处‘*’也可以改为任何你想开放的服务器IP
  ```

- (*)信任远程连接

  ```bash
  vi /var/lib/pgsql/9.5/data/pg_hba.conf
  # 修改如下内容，信任指定服务器连接
  # IPv4 local connections:
  host    all            all      127.0.0.1/32      md5
  # 增加一行
  host    all             all      0.0.0.0/0          md5
  ```

- (*)重启PostgreSQL服务

  ```bash
  systemctl restart postgresql-9.5.service
  ```

## 5. 安装 JQ

- 安装 JQ 需要的 epel 源

  ```bash
  wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
  rpm -ivh epel-release-latest-7.noarch.rpm
  yum repolist 
  ```

- 安装

  ```bash
  yum install jq -y
  ```


## 6.下载配置 Hyperledger Explorer

- 下载源码并切换到`cii-badge`分支

  ```bash
  git clone https://github.com/hyperledger/blockchain-explorer.git
  cd blockchain-explorer
  git checkout cii-badge
  ```

- 初始化数据库

  ```bash
  cd app/persistence/fabric/postgreSQL/
  chmod -R 775 db/
  cd db/
  ./createdb.sh
  # 如果执行 createdb.sh 提示没有权限，执行下面语句后再执行 createdb.sh
  chmod +x /root
  # 回到 blockchain-explorer 目录下
  cd ../../../../../../blockchain-explorer
  ```

- 修改有关数据库等配置的文件

  ```bash
  vim app/explorerconfig.json
  # 修改 username 和 passwd,官网还修改了 jwt.expiresIn 为 2 days,完整配置如下
  ```

  ```json
  {
  	"persistence": "postgreSQL",
  	"platforms": ["fabric"],
  	"postgreSQL": {
  		"host": "127.0.0.1",
  		"port": "5432",
  		"database": "fabricexplorer",
  		"username": "postgres", 
  		"passwd": "postgres"
  	},
  	"sync": {
  		"type": "local",
  		"platform": "fabric",
  		"blocksSyncTime": "1"
  	},
  	"jwt": {
  		"secret": "a secret phrase!!",
  		"expiresIn": "2 days"
  	}
  }
  ```

- 修改Fabric 网络配置（如果启动的是 `Fabic` 示例中的 `first-network`，则）

  - 修改 `app/platfom/fabric/config.json` 文件

    ```bash
    vim app/platfom/fabric/config.json
    # 官方默认写的是 first-network,我们可以改成自己的 my-network
    ```

    ```bash
    {
    	"network-configs": {
    		"my-network": {
    			"name": "my-network",
    			"profile": "./connection-profile/my-network.json"
    		}
    	},
    	"license": "Apache-2.0"
    }
    ```

  - 进一步修改 `./connection-profile/my-network.json`文件

    ```bash
    # 这里我们需要自己先建自己的 my-network 配置文件，然后按照自己需要进行配置
    vim app/platfom/fabric/connection-profile/my-network.json
    # 示例配置如下
    ```

    ```bash
    {
    	"name": "my-network",
    	"version": "1.0.0",
    	"license": "Apache-2.0",
    	"client": {
    		"tlsEnable": true,
    		"adminUser": "admin",
    		"adminPassword": "adminpw",
    		"enableAuthentication": false,
    		"organization": "Org1",
    		"connection": {
    			"timeout": {
    				"peer": {
    					"endorser": "300"
    				},
    				"orderer": "300"
    			}
    		}
    	},
    	"channels": {
    		"mychannel": {
    			"peers": {
    				"peer0.org1.example.com": {}
    			},
    			"connection": {
    				"timeout": {
    					"peer": {
    						"endorser": "6000",
    						"eventHub": "6000",
    						"eventReg": "6000"
    					}
    				}
    			}
    		}
    	},
    	"organizations": {
    		"Org1MSP": {
    			"mspid": "Org1MSP",
    			"fullpath": true,
    			"adminPrivateKey": {
    				"path": "/fabric-path/my-network/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore/9b5e1e91c5e7ca0c0e83799586e729d027e4c1275f079b6c59584f50c7df1faa_sk"
    			},
    			"signedCert": {
    				"path": "/fabric-path/my-network/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts/Admin@org1.example.com-cert.pem"
    			}
    		}
    	},
    	"peers": {
    		"peer0.org1.example.com": {
    			"tlsCACerts": {
    				"path": "/fabric-path/my-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt"
    			},
    			"url": "grpcs://localhost:7051",
    			"eventUrl": "grpcs://localhost:7053",
    			"grpcOptions": {
    				"ssl-target-name-override": "peer0.org1.example.com"
    			}
    		}
    	}
    }
    ```

    >需要注意一下几点事项
    >
    >1. `/fabric-path/my-network/`为自己实际网络配置文件的地址，需要自己修改，例如为：`/root/gopath/src/github.com/hyperledger/fabric/examples/my-network/`
    >2. `Org1MSP.adminPrivateKey.path`下的`..._sk`文件需要修改成自己实际的文件名称
    >3. 如果连接的是远程区块链网络，需要将`localhost`改成远程地址的IP或者域名地址，如果域名和IP没有绑定，需要在`/etc/hosts`文件中手动进行本地绑定，否则后面将无法运行成功

## 7.安装部署 Hyperledger Explorer

- 按照官方步骤安装

  ```bash
  cd blockchain-explorer
  npm install
  cd blockchain-explorer/app/test
  npm install
  npm run test
  cd ../../client/
  npm install
  npm run test:ci -- -u --coverage
  npm run build
  ```

- 如果遇到 root没权限，则需要使用非安全模式，顺便输出下详细日志

  ```bash
  cd blockchain-explorer
  npm install --unsafe-perm -d
  cd blockchain-explorer/app/test
  npm install --unsafe-perm -d
  npm run test
  cd ../../client/
  npm install --unsafe-perm -d
  npm run test:ci -- -u --coverage
  npm run build --unsafe-perm -d
  ```

  > **注**：
  >
  > ​	1.如果中间出错，重新安装时先要删除node_modules文件夹，client里的也需要，`npm run test`过程中出现问题时，如果最后所有测试都通过了可以不用管
  >
  > ​	2.如果`npm run build`时出现`The build failed because the process exited too early. This probably means the system ran out of memory or someone called kill -9 on the process`错误，可能是代码的打包比较占内存，多试几次看看，如果不行，停用掉几个服务，清清内存再试

- 运行程序

  ```bash
  cd blockchain-explorer
  ./start.sh
  ```

  > **注**：日志文件：
  >
  > `logs/app` app 日志
  > `logs/console` 运行日志
  > `logs/db` db 日志
  > 这几个文件里面的日志要结合看才能更好的解决问题
  >
  > 
  >
  > 启动后显示如下，查看日志，如果没有错误，访问：`http://IP:8080`即可查看区块链网络的信息

  ```bash
  ************************************************************************************
  **************************** Hyperledger Explorer **********************************
  ************************************************************************************
  ***** Please check the log [logs/console/console-2019-07-25.log] for any error *****
  ************************************************************************************
  ```

  

## 参考

- [Hyperledger Explorer 安装部署（pg版）](https://www.jianshu.com/p/eac679b2e871)
- [Fabric多台服务器的部署(五)](https://www.jianshu.com/p/54bdd1555d4b)
- [Hyperledger Explorer部署](https://www.e-learn.cn/content/qita/1947778)