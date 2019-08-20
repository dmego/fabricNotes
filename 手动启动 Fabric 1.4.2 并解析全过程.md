# 手动启动 Fabric 1.4.2 并解析全过程

> **注**：以基于`docker`容器的方式启动；测试网络为 `first-network`，所以各种配置文件可以不用创建和修改即可使用；网络模式为 `solo`

## 1. 生成组织关系和身份证书

Fabric 项目提供了 `cryptogen` 工具（基于 crypto 标准库）实现自动化生成各个组织、节点和用户成员的身份证书文件。这一过程需要依赖于`crypto-config.yaml`配置文件。

### 1.1 `cryptogen` 工具的使用

首先我们先来看一下`cryptogen`工具使用，这个工具之前我们已经安装在`$GOPATH/bin`目录下面了。我们使用命令`cryptogen --help`可以查看该工具主要有如下几个功能：

```bash
generate [<flags>] # 根据配置文件生成各种证书文件
showtemplate # 显示默认的配置文件
version # 显示本工具的版本信息
extend [<flags>] # 扩展现有的网络，根据配置文件生成需要扩展的证书文件
```

运行`cryptogen generate --help`，我们可以看到，对于`generate`命令还有如下的参数设置：

```bash
--output="crypto-config"  # 指定生成的各种证书文件的输出存放目录,例如 crypto-config
--config=CONFIG           # 指定使用到的配置文件,例如 crypto-config.yaml
```

通样，对于`extend`命令，我们也可以看到还有如下的参数可以设置：

```bash
--input="crypto-config"  # 指定已经存在的网络的证书文件存放位置
--config=CONFIG          # 指定使用到的扩展的配置文件
```

工具的基本使用示例如下：

```bash
cryptogen generate --config=./crypto-config.yaml --output ./crypto-config # 根据 crypto-config.yaml 配置文件在 crypto-config 目录下生成网络的各种证书文件
cryptogen extend --input="crypto-config" --config=config.yaml # 根据 config.yaml 扩展配置文件在现有的网络证书文件目录 crypto-config 下生成扩展的证书文件
```

### 1.2 `crypto-config.yaml`配置文件解析

一个基本的`crypto-config.yaml`配置文件如下所示，同时也是本文启动网络所使用到的配置文件

```yaml
OrdererOrgs: # 定义 orderer 排序节点组织结构
  - Name: Orderer # Orderer 排序节点组织的名称
    Domain: example.com # orderer 排序节点组织的根域名
    Specs: # 根据 “{{.Hostname}}.{{.Domain}}” 格式的模板生成各个排序节点的访问域名和该排序节点各种证书文件
      - Hostname: orderer  # 生成的访问域名为 orderer.example.com
      - Hostname: orderer2 # 生成的访问域名为 orderer2.example.com
      - Hostname: orderer3 # 生成的访问域名为 orderer3.example.com
      - Hostname: orderer4 # 生成的访问域名为 orderer4.example.com
      - Hostname: orderer5 # 生成的访问域名为 orderer5.example.com

PeerOrgs: # 定义 Peer 节点组织结构
  - Name: Org1 # 第一个组织名称 Org1
    Domain: org1.example.com # 第一个组织的根域名
    EnableNodeOUs: true # 如果为 true，则在 msp 目录下就会生成 config.yaml 文件
    Template: # 模板，根据默认的规则在 Org1 下生成 "0 ~ Count-1" 个peer节点
      Count: 2 # 定义生成 2 个 peer 节点（peer0.org1.example.com，peer1.org1.example.com）
    Users: # 定义创建普通用户的数量，管理员用户（Admin@org1.example.com）会默认创建
      Count: 1 # 创建一个普通用户（User1@org1.example.com）

  - Name: Org2 # 第二个组织名称 Org2
    Domain: org2.example.com # 第一个组织的根域名
    EnableNodeOUs: true # 如果为 true，则在 msp 目录下就会生成 config.yaml 文件
    Template: # 模板，根据默认的规则在 Org2 下生成 "0 ~ Count-1" 个peer节点
      Count: 2 # 定义生成 2 个 peer 节点（peer0.org2.example.com，peer1.org2.example.com）
    Users: # 定义创建普通用户的数量，管理员用户（Admin@org2.example.com）会默认创建
      Count: 1 # 创建一个普通用户（User1@org2.example.com）
```

### 1.3 生成组织关系和身份证书

使用上面的配置文件，通过下面的命令就可以为 Fabric 网络生成指定的拓扑结构的组织和身份证书文件，存放的文件夹目录为`crypto-config`。由于我们测试网络为`first-network`，所以我们可以切换到该目录下，直接运行下面命令即可：

```bash
cd /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network # 切换到 first-network 目录下
cryptogen generate --config=./crypto-config.yaml --output ./crypto-config # 生成证书文件
```

运行完成后，会有如下输出：

```bash
[root@localhost first-network]# cryptogen generate --config=./crypto-config.yaml --output ./crypto-config
org1.example.com
org2.example.com
```

此时，各种配置文件已经生成，我们可以到 `crypto-config`目录下查看生成的证书目录结构，具体的各种文件的解释说明可以到文档`《Fabric CA 的介绍及使用》`中查看，这里就不再赘述。

## 2 生成 Orderer 服务启动的创世区块和 Channel 等文件

 Orderer 节点在启动时，可以指定使用提前生成的初始区块文件作为系统通道的初始配置，创世区块中包括了 Ordering 服务的相关配置信息以及联盟信息。创世区块可以使用`configtxgen`工具生成。生成的过程需要依赖`configtx.yaml`配置文件。其中`configtx.yaml`配置文件定义了整个网络中相关配置和拓扑结构信息。

### 2.1 `configtxgen`工具的使用

这个工具同样的已经被我们安装到`$GOPATH/bin`目录下面了，我们使用`configtxgen --help`命令就能知道这个工具的使用方法及参数说明，该工具没有子命令，只有参数设置，具体解释如下：

```bash
-asOrg <string> # 以指定的组织身份执行更新配置交易（如更新锚节点）的生成，意味着在写集合中只包含了该组织有权限操作的键值
-channelCreateTxBaseProfile <string> # 指定排序系统通道当前状态的配置文件，以允许在通道创新交易（tx）生成期间修改非应用程序参数。仅与 'outputCreateChannelTx' 结合使用才有效
-channelID <string> # 指定在生成创世区块等文件时使用到的通道的 ID
-configPath <string> # 要使用的配置文件（configtx.yaml）的路径（如果已设置的话）
-inspectBlock <string> # 打印指定路径下区块（Block）中包含的配置[解析块]
-inspectChannelCreateTx <string> # 打印指定路径下交易（transaction ）中包含的配置 [解析交易]
-outputAnchorPeersUpdate <string> # 创建更新锚节点的配置更新请求，需要同时使用 -asOrg 来指定组织身份
-outputBlock <string> # 指定生成的创世区块（genesis.block）的路径（如果已设置的话）
-outputCreateChannelTx <string> # 指定生成通道配置文件路径（如果已设置的话）
-printOrg <string> # 将组织的定义打印为json,(对于手动向组件添加组织非常有用)
-profile <string> # 指定配置模板，一般为 configtx.yaml 中的 Profiles 里边的配置项（默认为 SampleInsecureSolo）
-version # 显示工具的版本信息
```

该工具的使用示例如下：

