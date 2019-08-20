# Fabric CA 的介绍及使用

## Fabric CA 简介

Fabric CA 是 Fabric 的证书颁发管理机构，原来是 Fabric 内的 MemberService 组件，后来独立成子项目，主要实现下面几个功能：

- 负责 Fabric 网络内所有实体`(Identity)`的身份管理，包括身份的注册和注销
- 负责证书管理，包括注册证书`(ECerts)`和交易证书`(TCerts)`等的发放和注销
- 服务端支持基于客户端命令行和`RESTful API`的交互方式

### 基本组件

- 服务端：`fabric-ca-server` 实现了核心的`PKI`服务功能，支持多种数据库后台(MySQL，SQLite等)，并支持集成`LDAP(轻量目录访问协议)`作为用户注册管理功能
- 客户端：`fabric-ca-client`封装了服务端的`RESTful API`，提供访问服务端的命令，供用户与服务端进行交互

下图说明了 Hyperledger Fabric CA 服务器如何搭配整个 Hyperledger Fabric 架构

![fabric-ca](images/fabric-ca.png)

## Fabric CA 服务端和客户端的安装

- 编译环境准备

  ```bash
  Golang 1.10+
  # 对于 Ubuntu 环境，需要安装以下依赖
  sudo apt-get install libtool libltdl-dev
  # 对于 CentOS 环境，需要安装以下依赖
  yum install libtool
  ```

- 使用源码编译安装

  ```bash
  git clone https://github.com/hyperledger/fabric-ca.git # clone 源码至本地
  cd $GOPATH/src/github.com/hyperledger/fabric-ca/ #切换至源码目录
  # 使用 make 命令编译安装
  make fabric-ca-server
  make fabric-ca-client
  ```

  - 编译生成的二进制文件在`fabric-ca/bin/`目录下，我们可以将其拷贝到`$GOPATH/bin/`目录下，或者将`fabric-ca/bin`目录添加到环境变量，这样就可以在任何地方使用这两个组件了

- 使用 `go get`命令安装

  ```bash
  go get -u github.com/hyperledger/fabric-ca/cmd/...
  ```

  该命令会默认将`Fabric-CA`的`Server`和`Client`的二进制文件安装到`$GOPATH/bin/`目录下

## fabric-ca-server