```bash
# 生成创世区块：指定配置为 TwoOrgsOrdererGenesis，通道ID为 byfn-sys-channel，生成的创世区块路径及名称为 ./channel-artifacts/genesis.block
configtxgen -profile TwoOrgsOrdererGenesis -channelID byfn-sys-channel -outputBlock ./channel-artifacts/genesis.block
# 创建通道：指定配置为 TwoOrgsChannel，通道ID为 mychannel，通道配置文件路径为 ./channel-artifacts/channel.tx
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel
# 更新锚节点：指定配置为 TwoOrgsChannel，通道ID为 mychannel，更新锚节点的组织为Org1，MSP为 Org1MSP，指定锚节点更新配置输出文件路径为 ./channel-artifacts/Org1MSPanchors.tx
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP
```

### 2.2 `configtx.yaml`配置文件解析

>`YAML`文件注释：`-`表示数组，`&`表示锚点，`*`表示引用，`<<`表示合并到当前数据

```yaml
################################################################################
#   组织部分：
#   - 本节定义了稍后将在配置中引用的不同组织的标识
################################################################################
Organizations: # 组织定义配置部分
    # 定义排序服务（Orderer）组织
    - &OrdererOrg # &OrdererOrg 这个用法类似指针，此处是被 Profiles 中的 - *OrdererOrg 所引用，下面还有很多类型的用例
        Name: OrdererOrg # Orderer 排序节点组织的名称
        ID: OrdererMSP # Orderer 排序节点组织的ID,也就是本地MSP的名称（这是引用组织的关键）
        MSPDir: crypto-config/ordererOrganizations/example.com/msp # Orderer 排序节点组织的 MSP 证书文件目录路径

        # 定义本层级的应用控制策略，对于排序节点（Orderer）的策略，其权威路径为：/Channel/<Application|Orderer>/<OrgName>/<PolicyName>
        Policies:
            Readers: # 可读
                Type: Signature
                Rule: "OR('OrdererMSP.member')"
            Writers: # 可写
                Type: Signature
                Rule: "OR('OrdererMSP.member')"
            Admins: # 管理员 admin
                Type: Signature
                Rule: "OR('OrdererMSP.admin')"

    - &Org1 # 定义组织1（Org1）
        Name: Org1MSP # 组织1（Org1） 的名称
        ID: Org1MSP # 组织1（Org1） 的ID，也就是 localMSP 的名称
        MSPDir: crypto-config/peerOrganizations/org1.example.com/msp # 组织1（Org1）的 MSP 证书文件目录路径

        # 定义本层级的应用控制策略，对于组织1（Org1）的策略，其权威路径为：/Channel/<Application|Orderer>/<OrgName>/<PolicyName>
        Policies:
            Readers: # 可读
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.peer', 'Org1MSP.client')"
            Writers: # 可写
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.client')"
            Admins: # 管理员 admin
                Type: Signature
                Rule: "OR('Org1MSP.admin')"

         # 定义组织1的锚节点，用于跨组织进行 Gossip 通信
        AnchorPeers:
            - Host: peer0.org1.example.com # 锚节点的主机名（域名）
              Port: 7051 # 锚节点的端口号

    - &Org2 # 定义组织2（Org2）
        Name: Org2MSP # 组织2（Org2） 的名称
        ID: Org2MSP # 组织2（Org2） 的ID，也就是 localMSP 的名称
        MSPDir: crypto-config/peerOrganizations/org2.example.com/msp # 组织2（Org2）的 MSP 证书文件目录路径

        # 定义本层级的应用控制策略，对于组织2（Org2）的策略，其权威路径为：/Channel/<Application|Orderer>/<OrgName>/<PolicyName>
        Policies:
            Readers: # 可读
                Type: Signature
                Rule: "OR('Org2MSP.admin', 'Org2MSP.peer', 'Org2MSP.client')"
            Writers: # 可写
                Type: Signature
                Rule: "OR('Org2MSP.admin', 'Org2MSP.client')"
            Admins: # 管理员 admin
                Type: Signature
                Rule: "OR('Org2MSP.admin')"

        # 定义组织2的锚节点，用于跨组织进行 Gossip 通信
        AnchorPeers:
            - Host: peer0.org2.example.com # 锚节点的主机名（域名）
              Port: 9051 # 锚节点的端口号

################################################################################
#
#  Capabilities 部分（这是在 v1.1.0 版本之后提出来的，不能用于在更早的版本上面）
#   - 本节定义了 fabric 网络的功能，这是 v1.1.0中一个新概念，不应该在 v1.0.x 的 peer 和
#    orderer 中使用，Capabilities 定义了存在于结构二进制文件中的功能，以便该二进制文件安全的
#    参与网络，如果添加新的 MSP 类型，则较新的二进制文件可能会识别并验证此类型的签名，而没有此
#    支持的旧二进制文件将无法验证这些事务，这可能导致具有不同世界状态的不同版本的结构二进制文件。
#    相反，为通道定义功能（Capabilities）会通知那些没有此功能的二进制文件，他们必须在升级之前
#    停止处理事务，对于v1.0.x，如果定义了任何功能（Capabilities）[包括关闭所有的功能配置]，
#    那么v1.0.1 的 peer 会主动崩溃
################################################################################
Capabilities:
    # 通道功能（Capabilities）适用于 orderer 节点和 peer 节点，并且必须得到两者的支持
    # 将功能设置为 true 以要求它
    Channel: &ChannelCapabilities
        # 通道（Channel）的 v1.4.2 是一个行为标记，已被确定为所有在v1.4,2版本运行的 orderers 和 peers
        # 但与先前版本的 orderers 和 peers 不兼容
        V1_4_2: true

    # Orderer功能（Capabilities）仅适用于排序节点，可以安全地与之前版本的 peers 一起使用
    # 将功能设置为 true 以要求它
    Orderer: &OrdererCapabilities
        # Orderer的 v1.4.2 是一个行为标记，已被确认为所有在 v1.4.2 版本运行的 orderers 所需的行为标志，但与之前版本的orderers 不兼容
        # 在启用 v1.4.2 版本的orderer功能之前，请确保通道上所有的排序节点（orderers）都为v1.4.2或更高版本。
        V1_4_2: true

    # 应用程序（Application）仅适用于 peer 节点网络，并且可以安全地与之前版本的 orderer 节点一起使用
    # 将功能的值设置为 true 以使其需要
    Application: &ApplicationCapabilities
        # 应用程序（Application）的 v1.4.2 版本启用了 Fabric 1.4.2 的新的非向后兼容功能和修复
        V1_4_2: true
        # 应用程序（Application）的 v1.3 版本启用了 Fabric 1.3 的新的非向后兼容功能和修复
        V1_3: false
        # 应用程序（Application）的 v1.2 版本启用了 Fabric 1.2 的新的非向后兼容功能和修复（如果设置了更高版本的功能，则无需设置此功能）
        V1_2: false
        # 应用程序（Application）的 v1.1 版本启用了 Fabric 1.1 的新的非向后兼容功能和修复（如果设置了更高版本的功能，则无需设置此功能）
        V1_1: false

################################################################################
#   应用程序（Application）部分
#   - 这个部分定义了配置事务或创世区块应用相关的参数设置的值
################################################################################
Application: &ApplicationDefaults
    # Organizations 是在网络的应用程序端定义为参与者的组织列表
    Organizations:

    # 定义本层级的应用控制策略，对于应用程序（application）策略，其权威路径为 /Channel/Application/<PolicyName>
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"

    # 指定功能能力（Capabilities）为上面定义的 ApplicationCapabilities
    Capabilities:
        <<: *ApplicationCapabilities

################################################################################
#   Orderer 节点部分
#   - 本节定义了要为orderer节点的配置事务或创世区块的相关参数设置的值
################################################################################
Orderer: &OrdererDefaults
    # 要启动的 Orderer 节点的共识类型，现在可以选的是 solo，kafka，etcdraft
    OrdererType: solo # 这里使用的是 solo 模式
    # orderer 节点的服务地址，如果是集群，需要设置多个地址
    Addresses:
        - orderer.example.com:7050
        #- orderer.example2.com:8050
        #- orderer.example3.com:9050
        #- orderer.example4.com:10050
        #- orderer.example5.com:11050

    # 区块打包的最大超时时间（到了该时间就要打包区块）
    BatchTimeout: 2s

    # 区块打包的最大包含交易数
    BatchSize:
        # 一个区块里最大的交易数（最大到了该数量就要打包区块）
        MaxMessageCount: 10
        # 一个区块的最大字节数（任何时候都不能超过，最大到了该大小就要打包区块）
        AbsoluteMaxBytes: 99 MB
        # 一个区块的建议字节数，如果一个交易消息的大小超过了该值，就会被放到一个更大的区块中去
        PreferredMaxBytes: 512 KB

    # kafka 共识的相关配置设置
    Kafka:
        # kafka 的 brokens 服务地址，允许有多个
        # 注意: 使用IP:端口(IP:port)表示法
        Brokers:
            - 127.0.0.1:9092

    # 参与维护 Orderer 排序节点的组织，默认为空
    Organizations:

    # 定义本层级的应用控制策略，对于 Orderer 策略，其权威路径为：/Channel/Orderer/<PolicyName>
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        # BlockValidation 配置项指定了哪些签名必须包含在区块中，以便对 peer 节点进行验证
        BlockValidation:
            Type: ImplicitMeta
            Rule: "ANY Writers"

################################################################################
#   Channel 通道部分
#    - 本节定义要写入创世区块或配置交易的通道参数
################################################################################
Channel: &ChannelDefaults

    # 定义本层级的应用控制策略，对于 Channel 策略，其权威路径为： /Channel/<PolicyName>
    Policies:
        # 谁可以调用(invoke)交付区块(Deliver)的 API
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        # 谁可以调用(invoke)广播区块(Broadcast)的 API
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        # 默认情况下，谁可以修改此配置级别的元素
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
  
   # Capabilities 配置描述应用层级的能力需求，这里直接引用前面 Capabilities 配置段中的 ChannelCapabilities 配置项
    Capabilities:
        <<: *ChannelCapabilities

################################################################################
#   Profile 部分配置
#   - 本节定义用于 configtxgen 工具的配置入口，包含委员会（Consortiums）的配置入口。 Profile
#   部分的配置主要是引入上面五个部分的配置参数。 configtxen 工具通过调用 Profile 参数，可以
#   实现生成特点的区块文件
################################################################################
Profiles:

    # 创建 Orderer 创世区块的相关参数配置（Solo共识），包含 orderer 和联盟（Consortiums）两个部分
    TwoOrgsOrdererGenesis:
        # 合并引入 Channel 部分的配置
        <<: *ChannelDefaults
        # 合并引入 Orderer 部分的配置
        Orderer:
            <<: *OrdererDefaults
            # 指定组织 配置为 OrdererOrg
            Organizations:
                - *OrdererOrg
            # 指定能力要求为 OrdererCapabilities
            Capabilities:
                <<: *OrdererCapabilities
        #定义 Orderer 所服务的联盟列表
        Consortiums:
            # 指定联盟列表名称为 SampleConsortium
            SampleConsortium:
                # 定义联盟中的组织，这里为 Org1，Org2
                Organizations:
                    - *Org1
                    - *Org2

    # 创建应用通道（Channel）的配置
    TwoOrgsChannel:
        # 应用通道关联的联盟列表为 SampleConsortium
        Consortium: SampleConsortium
        # 合并引入通道（Channel）配置
        <<: *ChannelDefaults
        # 合并引入 Application 的相关配置
        Application:
            <<: *ApplicationDefaults
            # 初始加入通道的组织，这里为 Org1，Org2
            Organizations:
                - *Org1
                - *Org2
            # 指定能力要求为 ApplicationCapabilities
            Capabilities:
                <<: *ApplicationCapabilities

    # 创建 Orderer 创世区块的相关参数配置（kafka 共识）
    # 生成创世区块时，configtxgen 指定的 -profile 参数为 SampleDevModeKafka
    SampleDevModeKafka:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            OrdererType: kafka
            Kafka:
                Brokers:
                - kafka.example.com:9092

            Organizations:
            - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Application:
            <<: *ApplicationDefaults
            Organizations:
            - <<: *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                - *Org1
                - *Org2

    # 创建 Orderer 创世区块的相关参数配置（raft 共识）
    # 生成创世区块时，configtxgen 指定的 -profile 参数为 SampleMultiNodeEtcdRaft
    SampleMultiNodeEtcdRaft:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            OrdererType: etcdraft
            EtcdRaft:
                # 指定参与共识的 orderer 节点地址，端口，和证书文件
                Consenters:
                - Host: orderer.example.com
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                - Host: orderer2.example.com
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
                - Host: orderer3.example.com
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
                - Host: orderer4.example.com
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer4.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer4.example.com/tls/server.crt
                - Host: orderer5.example.com
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer5.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer5.example.com/tls/server.crt
            Addresses:
                - orderer.example.com:7050
                - orderer2.example.com:7050
                - orderer3.example.com:7050
                - orderer4.example.com:7050
                - orderer5.example.com:7050

            Organizations:
            - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Application:
            <<: *ApplicationDefaults
            Organizations:
            - <<: *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                - *Org1
                - *Org2
```

### 2.3 生成 Orderer 服务启动的创世区块和 Channel 等文件

#### 2.3.1 生成创世区块

通过如下命令指定使用`configtx.yaml`文件中定义的`TwoOrgsOrdererGenesis`模板，来生成 Orderer 服务系统通道的初始创世区块文件。

```bash
cd /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network # 切换到 first-network 目录下

# -profile 指定配置模板为 TwoOrgsOrdererGenesis；-outputBlock 指定生成的创世区块文件路径；-channelID 指定系统通道名称为 byfn-sys-channel
configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block -channelID byfn-sys-channel
```

运行完成后，会有如下输出，可以看出 Orderer 共识使用的是 solo

```bash
2019-07-25 21:38:20.012 EDT [common.tools.configtxgen] main -> INFO 002 Loading configuration
2019-07-25 21:38:20.080 EDT [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 orderer type: solo
2019-07-25 21:38:20.080 EDT [common.tools.configtxgen.localconfig] Load -> INFO 004 Loaded configuration: /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/configtx.yaml
2019-07-25 21:38:20.147 EDT [common.tools.configtxgen.localconfig] completeInitialization -> INFO 005 orderer type: solo
2019-07-25 21:38:20.147 EDT [common.tools.configtxgen.localconfig] LoadTopLevel -> INFO 006 Loaded configuration: /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/configtx.yaml
2019-07-25 21:38:20.159 EDT [common.tools.configtxgen] doOutputBlock -> INFO 007 Generating genesis block
2019-07-25 21:38:20.159 EDT [common.tools.configtxgen] doOutputBlock -> INFO 008 Writing genesis block
```

(*) 生成的文件位于目录`channel-artifacts`下，我们可以通过下面的命令将`Block`详细内容导入到`json`文件中方便查看：

```bash
# -inspectBlock 指定打印配置的块路径为 channel-artifacts/genesis.block
configtxgen -inspectBlock channel-artifacts/genesis.block > channel-artifacts/genesis.block.json
```

#### 2.3.2 生成新建应用通道的配置交易

接下来我们需要使用如下命令新建应用通道（Channel）：