我们使用`fabric-ca-server --help`命令就能查看 CA 服务端所支持的参数和命令，另外[官方文档](https://hyperledger-fabric-ca.readthedocs.io/en/latest/servercli.html)上也有介绍

- 配置读取：`fabric-ca-server`服务所需要的相关配置项会依次尝试从`命令行参数`，`环境变量(命令需要带有FABRIC_CA_SERVER)前缀`，或`主配置目录(未指定配置文件路径时)`下的本地配置文件读取。

  例如，指定启用`TLS`可以通过以下三种方式，优先级`从高到低`：

  - 命令行参数：--tls-enabled=true
  - 环境变量：FABRIC_CA_SERVER_TLS_ENABLED=true
  - 配置文件：tls.enabled=true

- 主配置目录：`fabric-ca-server`服务会依次从环境变量`FABRIC_CA_SERVER_HOME`、`FABRIC_CA_HOME`、`CA_CFG_PATH`等中读取主配置，一般推荐默认的`/etc/hyperledger/fabric-ca-server`路径作为主配置目录环境变量指向路径。如果当这些环境变量都不存在，会使用`当前目录`作为主配置目录

- `init`命令：初始化一个`fabric-ca-server`服务，主要用于生成秘钥相关的证书文件(文件已经存在的话则跳过生成)以及配置文件等。

  当主目录下配置文件不存在时，需要使用 `-b`参数指定 CA 服务端启动后默认的管理员用户名和密码，以便做进一步的操作；如果有配置文件存在或者指定启用了`LDAP`功能，则不需要使用`-b`参数，示例命令入下：

  ```bash
  fabric-ca-server init -b admin:adminpw
  ```

  上面命令执行完之后，会在主配置目录生成下面几个文件（事先不存在的情况下）：

  - `ca-cert.pem`：自签名的CA证书文件
  - `fabric-ca-server-config.yaml`：默认的 CA 服务端配置文件
  - `fabric-ca-server.db`：默认存放数据的 Sqlite 数据库文件
  - `msp/keystore/`：该路径下有存放个人身份的私钥文件(_sk文件)，对应签名证书等
    - `..._sk`：个人身份的私钥文件
    - `IssuerSecretKey`：`idemix`证书颁发者的秘钥文件
    - `IssuerRevocationPrivateKey`：``idemix`证书吊销者的秘钥文件
  - `IssuerPublicKey`：`idemix`证书颁发者的公钥文件
  - `IssuerRevocationPublicKey`：`idemix`证书吊销者的公钥文件

- `start`命令：使用该命令快速启动一个 CA 服务，默认会查找本地主配置目录路径下的证书文件和配置文件，类似的，如果没有指定`LDAP`服务，或者配置文件不存在，需要使用`-b`参数来指定 Fabric-CA 默认启动的管理员用户名和密码，示例命令如下：

  ```bash
  fabric-ca-server start -b admin:adminpw
  ```

  执行上面命令后，默认的管理员名称和密码是：`admin`和`adminpw`

  如果之前没有执行初始化，则会先执行初始化操作

- 配置文件`fabric-ca-server-config.yaml`：在配置文件中可以配置端口、TLS、CA、注册管理、数据库、LDAP、证书的申请和签发、BCCSP(加密库)、中间 CA 的配置等

## fabric-ca-client

同样的，我们使用`fabric-ca-client --help`命令就能查看 CA 客户端所支持的参数和命令，另外[官方文档](https://hyperledger-fabric-ca.readthedocs.io/en/latest/clientcli.html)上也有介绍

- 配置读取：与服务端的配置读取类似，`fabric-ca-client`服务所需要的相关配置项会依次尝试从`命令行参数`，`环境变量(命令需要带有FABRIC_CA_CLIENT_前缀`，或`主配置目录(未指定配置文件路径时)`下的本地配置文件读取。
- 主配置目录：`fabric-ca-client`服务会依次从环境变量`FABRIC_CA_CLIENT_HOME`、`FABRIC_CA_HOME`、`CA_CFG_PATH`等中读取主配置，一般推荐使用`$HOME/.fabric-ca-client`路径作为主配置目录环境变量指向路径。如果当这些环境变量都不存在，会默认使用`$HOME/.fabric-ca-client`作为主配置目录

一般情况下，主要用到下面几个命令：

- `enroll`：登记实体身份。该命令会向服务端申请签发`ECert`证书，示例命令如下

  ```bash
  fabric-ca-client enroll -u http://admin:adminpw@localhost:7054
  ```

  上面命令使用默认的 admin 管理员用户进行登记。如果未指定配置目录，默认会在`$HOME/.fabric-ca-client`目录下生成下面几个文件

  ```bash
  ├── fabric-ca-client-config.yaml # 客户端的配置文件
  └── msp
      ├── cacerts
      │   └── localhost-7054.pem # CA 根证书
      ├── IssuerPublicKey # idemix 证书颁发者公钥
      ├── IssuerRevocationPublicKey # idemix 证书吊销者公钥
      ├── keystore
      │   └── ......_sk # 秘钥
      ├── signcerts
      │   └── cert.pem # 客户端证书
      └── user
  ```

- `rigister`：注册新的用户实体。执行注册新的用户实体的客户端必须已经通过登记认证，并且拥有足够的权限(所注册用户的`hf.Registrar.Roles`和`affiliation`都不能超出调用者的属性)来进行注册。

## crypto-config 目录文件介绍

```bash
├── ordererOrganizations # orderer 节点组织树
│   └── example.com # 组织域，组织的命名域
│       ├── ca # 存放组织(example.com)的 CA 根证书和私钥文件 ?? 【有中间CA情况下，这里是根还是中间】
│       │   ├── .........._sk # 组织(example.com)的私钥文件
│       │   └── ca.example.com-cert.pem # 组织(example.com)的CA根证书
│       │
│       ├── msp # 存放代表该组织(example.com)的身份信息
│       │   ├── admincerts # 存放组织管理员的身份验证证书，被组织(example.com)的根证书签名
│       │   │   └── Admin@example.com-cert.pem
│       │   ├── cacerts # 存放组织(example.com)CA根证书，同 ca 目录下的文件
│       │   │   └── ca.example.com-cert.pem
│       │   ├── intermediatecerts # (可选) 存放组织(example.com)的中间 CA 证书
│       │   │   └── intermediateca.example.com-cert.pem
│       │   ├── tlscacerts # 存放组织(example.com)用于 TSL 安全连接的 CA 根证书，自签名
│       │   │   └── tlsca.example.com-cert.pem
│       │   └── intermediatecerts # (可选) 存放组织(example.com)用于 TSL 安全连接的中间 CA 证书
│       │       └── intermediateca.example.com-cert.pem
│       │  
│       ├── orderers # 存放该组织所有 orderer 节点的信息
│       │   └── orderer.example.com # 第一个 orderer 节点的信息，包含代表其身份的msp证书文件和用于TLS连接的tls证书文件
│       │       ├── msp # 存放代表第一个 orderer 节点身份的 msp 证书文件
│       │       │   ├── admincerts # 存放组织管理员的身份验证证书，Oderer节点将基于其来认证xxx管理员身份 ??
│       │       │   │   └── Admin@example.com-cert.pem
│       │       │   ├── cacerts # 存放组织(example.com)的 CA 根证书
│       │       │   │   └── ca.example.com-cert.pem
│       │       │   ├── intermediatecerts # (可选) 存放组织(example.com)的中间 CA 证书
│       │       │   │   └── intermediateca.example.com-cert.pem
│       │       │   ├── keystore # 存放本 orderer 节点的身份私钥，用来签名
│       │       │   │   └── .........._sk
│       │       │   ├── signcerts # 存放验证本 orderer 节点签名的证书，被组织 CA 根证书签名
│       │       │   │   └── orderer.example.com-cert.pem
│       │       │   ├── tlscacerts # 存放 TLS 连接用的身份证书， 既组织的 TLS CA 根证书
│       │       │   │   └── tlsca.example.com-cert.pem
│       │       │   ├── intermediatecerts # 存放 TLS 中间 CA 证书，既组织的 TLS 中间 CA 证书
│       │       │   │   └── intermediateca.example.com-cert.pem
│       │       │   ├── crls #(可选) 存放证书吊销列表
│       │       │   │   └── XXXX
│       │       │   └──  config.yaml # 映射部门和证书 OU 字段的 YAML 文件
│       │       └── tls # 存放本 orderer 节点 tls 相关的证书和私钥
│       │           ├── ca.crt # 组织(example.com)的 tlsCA 根证书[tlsca.example.com-cert.pem]
│       │           ├── server.crt # 验证本节点签名的证书，被组织 tlsCA 根证书签名 tls/signcerts [cert.pem]
│       │           └── server.key # 本节点的身份私钥，用来签名 tls/keystore[.........._sk]
│       │
│       ├── tlsca # 存放组织(example.com)用于 TSL 安全连接的 CA 根证书和秘钥 ?? 【有中间tlsCA情况下，这里是根还是中间】
│       │   ├── .........._sk # 秘钥
│       │   └── tlsca.example.com-cert.pem # 证书
│       └── users # 存放属于该组织的用户实体
│           └── Admin@example.com # 组织管理员用户信息，包括其 msp 证书和 tls 证书
│               ├── msp # 存放管理员用户的 msp 证书文件
│               │   ├── admincerts # 存放组织管理员的身份验证证书，被组织(example.com)的根证书签名
│               │   │   └── Admin@example.com-cert.pem
│               │   ├── cacerts # 存放组织(example.com)的 CA 根证书
│               │   │   └── ca.example.com-cert.pem
│               │   ├── keystore # 本用户的身份私钥，用来签名
│               │   │   └── .........._sk
│               │   ├── signcerts # 管理员用户的身份验证证书，这是该组织下所有实体 msp 文件中 admincerts 文件的来源
│               │   │   └── Admin@example.com-cert.pem
│               │   └── tlscacerts # 存放 TLS 连接用的身份证书， 既组织的 TLS CA 根证书
│               │       └── tlsca.example.com-cert.pem
│               └── tls
│                   ├── ca.crt # 组织(example.com)的 tlsCA 根证书[ca.example.com-cert.pem]
│                   ├── client.crt # 管理员用户的身份验证证书 Admin@example.com/tls/signcerts [cert.pem]
│                   └── client.key # 管理员用户的身份私钥，用来签名 Admin@example.com/tls/keystore [.........._sk]
│
│
└── peerOrganizations # peer 节点组织数，与 orderer 节点结构相同
    └── org1.example.com
        ├── ca
        ├── msp
        ├── peers
        │   └── peer0.org1.example.com
        │       ├── msp
        │       └── tls
        ├── tlsca
        └── users
            ├── Admin@org1.example.com
            │   ├── msp
            │   └── tls
            └── User1@org1.example.com
                ├── msp
                └── tls
```

**config.yaml文件介绍**：映射部门和证书 OU 字段的 YAML 配置文件，用`<Certificate,OrganizationalUnitIdentifier>`表示一个映射表

```yaml
NodeOUs:
  Enable: true # 启用标识分类
  ClientOUIdentifier: # 客户端(client)标识符
    Certificate: cacerts/ca.org1.example.com-cert.pem # 验证客户端(client)身份的 CA 或中间 CA 证书
    OrganizationalUnitIdentifier: client # 与客户端(client)的x509证书中包含的OU匹配的值
  PeerOUIdentifier: # 对等节点(peer)标识符
    Certificate: cacerts/ca.org1.example.com-cert.pem # 验证对等节点(peer)身份的 CA 或中间 CA 证书
    OrganizationalUnitIdentifier: peer # 与对等节点(peer)的x509证书中包含的OU匹配的值
```

- ecdsa-with-SHA256 使用 SHA256 算法对内容先计算摘要，再应用签名算法对摘要进行签名
- Orderer 节点管理员证书位于 `crypto-config/ordererOrganizations/example.com/msp/admincerts`。由 CA 颁发给 `Admin@example.com`
- TLS 证书是自签名证书，与之前的 CA 证书没有关系。位于 `crypto-config/ordererOrganizations/example.com/msp/tlscacerts`
- TLS 客户端证书是由 TLS CA 颁发给 `Admin@example.com` 的用于 TLS 连接的证书。位于 `crypto-config/ordererOrganizations/example.com/users/Admin@example.com/tls`

### 证书间的关系

- 颜色相同的线条表示文件内容相同。
- 红色箭头表示 CA 颁发证书给某个组织或个人

![crypto-config](images/crypto-config.png)

## Fabric CA 的实践使用

- [HyperLedger Fabric 1.4 生产环境使用ca生成msp和tls](https://www.cnblogs.com/llongst/p/9786024.html)
- [超级账本HyperLedger：Fabric-CA的使用演示](https://www.lijiaocn.com/项目/2018/05/04/fabric-ca-example.html)
- <https://github.com/dmego/fabric-first-network>