```bash
# -profile 指定配置模板为 TwoOrgsChannel；-outputCreateChannelTx 指定生成的通道文件路径；-channelID 指定通道名称为 mychannel
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel
```

运行完成后，会有如下输出：

```bash
2019-07-25 22:04:30.305 EDT [common.tools.configtxgen] main -> INFO 001 Loading configuration
2019-07-25 22:04:30.376 EDT [common.tools.configtxgen.localconfig] Load -> INFO 002 Loaded configuration: /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/configtx.yaml
2019-07-25 22:04:30.451 EDT [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 orderer type: solo
2019-07-25 22:04:30.451 EDT [common.tools.configtxgen.localconfig] LoadTopLevel -> INFO 004 Loaded configuration: /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/configtx.yaml
2019-07-25 22:04:30.451 EDT [common.tools.configtxgen] doOutputChannelCreateTx -> INFO 005 Generating new channel configtx
2019-07-25 22:04:30.458 EDT [common.tools.configtxgen] doOutputChannelCreateTx -> INFO 006 Writing new channel tx
```

#### 2.3.3 生成锚节点配置更新文件

通道创建完成之后，我们接着需要提前生成锚节点配置更新文件，生成命令如下：

```bash
# -profile 指定配置模板为 TwoOrgsChannel；-outputAnchorPeersUpdate 指定生成的锚节点配置更新文件路径；-channelID 指定通道名称为 mychannel；-asOrg 指定组织为 Org1MSP
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP

# -profile 指定配置模板为 TwoOrgsChannel；-outputAnchorPeersUpdate 指定生成的锚节点配置更新文件路径；-channelID 指定通道名称为 mychannel；-asOrg 指定组织为 Org2MSP
configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
```

两条命令运行完成后，都会有如下输出：

```bash
2019-07-25 22:19:42.190 EDT [common.tools.configtxgen] main -> INFO 001 Loading configuration
2019-07-25 22:19:42.260 EDT [common.tools.configtxgen.localconfig] Load -> INFO 002 Loaded configuration: /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/configtx.yaml
2019-07-25 22:19:42.324 EDT [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 orderer type: solo
2019-07-25 22:19:42.325 EDT [common.tools.configtxgen.localconfig] LoadTopLevel -> INFO 004 Loaded configuration: /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/configtx.yaml
2019-07-25 22:19:42.325 EDT [common.tools.configtxgen] doOutputAnchorPeersUpdate -> INFO 005 Generating anchor peer update
2019-07-25 22:19:42.325 EDT [common.tools.configtxgen] doOutputAnchorPeersUpdate -> INFO 006 Writing anchor peer update
```

使用提前生成锚节点配置更新文件，组织管理员身份就可以更新指定应用通道中组织的锚节点配置。所谓的锚节点就是负责代表组织与其他组织中的节点进行 Gossip 通信的节点。

(*)我们也可以通过下面命令将上面生成的交易（transaction）文件导出到`json`文件中进行查看：

```bash
# -inspectChannelCreateTx 指定要解析的交易文件为 channel-artifacts/channel.tx
configtxgen -inspectChannelCreateTx channel-artifacts/channel.tx > channel-artifacts/channel.tx.json

# -inspectChannelCreateTx 指定要解析的交易文件为 channel-artifacts/Org1MSPanchors.tx
configtxgen -inspectChannelCreateTx channel-artifacts/Org1MSPanchors.tx > channel-artifacts/Org1MSPanchors.tx.json

# -inspectChannelCreateTx 指定要解析的交易文件为 channel-artifacts/Org2MSPanchors.tx
configtxgen -inspectChannelCreateTx channel-artifacts/Org2MSPanchors.tx > channel-artifacts/Org2MSPanchors.tx.json
```

至此，所以文件都生成完成，我们可以在`channel-artifacts`目录下可以看到，生成的文件有：

```bash
├── channel.tx # 应用通道的配置交易文件
├── genesis.block # Ordering 服务启动的创世区块
├── Org1MSPanchors.tx # 组织1的锚节点配置更新的配置交易文件
└── Org2MSPanchors.tx # 组织2的锚节点配置更新的配置交易文件
```

## 3 使用 Docker 容器方式启动 Fabric 网络

### 3.1 所需的`docker-compose`启动文件介绍

启动网络使用到了三个配置文件，他们具有继承和扩展等关系。使用到的配置文件是下面三个：

```bash
base/peer-base.yaml
base/docker-compose-base.yaml
docker-compose-cli.yaml
```

#### 3.1.1 `peer-base.yaml`文件解析

这个配置文件中定义了将要启动的容器(container)所使用的镜像(image)，并且定义了容器启动后自动执行的命令，具体配置详情如下：

```yaml
version: '2'  # 使用的是第2版 Compose 文件格式

services: # 服务
  peer-base: # 定义一个名为 peer-base 的服务
    image: hyperledger/fabric-peer:$IMAGE_TAG # 该服务所依赖的镜像
    environment: # 定义该服务的环境变量
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock # Docker 服务地址
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_byfn # 链码容器使用的网络方式，这里是bridge方式
      - FABRIC_LOGGING_SPEC=INFO # 定义日志级别为 INFO
      #- FABRIC_LOGGING_SPEC=DEBUG
      - CORE_PEER_TLS_ENABLED=true # 使用 TLS
      - CORE_PEER_GOSSIP_USELEADERELECTION=true # 使用选举 Leader 的方式
      - CORE_PEER_GOSSIP_ORGLEADER=false # 不指定 Leader
      - CORE_PEER_PROFILE_ENABLED=true # 使用 profile
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt # 在容器中的 TLS 的证书路径
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key # 在容器中的 TLS 的秘钥路径
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt # 在容器中的 TLS 的根证书路径
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer # 工作目录，即进入容器所在的默认位置
    command: peer node start # 启动容器后执行的第一条命令，启动 Peer 节点

  orderer-base: # 定义一个名为 orderer-base 的服务
    image: hyperledger/fabric-orderer:$IMAGE_TAG # 指定该服务依赖的镜像
    environment: # 定义该服务的环境变量
      - FABRIC_LOGGING_SPEC=INFO # 定义日志级别为 INFO
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0 # orderer 节点的监听地址
      - ORDERER_GENERAL_GENESISMETHOD=file # 创世区块文件的类型为 file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block # 指定创世区块在容器中的路径
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP # Orderer 的本地 MSP ID
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp # 在容器中的本地的MSP文件夹路径
      - ORDERER_GENERAL_TLS_ENABLED=true # 使用 TLS
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key # 在容器中的 TLS 的秘钥路径
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt # 在容器中的 TLS 的证书路径
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt] # 在容器中的 TLS 的根证书路径
      # 以下为kafka集群的配置，没有使用到
      #- ORDERER_KAFKA_TOPIC_REPLICATIONFACTOR=1
      #- ORDERER_KAFKA_VERBOSE=true
      #- ORDERER_GENERAL_CLUSTER_CLIENTCERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      #- ORDERER_GENERAL_CLUSTER_CLIENTPRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      #- ORDERER_GENERAL_CLUSTER_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric # 工作目录，即进入容器所在的默认位置
    command: orderer # 启动容器后执行的第一条命令，启动 Orderer 节点
```

#### 3.1.2 `docker-compose-base.yaml`文件解析

这个配置文件中定义了五个容器，分别是`orderer.example.com`、`peer0.org1.example.com`、`peer1.org1.example.com`、`peer0.org2.example.com`、`peer1.org2.example.com`，其中一个`orderer`节点配置继承自 `peer-base.yaml`中的`orderer-base`服务；四个`peer`节点配置继承自`peer-base.yaml`中的`peer-base`服务。具体详细配置如下：

```yaml
version: '2' # 使用的是第2版 Compose 文件格式

services: # 服务，包含多个容器实例

  orderer.example.com: # 定义一个名为 orderer.example.com 的服务
    container_name: orderer.example.com # 容器名称
    extends: # 扩展，代表要加载的文件或服务
      file: peer-base.yaml # 扩展的文件名
      service: orderer-base # 扩展的服务名称
    volumes: # 挂载的卷 [本机路径下的目录或文件]：[容器中所映射到的地址]
        - ../channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block # 映射创世区块到容器中
        - ../crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp:/var/hyperledger/orderer/msp # 映射 orderer 节点的 msp 文件目录到容器中
        - ../crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/:/var/hyperledger/orderer/tls # 映射 orderer 节点的 tls 文件目录到容器中
        - orderer.example.com:/var/hyperledger/production/orderer
    ports: # 所映射的端口 [本机端口]：[容器端口]
      - 7050:7050

  peer0.org1.example.com: # 定义一个名为 peer0.org1.example.com 的服务
    container_name: peer0.org1.example.com # 容器名称
    extends: # 扩展
      file: peer-base.yaml # 扩展的文件名
      service: peer-base # 扩展的服务名称
    environment: # 定义该服务的环境变量
      - CORE_PEER_ID=peer0.org1.example.com # peer 节点的 ID
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051 # peer 节点的访问路径
      - CORE_PEER_LISTENADDRESS=0.0.0.0:7051 # peer 节点的监听路径
      - CORE_PEER_CHAINCODEADDRESS=peer0.org1.example.com:7052 # peer 节点的链码访问地址
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:7052 # peer 节点的链码监听地址，指定为 0.0.0.0 则自动进行探测
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org1.example.com:8051 # 指定用于在本组织内引导 Gossip 通信的节点
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051 # Gossip 通信的外部节点，既为本组织的锚节点
      - CORE_PEER_LOCALMSPID=Org1MSP # 所属组织的 MSP 的 ID
    volumes: # 挂载的卷 [本机路径下的目录或文件]：[容器中所映射到的地址]
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/etc/hyperledger/fabric/msp # 映射 peer0.org1.example.com 节点的 msp 文件目录到容器中
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls:/etc/hyperledger/fabric/tls # 映射 peer0.org1.example.com 节点的 tls 文件目录到容器中
        - peer0.org1.example.com:/var/hyperledger/production
    ports: # 所映射的端口 [本机端口]：[容器端口]
      - 7051:7051

  ## 下面定义的 peer 服务的相关参数解释与 peer0.org1.example.com 相似
  peer1.org1.example.com:  # 定义一个名为 peer1.org1.example.com 的服务
    container_name: peer1.org1.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org1.example.com
      - CORE_PEER_ADDRESS=peer1.org1.example.com:8051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:8051
      - CORE_PEER_CHAINCODEADDRESS=peer1.org1.example.com:8052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:8052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org1.example.com:8051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls:/etc/hyperledger/fabric/tls
        - peer1.org1.example.com:/var/hyperledger/production

    ports:
      - 8051:8051

  peer0.org2.example.com:  # 定义一个名为 peer0.org2.example.com 的服务
    container_name: peer0.org2.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org2.example.com
      - CORE_PEER_ADDRESS=peer0.org2.example.com:9051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:9051
      - CORE_PEER_CHAINCODEADDRESS=peer0.org2.example.com:9052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:9052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.example.com:9051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org2.example.com:10051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls:/etc/hyperledger/fabric/tls
        - peer0.org2.example.com:/var/hyperledger/production
    ports:
      - 9051:9051

  peer1.org2.example.com: # 定义一个名为 peer1.org2.example.com 的服务
    container_name: peer1.org2.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org2.example.com
      - CORE_PEER_ADDRESS=peer1.org2.example.com:10051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:10051
      - CORE_PEER_CHAINCODEADDRESS=peer1.org2.example.com:10052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:10052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org2.example.com:10051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.example.com:9051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls:/etc/hyperledger/fabric/tls
        - peer1.org2.example.com:/var/hyperledger/production
    ports:
      - 10051:10051
```

#### 3.1.3 `docker-compose-cli.yaml`文件解析

这个配置文件扩展了`docker-compose-base.yaml`中的内容，并指定了`docker` 容器所加入的网络`networks`为`byfn`。而且还启动了一个客户端(`cli`)容器，这是一个`fabric`工具集容器，我们会登录这个容器和 fabric 各个节点进行交互和操作。

```yaml
version: '2'  # 使用的是第2版 Compose 文件格式

volumes: # 声明要挂载的卷，既 1 orderer + 4 peer
  orderer.example.com:
  peer0.org1.example.com:
  peer1.org1.example.com:
  peer0.org2.example.com:
  peer1.org2.example.com:

networks: # 声明一个名称为 byfn 的网络
  byfn:

services: # 声明服务，包含多个容器实例

  orderer.example.com: # 定义一个名为 orderer.example.com 的服务
    extends: # 扩展
      file:   base/docker-compose-base.yaml # 扩展的文件名
      service: orderer.example.com # 扩展的服务名称
    container_name: orderer.example.com # 容器名称
    networks: # 指定容器加入的网络，如果需要加入多个网络，可以定义多个
      - byfn

  peer0.org1.example.com: # 定义一个名为 peer0.org1.example.com 的服务
    container_name: peer0.org1.example.com # 容器名称
    extends: # 扩展
      file:  base/docker-compose-base.yaml # 扩展的文件名
      service: peer0.org1.example.com # 扩展的服务名称
    networks: # 指定容器加入的网络
      - byfn

  # 下面 peer 节点相关参数与上类似
  peer1.org1.example.com:
    container_name: peer1.org1.example.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer1.org1.example.com
    networks:
      - byfn

  peer0.org2.example.com:
    container_name: peer0.org2.example.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer0.org2.example.com
    networks:
      - byfn

  peer1.org2.example.com:
    container_name: peer1.org2.example.com
    extends:
      file:  base/docker-compose-base.yaml
      service: peer1.org2.example.com
    networks:
      - byfn

  cli: # 定义一个客户端服务，方便与各个节点进行交互
    container_name: cli # 容器名称为 cli
    image: hyperledger/fabric-tools:$IMAGE_TAG # 使用到的镜像为 fabric-tools
    tty: true # 使用伪终端
    stdin_open: true # 标准输入
    environment: # 定义该服务的环境变量
      - SYS_CHANNEL=$SYS_CHANNEL # 系统通道名称
      - GOPATH=/opt/gopath # 指定 GOPATH 路径
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock # Docker 服务地址
      #- FABRIC_LOGGING_SPEC=DEBUG
      - FABRIC_LOGGING_SPEC=INFO # 日志级别为 INFO
      - CORE_PEER_ID=cli # 当前节点的 ID 为 cli
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051 # 以下与 peer-base.yaml 相同，表示当前客户端容器默认与 peer0.org1.example.com 进行交互
      - CORE_PEER_LOCALMSPID=Org1MSP # 组织1的本地MSP的ID
      - CORE_PEER_TLS_ENABLED=true # 使用 TLS
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt # peer0.org1.example.com 节点的 TLS 证书路径
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key # peer0.org1.example.com 节点的 TLS 密钥路径
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt # peer0.org1.example.com 节点的 TLS 根证书路径
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp # 组织1中的管理员（Admin@org1.example.com）的MSP文件目录路径
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer # 工作目录，进入容器所在的默认位置
    command: /bin/bash # 启动容器后运行的第一条命令：使用 bash
    volumes: # 挂载的卷 [本机路径下的目录或文件]：[容器中所映射到的地址]
        - /var/run/:/host/var/run/
        - ./../chaincode/:/opt/gopath/src/github.com/chaincode # 映射链码文件夹路径
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ # 映射证书文件夹路径
        - ./scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/ # 映射的脚本文件夹路径
        - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts # 映射的创世区块和应用通道等文件目录路径
    depends_on: # 依赖，需要首先按顺序启动以下容器服务，但是不会等待以下容器完全启动才启动当前容器
      - orderer.example.com
      - peer0.org1.example.com
      - peer1.org1.example.com
      - peer0.org2.example.com
      - peer1.org2.example.com
    networks: # 指定当前容器所加入的网络
      - byfn
```

### 3.2 启动 Fabric 网络中的 Orderer、peer 和 cli 容器服务

当所有的配置文件定义好之后，我们就可以使用`docker-compose`工具来启动所有容器了，启动命令如下：

```bash
# 如果不在 first-network 目录下，要进入该目录下
cd /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network
# -f(--file)指定使用的 Compose 模板文件，up 启动配置文件中定义的所有服务容器，-d 在后台运行服务容器
docker-compose -f docker-compose-cli.yaml up -d
```

该命令执行完成之后，会有如下输出：

```bash
[root@localhost first-network]# docker-compose -f docker-compose-cli.yaml up -d
Creating network "net_byfn" with the default driver
Creating volume "net_peer0.org2.example.com" with default driver
Creating volume "net_peer1.org2.example.com" with default driver
Creating volume "net_peer1.org1.example.com" with default driver
Creating volume "net_peer0.org1.example.com" with default driver
Creating volume "net_orderer.example.com" with default driver
Creating peer0.org1.example.com ... done
Creating peer1.org1.example.com ... done
Creating peer0.org2.example.com ... done
Creating orderer.example.com    ... done
Creating peer1.org2.example.com ... done
Creating cli                    ... done
```

我们可以使用`docker ps`命令来查看后台启动的所有容器，具体如下。我们可以看到，在后台已经有一个 cli 容器，一个 orderer 容器，四个 peer 容器启动了。

```bash
[root@localhost first-network]# docker ps
CONTAINER ID        IMAGE                               COMMAND             CREATED             STATUS              PORTS                      NAMES
7ddf52548cfd        hyperledger/fabric-tools:latest     "/bin/bash"         6 minutes ago       Up 6 minutes                                   cli
a495b24483dc        hyperledger/fabric-orderer:latest   "orderer"           6 minutes ago       Up 6 minutes        0.0.0.0:7050->7050/tcp     orderer.example.com
5349a515df51        hyperledger/fabric-peer:latest      "peer node start"   6 minutes ago       Up 6 minutes        0.0.0.0:10051->10051/tcp   peer1.org2.example.com
a8a579ebd979        hyperledger/fabric-peer:latest      "peer node start"   6 minutes ago       Up 6 minutes        0.0.0.0:7051->7051/tcp     peer0.org1.example.com
5614be188b35        hyperledger/fabric-peer:latest      "peer node start"   6 minutes ago       Up 6 minutes        0.0.0.0:9051->9051/tcp     peer0.org2.example.com
d6672dcc70d4        hyperledger/fabric-peer:latest      "peer node start"   6 minutes ago       Up 6 minutes        0.0.0.0:8051->8051/tcp     peer1.org1.example.com
```

## 4 创建、加入通道，更新锚节点

### 4.1 创建通道操作

当所有容器启动完成之后，我们就可以进入容器，进行创建通道等操作了。首先我们需要使用如下命令进入客户端容器：

```bash
docker exec -it cli bash
```

执行完成之后，我们可以看到光标会变成如下：

```bash
[root@localhost first-network]# docker exec -it cli bash
root@7ddf52548cfd:/opt/gopath/src/github.com/hyperledger/fabric/peer# # 光标所在位置
```

接着先将`Orderer`的`tlsca`证书文件路径写入环境变量：

```bash
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

执行下面命令，创建通道：

```bash
# channel create 创建一个新通道，-o 指定orderer节点地址，-c 指定生成的通道ID -f 指定新建应用通道的配置交易文件 --tls 使用TLS，--cafile 指定 tls 的证书文件路径
peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls true --cafile $ORDERER_CA
```

执行完成之后，会有如下输出：

```bash
2019-07-26 12:36:53.557 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-07-26 12:36:53.632 UTC [cli.common] readBlock -> INFO 002 Received block: 0
```

(*) 我们使用 `ll`命令可以看到，该目录下多了一个`mychannel.block`文件，我们可以使用如下命令将其移动到 `channel-artifacts`文件夹下：

```bash
mv mychannel.block channel-artifacts/
```

(*) 我们还可以使用如下命令将生成的通道文件解析成`json`格式的文件方便查看：

```bash
configtxgen -inspectBlock channel-artifacts/mychannel.block > channel-artifacts/mychannel.json
```

### 4.2 加入通道

创建通道完成之后，我们需要将所有节点都加入到该通道中

#### 4.2.1 `peer0.org1.example.com`节点加入通道

先配置环境变量，由于`peer0.org1.example.com`节点是默认与`cli`容器连接的，所有可以不必配置环境变量，但是为与其他节点加入通道流程一致，这里也配置一下环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
```

执行下面命令，加入通道：

```bash
# channel join 加入通道，-b 通道的区块文件
peer channel join -b channel-artifacts/mychannel.block
```

执行完成后，会有如下输出：

```bash
root@7ddf52548cfd:/opt/gopath/src/github.com/hyperledger/fabric/peer# peer channel join -b channel-artifacts/mychannel.block
2019-07-26 13:04:33.273 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-07-26 13:04:33.349 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

#### 4.2.2 `peer1.org1.example.com`节点加入通道

配置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org1MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer1.org1.example.com:8051
```

执行下面命令，加入通道，执行完成后，输出内容与`peer0.org1.example.com`的相同

```bash
peer channel join -b channel-artifacts/mychannel.block
```

#### 4.2.3 `peer0.org2.example.com`节点加入通道

配置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org2MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
```

执行下面命令，加入通道，执行完成后，输出内容与`peer0.org1.example.com`的相同

```bash
peer channel join -b channel-artifacts/mychannel.block
```

#### 4.2.4 `peer1.org2.example.com`节点加入通道

配置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org2MSP"
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer1.org2.example.com:10051
```

执行下面命令，加入通道，执行完成后，输出内容与`peer0.org1.example.com`的相同

```bash
peer channel join -b channel-artifacts/mychannel.block
```

### 4.3 更新锚节点操作

#### 4.3.1 更新 `Org1`的锚节点

配置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org1MSP"
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
```

执行下面命令，更新`org1`的锚节点：

```bash
# channel update 更新通道操作，-o 指定 orderer 节点地址，-c 指定通道ID，-f 指定锚节点配置更新文件路径 --tls 启用TLS，--cafile 指定tlsca证书的文件路径
peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls true --cafile $ORDERER_CA
```

执行完成后，会有如下输：

```bash
2019-07-26 13:17:34.520 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-07-26 13:17:34.693 UTC [channelCmd] update -> INFO 002 Successfully submitted channel update
```

#### 4.3.2 更新`Org2`的锚节点

配置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org2MSP"
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
```

执行下面命令，更新`org2`的锚节点，执行完成后，输出内容与上面的相同：

```bash
peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls true --cafile $ORDERER_CA
```

## 5 安装、实例化、执行链码

### 5.1 安装链码

如果需要通过某个`peer`节点访问和执行`chancode`，那个这个节点上必须要先安装这个`chaincode`。这里我们将在`peer0.org1.example.com`和`peer0.org2.example.com`两个节点上安装链码

#### 5.1.1 在 `peer0.org1.example.com`上安装链码

首先设置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org1MSP"
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
```

然后执行下面命令，安装链码：

```bash
# chaincode install 安装链码，-n 指定链码名称，-v 指定链码版本，-l 指定链码的语言，-p 指定链码文件夹的路径
peer chaincode install -n mycc -v 1.0 -l golang -p "github.com/chaincode/chaincode_example02/go/"
```

安装完成之后，会有如下输出：

```bash
2019-07-26 13:36:08.689 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2019-07-26 13:36:08.689 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2019-07-26 13:36:09.486 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK" >
```

#### 5.1.2 在 `peer0.org2.example.com`上安装链码

首先设置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org2MSP"
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
```

然后执行下面命令，安装链码，安装完成之后，输出内容与上相同

```bash
peer chaincode install -n mycc -v 1.0 -l golang -p "github.com/chaincode/chaincode_example02/go/"
```

### 5.2 实例化链码

`chaincode`安装后需要实例化才能使用，在`channel`中，一个`chaincode`只需要进行一次实例化(`instantiate`)操作即可。

我们在`peer0.org2.example.com`节点上实例化链码，所以先配置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org2MSP"
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051
```

执行下面命令，实例化链码：

```bash
# chaincode instantiate 实例化链码，-o 指定orderer节点地址，-C 指定通道ID，-n 指定链码名称吗，-v 指定链码版本，-c 指定调用链码 init 方法时传入的参数，-P 指定背书策略，这里是策略是需要org1或者org2的成员同意 --tls 启用 TLS --cafile 指定tlsca证书文件路径
peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR ('Org1MSP.member','Org2MSP.member')" --tls true --cafile $ORDERER_CA
```

执行完成之后，只有如下内容输出：

```bash
2019-07-26 13:57:14.619 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2019-07-26 13:57:14.619 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
```

### 5.3 调用链码

经过上面的所有操作，我们的网络环境和链码都已经配置完成了，下面我们就可以对链码执行`query`和`invoke`操作了。

**查询`a`账户的余额**

```bash
# chaincode query 链码查询操作
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

得到输出结果为`100`

**查询`b`账户的余额**

```bash
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","b"]}'
```

得到输出结果为`200`

上面的两个查询命令会获取经背书的`chaincode`执行结果。但它不会产生交易`transaction`

**`a`账户向`b`账户转账`10`元**

```bash
# chaincode invoke 执行链码 invoke 方法，-o 指定orderer节点地址，--tls 启动TLS，--cafile 指定tlsca的证书文件路径 -C 指定通道名称，-n 指定链码名称，-peerAddress 指定需要连接到的peer节点地址，--tlsRootCertFiles  如果启用了TLS，则指向需要连接的peer节点的TLS根证书文件路径 -c 执行invoke方法传入的参数
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile $ORDERER_CA -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}'
```

执行完成之后，会有如下输出：

```bash
2019-07-26 14:29:37.209 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 001 Chaincode invoke successful. result: status:200
```

接着我们可以再次执行上面的查询`a`和`b`余额的命令，查询现在的余额，经过查询，可以发现现在`a`余额为`90`，`b`余额为`210`。

我们还可以尝试连接到`peer1.org1.example.com`上进行查询操作，具体步骤如下：

先配置环境变量：

```bash
export CORE_PEER_LOCALMSPID="Org1MSP"
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer1.org1.example.com:8051
```

执行查询链码操作：

```bash
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

执行完成之后，我们会发现返回了如下错误：

```bash
Error: endorsement failure during query. response: status:500 message:"cannot retrieve package for chaincode mycc/1.0, error open /var/hyperledger/production/chaincodes/mycc.1.0: no such file or directory"
```

这个原因就是我们没有在`peer1.org1.example.com`节点上安装链码，所以我们无法在该节点上执行调用链码的操作，所以我们需要先再该节点上安装链码，具体命令如下：

```bash
peer chaincode install -n mycc -v 1.0 -l golang -p "github.com/chaincode/chaincode_example02/go/"
```

然后我们再进行查询操作：

```bash
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

我们可以看到，成功查询出了`a`账户的余额为`90`

我们可以使用`exit`命令退出`cli`容器，光标回到了主机下面。经过上面调用链码等操作后，我们再次使用`docker ps`命令来查看后台运行的容器服务情况如下：

```bash
CONTAINER ID        IMAGE                                                                                                  COMMAND                  CREATED             STATUS              PORTS                      NAMES
9096320344ee        dev-peer1.org1.example.com-mycc-1.0-cd123150154e6bf2df7ce682e0b1bcbea40499416f37a6da3aae14c4eb51b08d   "chaincode -peer.add…"   48 seconds ago      Up 46 seconds                                  dev-peer1.org1.example.com-mycc-1.0
e60c9a77b244        dev-peer0.org1.example.com-mycc-1.0-384f11f484b9302df90b453200cfb25174305fce8f53f4e94d45ee3b6cab0ce9   "chaincode -peer.add…"   20 minutes ago      Up 20 minutes                                  dev-peer0.org1.example.com-mycc-1.0
b685dd5d864c        dev-peer0.org2.example.com-mycc-1.0-15b571b3ce849066b7ec74497da3b27e54e0df1345daff3951b94245ce09c42b   "chaincode -peer.add…"   44 minutes ago      Up 44 minutes                                  dev-peer0.org2.example.com-mycc-1.0
7ddf52548cfd        hyperledger/fabric-tools:latest                                                                        "/bin/bash"              3 hours ago         Up 3 hours                                     cli
a495b24483dc        hyperledger/fabric-orderer:latest                                                                      "orderer"                3 hours ago         Up 3 hours          0.0.0.0:7050->7050/tcp     orderer.example.com
5349a515df51        hyperledger/fabric-peer:latest                                                                         "peer node start"        3 hours ago         Up 3 hours          0.0.0.0:10051->10051/tcp   peer1.org2.example.com
a8a579ebd979        hyperledger/fabric-peer:latest                                                                         "peer node start"        3 hours ago         Up 3 hours          0.0.0.0:7051->7051/tcp     peer0.org1.example.com
5614be188b35        hyperledger/fabric-peer:latest                                                                         "peer node start"        3 hours ago         Up 3 hours          0.0.0.0:9051->9051/tcp     peer0.org2.example.com
d6672dcc70d4        hyperledger/fabric-peer:latest                                                                         "peer node start"        3 hours ago         Up 3 hours          0.0.0.0:8051->8051/tcp     peer1.org1.example.com
```

我们可以看到，相比于之前，现在多了三个容器，不难看出，这三个容器都是链码容器。在节点的`chaincode`第一次被实例化或使用激活时，会启用一个容器来运行`chaincode`。

### 5.4 其他`Channel`、`Chaincode`操作

我们还是先执行`docker exec -it cli bash`进入`cli`容器内进行操作

- 列出当前节点所加入的`Channel`:

  ```bash
  peer channel list
  ```

  输出内容为：

  ```bash
  2019-07-26 16:02:11.361 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
  Channels peers has joined:
  mychannel
  ```

- 列出当前节点所有已经安装的`Chaincode`:

  ```bash
  peer chaincode list --installed
  ```

  输出结果为：

  ```bash
  Get installed chaincodes on peer:
  Name: mycc, Version: 1.0, Path: github.com/chaincode/chaincode_example02/go/, Id: 333a19b11063d0ade7be691f9f22c04ad369baba15660f7ae9511fd1a6488209
  ```

- 获取特定`Channel`上的区块信息：

  ```bash
  peer channel getinfo -c mychannel
  ```

  输出结果为：

  ```bash
  2019-07-26 16:06:02.085 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
  Blockchain info: {"height":6,"currentBlockHash":"kIw2CHcxYZU3gDwZUH7l9qN/m0Sn8DXXJ8ra7Z5O9Jk=","previousBlockHash":"e3+BrzDCedbzz+zvcW3E+tCXG+DE4C0KOerbUxU7h8E="}
  ```

- 进一步获取特定高度的`Block`详细内容，例如，下面是获取最后一块（从`0`开始编号）`Block`的详细内容

  ```bash
  peer channel fetch 5 mychannel_5.block -o orderer.example.com:7050 -c mychannel --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
  ```

  运行完成后，会有如下输出：

  ```bash
  2019-07-26 16:12:50.014 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
  2019-07-26 16:12:50.017 UTC [cli.common] readBlock -> INFO 002 Received block: 5
  ```

  我们使用`ll`查看当前目录下，会发现多了一个`mychannel_5.block`文件，我们接下来就可以使用`configtcgen`工具将这个区块文件解析成`json`格式的文件了，具体命令如下：

  ```bash
  configtxgen --inspectBlock mychannel_5.block > channel-artifacts/mychannel_5.json
  ```

  运行完成后，有如下输出：

  ```bash
  2019-07-26 16:13:07.306 UTC [common.tools.configtxgen] main -> INFO 001 Loading configuration
  2019-07-26 16:13:07.631 UTC [common.tools.configtxgen.localconfig] completeInitialization -> INFO 002 Orderer.Addresses unset, setting to [127.0.0.1:7050]
  2019-07-26 16:13:07.631 UTC [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 orderer type: solo
  2019-07-26 16:13:07.631 UTC [common.tools.configtxgen.localconfig] LoadTopLevel -> INFO 004 Loaded configuration: /etc/hyperledger/fabric/configtx.yaml
  2019-07-26 16:13:07.631 UTC [common.tools.configtxgen] doInspectBlock -> INFO 005 Inspecting block
  2019-07-26 16:13:07.631 UTC [common.tools.configtxgen] doInspectBlock -> INFO 006 Parsing genesis block
  ```

  我们在主机的`/opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/channel-artifacts`目录下就能看到刚才解析出来的`json`格式的区块文件`mychannel_5.json`。
  
#### 5.5 关闭网络
  
  如果我们需要将刚才搭建出来的网络关闭，并且清除一些环境信息等，我们可以使用`first-network`提供了脚本进行关闭即可，具体操作如下：
  
  ```bash
  # 首先来到 first-network 目录下
  cd /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network
  # 执行关闭网络的脚本命令
  ./byfn.sh down
  ```
  
  输入命令后，接着会让选择是否继续，我们输入`y`继续执行。执行完成后，会有如下输出：
  
  ````bash
  [root@localhost first-network]# ./byfn.sh down
  Stopping for channel 'mychannel' with CLI timeout of '10' seconds and CLI delay of '3' seconds
  Continue? [Y/n] y
  proceeding ...
  WARNING: The BYFN_CA2_PRIVATE_KEY variable is not set. Defaulting to a blank string.
  WARNING: The BYFN_CA1_PRIVATE_KEY variable is not set. Defaulting to a blank string.
  Stopping cli                    ... done
  Stopping orderer.example.com    ... done
  Stopping peer1.org2.example.com ... done
  Stopping peer0.org1.example.com ... done
  Stopping peer0.org2.example.com ... done
  Stopping peer1.org1.example.com ... done
  Removing cli                    ... done
  Removing orderer.example.com    ... done
  Removing peer1.org2.example.com ... done
  Removing peer0.org1.example.com ... done
  Removing peer0.org2.example.com ... done
  Removing peer1.org1.example.com ... done
  Removing network net_byfn
  Removing volume net_peer0.org3.example.com
  WARNING: Volume net_peer0.org3.example.com not found.
  Removing volume net_peer1.org3.example.com
  WARNING: Volume net_peer1.org3.example.com not found.
  Removing volume net_orderer2.example.com
  WARNING: Volume net_orderer2.example.com not found.
  Removing volume net_orderer.example.com
  Removing volume net_peer0.org2.example.com
  Removing volume net_peer0.org1.example.com
  Removing volume net_peer1.org1.example.com
  Removing volume net_peer1.org2.example.com
  Removing volume net_orderer5.example.com
  WARNING: Volume net_orderer5.example.com not found.
  Removing volume net_orderer4.example.com
  WARNING: Volume net_orderer4.example.com not found.
  Removing volume net_orderer3.example.com
  WARNING: Volume net_orderer3.example.com not found.
  9096320344ee
  e60c9a77b244
  b685dd5d864c
  Untagged: dev-peer1.org1.example.com-mycc-1.0-cd123150154e6bf2df7ce682e0b1bcbea40499416f37a6da3aae14c4eb51b08d:latest
  Deleted: sha256:bffb85af26a7def57aa707fa85b99b59b966080d7964823004010f082a0ad198
  Deleted: sha256:48c94dc3357616bc7c38a70644e9abf4be2df1a00709386ad7d7c98f281a4ea9
  Deleted: sha256:506f39276ec553d0eb2b152dbac19e284f4ef90431213e377acbdd77dbaf1702
  Deleted: sha256:0a59238e1a3fe6dd87e5f401063534937136382e1f21c61a8db71143f7967d84
  Untagged: dev-peer0.org1.example.com-mycc-1.0-384f11f484b9302df90b453200cfb25174305fce8f53f4e94d45ee3b6cab0ce9:latest
  Deleted: sha256:3bff91b1dec74033b6be007f24535c89a8a40bc1b12eab128d90e1b2d9026eb7
  Deleted: sha256:8fd2fc3bfd9a9a0dae331d155600a442d11a61428806feb5a2efc86dbd264e84
  Deleted: sha256:a2aa63cc2b7c032094d1afe69fddc4125855d1c072cb4dacdbf408fc34f5378e
  Deleted: sha256:ba6b3377507dc8417b2742fb30b494a4a70341665ecd24f34e5dd51f97c9be28
  Untagged: dev-peer0.org2.example.com-mycc-1.0-15b571b3ce849066b7ec74497da3b27e54e0df1345daff3951b94245ce09c42b:latest
  Deleted: sha256:1d073a8218d632d77ccb818dc1be159baf0ebdbd7150e603d6bb84e9a5ab91b8
  Deleted: sha256:253c4f60d83ae040fb3a258cb755f7d24e1ae3bc55e0dbab4f75f41f71104c81
  Deleted: sha256:a2cc738b0997ece4ca4f6b568b2a3477838e9c715cdb31c2e0e65e40656f0dae
  Deleted: sha256:876947d888edf4b1f8863275afd249f210d0723029e3ffad27d68ef9b12bcc42
  ````
  
  关闭网络后，我们可以使用`docker ps -a`命令查看后台是否存在运行着的或关闭的容器，我们可以发现，所有容器清除完成了。

## 6 参考

- [分步详解 Fabric 区块链网络的部署](https://www.ibm.com/developerworks/cn/cloud/library/cl-lo-hyperledger-fabric-practice-analysis/index.html)

- [fabric主要配置文件细讲](https://blog.csdn.net/qq_25870633/article/details/81184781)

- [Hyperledger Fabric相关文件解析](https://www.cnblogs.com/cbkj-xd/p/11153080.html)

- [深入解析Hyperledger Fabric搭建的全过程](https://www.cnblogs.com/cbkj-xd/p/11067810.html)

- [Docker — 从入门到实践](https://docker_practice.gitee.io/)
