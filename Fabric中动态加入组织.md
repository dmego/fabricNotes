# Fabric 中动态加入组织

> 注：测试 `Fabric` 版本为 `1.4.2`；测试网络配置为`first-network`

## 1. 启动测试网络

首先我们需要进入`first-network`文件夹下，启动测试网络：

```bash
# 进入 first-network 文件夹
cd /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/

# 启动网络
./byfn.sh up
```

## 2. 为新组织生成证书

其实`first-network`直接提供了添加新组织的脚本和配置文件，这里我们不需要编写配置，只需要解读这些配置内容就行。

首先，我们需要使用`org3-artifacts/org3-crypto.yaml`文件来为新组织`org3`生成他的组织关系证书文件，`org3-crypto.yaml`文件内容如下：

```bash
# ---------------------------------------------------------------------------
# "PeerOrgs" - Definition of organizations managing peer nodes
# ---------------------------------------------------------------------------
PeerOrgs:
  # ---------------------------------------------------------------------------
  # Org3
  # ---------------------------------------------------------------------------
  - Name: Org3
    Domain: org3.example.com
    EnableNodeOUs: true
    Template:
      Count: 2 # 新组织 org3 有两个 peer 节点（peer0.org3,peer1.org3）
    Users:
      Count: 1
```

我们使用如下命令，来生成`org3`的组织关系证书文件：

```bash
cd org3-artifacts/
cryptogen generate --config=./org3-crypto.yaml
```

执行完成后，我们在`org3-artifacts`目录下就能看到`crypto-config`文件夹，里面就有新生成的`org3`的组织关系证书文件。

## 3. 为新组织生成配置文件

接下来使用`org3-artifacts`下的`configtx.yaml`文件和`configtxgen`工具为`org3`生成`json`格式的配置文件。其中`configtx.yaml`文件内容如下：

```bash
Organizations:
    - &Org3 # 添加一个新组织 org3
        Name: Org3MSP
        ID: Org3MSP
        MSPDir: crypto-config/peerOrganizations/org3.example.com/msp

        # 策略定义，如果配置文件中没有，建议添加这些配置
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org3MSP.admin', 'Org3MSP.peer', 'Org3MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org3MSP.admin', 'Org3MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org3MSP.admin')"

        # 锚节点定义
        AnchorPeers:
            - Host: peer0.org3.example.com
              Port: 11051
```

使用如下命令来生成配置文件：

```bash
export FABRIC_CFG_PATH=$PWD
configtxgen -printOrg Org3MSP -profile ./configtx.yaml > ../channel-artifacts/org3.json
```

命令执行完成之后，我们会发现在`channel-artifacts`文件夹下多了一个`org3.json`文件。

将 `Orderer`的组织证书文件拷贝到`org3-artifacts/crypto-config`下来，以便接下来的工作，同时也将`org3-artifacts/crypto-config/peerOrganizations/org3.example.com`文件夹拷贝到`crypto-config/peerOrganizations/`目录下来，以便`cli`客户端节点进行最后的测试操作，具体命令如下：

```bash
# 返回到 first-network 目录
cd ..

# 将 Orderer的组织证书文件拷贝到 org3-artifacts/crypto-config
cp -r crypto-config/ordererOrganizations org3-artifacts/crypto-config/

# 将 org3.example.com 文件夹拷贝到 crypto-config/peerOrganizations
cp -r org3-artifacts/crypto-config/peerOrganizations/org3.example.com crypto-config/peerOrganizations/
```

## 4. 生成和提交新组织的配置

通过上面两步，我们已经生成了`org3`的证书文件和配置，但是还没有与区块链网络进行关联，接下来，我们就需要为网络中的`channel`添加一个新的组织进来。

### 4.1 进入`cli`容器，获取最新的`channel`配置

进入`cli`容器，先设置`Orderer CA`和`channel`的环境变量

```bash
# 进入 cli 容器
docker exec -it cli bash

# 设置`Orderer CA`和`channel`的环境变量
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CHANNEL_NAME=mychannel
```

拉取当前最新的配置区块，它会吧`channel`的配置保存到`protobuf`文件中去，具体命令如下：

```bash
# peer channel fetch config 拉取最新的 channel 配置区块文件，保存到 config_block.pb 文件中
peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```

命令执行完成之后，我们可以看到有如下输出：

```bash
2019-08-26 19:35:18.275 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-08-26 19:35:18.310 UTC [cli.common] readBlock -> INFO 002 Received block: 4
2019-08-26 19:35:18.312 UTC [cli.common] readBlock -> INFO 003 Received block: 2
```

通过输出日志我们发现，当前共有`4`个区块，其中最新的`channel`配置区块在第`2`块的位置，其中前几块区块主要做了如下的配置更改：

- 区块`0`：使用应用`channel`创世区块创建应用`channel`

- 区块`1`：更新`org1`的锚节点

- 区块`2`：更新`org2`的锚节点

  所以，我们通过上面的命令，得到了区块`2`中最新的配置。但是我们得到的配置是`protobuf`格式的，我们需要将其转换为可读的`JSON`格式，这里需要使用两个工具，一个是`configtxlator`，它是一个可以将系统所需的配置文件，在二进制(例如`protobuf`)格式和`JSON`格式之间进行转换的工具，方便用户更新`channel`的配置。另一个工具是`jq`，它是一个`JSON`命令行处理工具，可以对 `JSON` 进行过滤、格式化、修改等操作。

  `configtxlator`工具之前已经安装在`$GOPATH`目录下了，执行下面命令安装`JQ`：

  ```bash
  apt-get -y update && apt-get -y install jq
  ```

  使用下面命令，将配置相关的数据从`config_block.pb`中读取出来，并存在`config.json`文件中：

  ```bash
  # proto_decode 解码 pb 格式文件
  configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
  ```

### 4.2 利用`channel`配置和`org3`配置生成更新`channel`的配置

上一步生成的`config.json`文件包含了`org1`和`org2`的信息，现在要把`org3.json`加入到`config.json`中：

```bash
jq -s '.[0] * {"channel_group":{"groups":{"Application":{"groups": {"Org3MSP":.[1]}}}}}' config.json ./channel-artifacts/org3.json > modified_config.json
```

`jq -s`指定了格式，`.[0]，.[1]`代表了第`0`，`1`个参数，命令执行的效果就是增加了和`org1`和`org2`平级的配置，然后将这个新的配置保存到文件`modified_config.json`中。我们可以使用`diff config.json modified_config.json`来进行对比。

接下来为了生成`channel`的配置更新文件，我们需要做下面的操作：

- `config.json --> config.pb`
- `modified_config.json --> modified_config.pb`
- `modified_config.pb - config.pb --> org3_update.pb`

命令如下：

```bash
# proto_encode 是将 json 格式编码成 pb 格式文件；config.json --> config.pb
configtxlator proto_encode --input config.json --type common.Config --output config.pb

# modified_config.json --> modified_config.pb
configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb

# compute_update 是比较两个pb格式文件的，计算出增量；modified_config.pb - config.pb --> org3_update.pb
configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_config.pb --output org3_update.pb
```

但是，`org3_update.pb`不能直接用来升级，需要再在它外面封装一层，但是，封装需要使用`JSON`来处理，具体封装过程的命令如下：

```bash
# proto_decode 解码成 json 格式文件
configtxlator proto_decode --input org3_update.pb --type common.ConfigUpdate | jq . > org3_update.json

# 再封装一层
echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat org3_update.json)'}}}' | jq . > org3_update_in_envelope.json

# 编码成pb格式文件
configtxlator proto_encode --input org3_update_in_envelope.json --type common.Envelope --output org3_update_in_envelope.pb
```

现在，生成的`org3_update_in_envelope.pb`文件就是用来升级的。

### 4.3 对更新`channel`配置进行签名

虽然我们现在生成了配置更新文件`org3_update_in_envelope.pb`，但是我们还不直接拿这个文件来升级，因为对于`Fabric`网络而言，更新`channel`配置是以一笔交易形式提交的。要完成这笔交易，将`org3`的配置成功更新，我们就必须需要权限认证。

通道到默认修改策略是`MOJORITY`，也就是需要多数`org`的`Admin`账号进行签名。也就是说，通道所有组织都需要对这个文件进行签名。现在启动的`Fabric`网络中只有两个`org`，所以只需要这个两个组织的签名就行。

由于`cli`中的环境变量设置的就是`org1`的，所以可以直接进行签名：

```bash
peer channel signconfigtx -f org3_update_in_envelope.pb
```

接下来，设置`org2`的环境变量，再次对文件进行签名：

```bash
# 设置环境变量
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051

# 对文件进行签名
peer channel signconfigtx -f org3_update_in_envelope.pb
```

### 4.4 发送更新`channel`配置的交易

所有组织管理员签名完成之后，我们就可以发送更新`channel`配置的交易了，使用如下命令：

```bash
peer channel update -f org3_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls --cafile $ORDERER_CA
```

执行完成后，有如下输出：

```bash
2019-08-26 21:17:26.330 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-08-26 21:17:26.400 UTC [channelCmd] update -> INFO 002 Successfully submitted channel update
```

### 4.5 配置选举以接收区块

新加入的组织节点只能使用创世区块启动，创世区块不包含他们已经加入到通道的配置，所以他们无法利用`gossip`验证他们从本组织其他`peer`节点发来到区块，直到他们接收到了他们加入通道的配置交易。所以新加入的节点必须配置他们从哪接收区块的`orderer`排序节点。

如果使用的静态`leader`模式，使用如下配置：

```bash
CORE_PEER_GOSSIP_USELEADERELECTION=false
CORE_PEER_GOSSIP_ORGLEADER=true
```

如果使用的是动态`leader`模式，使用如下配置：

```bash
CORE_PEER_GOSSIP_USELEADERELECTION=true
CORE_PEER_GOSSIP_ORGLEADER=false
```

这样新组织的`peer`都宣称自己是`leader`节点，加速了获取区块，等他们接收到它们自己的配置交易，就会只有`1`个`leader` 节点代表新组织。

## 5. 将新组织添加到通道中

### 5.1 使用`docker-compose`文件启动新组织的节点容器

我们在`first-network`文件夹下可以看到`docker-compose-org3.yaml`文件，这个文件里定义了新组织的节点容器配置等各种信息，具体内容如下：

```yaml
version: '2'

volumes:
  peer0.org3.example.com:
  peer1.org3.example.com:

networks: # 网络定义
  byfn:

services:

  peer0.org3.example.com: # peer0.org3.example.com 容器服务
    container_name: peer0.org3.example.com
    extends:
      file: base/peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org3.example.com
      - CORE_PEER_ADDRESS=peer0.org3.example.com:11051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:11051
      - CORE_PEER_CHAINCODEADDRESS=peer0.org3.example.com:11052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:11052
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org3.example.com:12051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org3.example.com:11051
      - CORE_PEER_LOCALMSPID=Org3MSP
    volumes:
        - /var/run/:/host/var/run/
        - ./org3-artifacts/crypto-config/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/msp:/etc/hyperledger/fabric/msp
        - ./org3-artifacts/crypto-config/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls:/etc/hyperledger/fabric/tls
        - peer0.org3.example.com:/var/hyperledger/production
    ports:
      - 11051:11051
    networks:
      - byfn

  peer1.org3.example.com: # peer1.org3.example.com 容器服务
    container_name: peer1.org3.example.com
    extends:
      file: base/peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org3.example.com
      - CORE_PEER_ADDRESS=peer1.org3.example.com:12051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:12051
      - CORE_PEER_CHAINCODEADDRESS=peer1.org3.example.com:12052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:12052
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org3.example.com:11051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org3.example.com:12051
      - CORE_PEER_LOCALMSPID=Org3MSP
    volumes:
        - /var/run/:/host/var/run/
        - ./org3-artifacts/crypto-config/peerOrganizations/org3.example.com/peers/peer1.org3.example.com/msp:/etc/hyperledger/fabric/msp
        - ./org3-artifacts/crypto-config/peerOrganizations/org3.example.com/peers/peer1.org3.example.com/tls:/etc/hyperledger/fabric/tls
        - peer1.org3.example.com:/var/hyperledger/production
    ports:
      - 12051:12051
    networks:
      - byfn


  Org3cli: # 定义一个新组织专用 cli 容器服务，用来将新组织加入通道等操作
    container_name: Org3cli
    image: hyperledger/fabric-tools:$IMAGE_TAG
    tty: true
    stdin_open: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - FABRIC_LOGGING_SPEC=INFO
      #- FABRIC_LOGGING_SPEC=DEBUG
      - CORE_PEER_ID=Org3cli
      - CORE_PEER_ADDRESS=peer0.org3.example.com:11051
      - CORE_PEER_LOCALMSPID=Org3MSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ./../chaincode/:/opt/gopath/src/github.com/chaincode
        - ./org3-artifacts/crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ./crypto-config/peerOrganizations/org1.example.com:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com
        - ./crypto-config/peerOrganizations/org2.example.com:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com
        - ./scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/
    depends_on:
      - peer0.org3.example.com
      - peer1.org3.example.com
    networks:
      - byfn
```

我们使用如下命令来启动新组织中的节点容器和`cli`容器：

```bash
docker-compose -f docker-compose-org3.yaml up -d
```

启动成功后，我们使用`docker ps`命令可以看到，在后台多了三个容器在运行，分别是：`peer0.org3.example.com`、`peer1.org3.example.com`和`Org3cli`。

### 5.2 拉取`channel`的创世区块

接着我们进入`Org3cli`容器内，将新组织的两个节点加入到通道中来，首先是先设置环境变量，拉取通道的创世区块文件，具体命令如下：

```bash
#进入 Org3cli 容器
docker exec -it Org3cli bash

# 设置环境变量
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CHANNEL_NAME=mychannel

# 拉取创世区块文件
peer channel fetch 0 mychannel.block -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```

拉取结束后，会有如下输出：

```bash
2019-08-26 21:40:17.146 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-08-26 21:40:17.151 UTC [cli.common] readBlock -> INFO 002 Received block: 0
```

### 5.3 新组织节点加入通道

最后我们需要将新组织的两个节点`peer0`和`peer1`加入到通道中去，具体命令如下：

```bash
# 设置 peer0.org3.example.com 的环境变量
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_ADDRESS=peer0.org3.example.com:11051

# 将 peer0.org3.example.com 加入到通道中
peer channel join -b mychannel.block

# 设置 peer1.org3.example.com 的环境变量
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer1.org3.example.com/tls/ca.crt
export CORE_PEER_ADDRESS=peer1.org3.example.com:12051

# 将 peer1.org3.example.com 加入到通道中
peer channel join -b mychannel.block
```

加入到通道中的操作都会返回如下输出结果：

```bash
2019-08-26 21:46:12.765 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-08-26 21:46:12.801 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

## 6. 升级链码和背书策略

新组织节点加入通道中了，我们之前链码就需要进行升级，因为背书策略需要升级，新组织需要参与到背书中来。

我们首先新建一个终端，执行如下命令进入`cli`容器：

```bash
docker exec -it cli bash

# 设置`Orderer CA`和`channel`的环境变量
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CHANNEL_NAME=mychannel
```

由于之前我们已经将新组织的各种证书拷贝到了网络的证书文件夹目录，这里可以执行下面的命令在`org3`上安装升级新链码：

```bash
# 设置环境变量
export CORE_PEER_LOCALMSPID="Org3MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
export CORE_PEER_ADDRESS=peer0.org3.example.com:11051

# 升级链码
peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go
```

执行下面的命令在`org2`上安装升级新链码：

```bash
# 设置环境变量
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer0.org2.example.com:9051

# 升级链码
peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go
```

执行下面的命令在`org1`上安装升级新链码：

```bash
# 设置环境变量
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051

# 升级链码
peer chaincode install -n mycc -v 2.0 -p github.com/chaincode/chaincode_example02/go
```

更新升级链码后，我们就需要升级背书策略，具体命令如下：

```bash
# 这里背书策略设置为 AND，也就是说需要三个组织的所有节点都背书
peer chaincode upgrade -o orderer.example.com:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA -C $CHANNEL_NAME -n mycc -v 2.0 -c '{"Args":["init","a","90","b","210"]}' -P "AND ('Org1MSP.peer','Org2MSP.peer','Org3MSP.peer')"
```

## 7. 调用链码测试是否成功

链码升级安装成功后，我们就需要调用链码进行测试，我们将先环境切换到`org3`中来：

```bash
export CORE_PEER_LOCALMSPID="Org3MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp
export CORE_PEER_ADDRESS=peer0.org3.example.com:11051
```

然后执行下面的命令，查询`a`账户余额：

```bash
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```

得到结果为`90`

接着，执行下面命令进行转账操作：

```bash
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt --peerAddresses peer0.org3.example.com:11051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}'
```

这里我们会发现多了好多`--peerAddresses`和`--tlsRootCertFiles`参数，这些参数的设置就和之前设置的背书策略`AND`有关，如果我们背书策略设置为`OR`，就不用设置这些参数了。

转账成功后，我们再次执行命令，查询`a`账户的余额，得到结果为`80`

```bash
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```

## 8. 设置新组织的锚节点(可选)

`org3`的节点能够与`org1`和`org2`的节点建立`gossip`连接，这是由于`org1`和`org2`在通道配置中定义了锚节点，同样，`org3`等新加入的组织为了能够让其他组织的新节点或其他新组织发现`org3`的节点，也必须配置锚节点。

这里更新锚节点的流程与启动网络更新锚节点的方式不太一样，与添加新组织的流程类似，具体如下。

### 8.1 拉取并转换最新的通道配置

我们使用如下命令先进入`org3cli`客户端容器：

```bash
docker exec -it Org3cli bash

# 设置环境变量
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
export CHANNEL_NAME=mychannel
```

使用下面的命令拉取最新的通道配置区块：

```bash
peer channel fetch config config_block.pb -o orderer.example.com:7050 -c $CHANNEL_NAME --tls --cafile $ORDERER_CA
```

获取配置区块之后，我们需要将其转换成`JSON`格式，就像之前添加新组织那样操作。转换时，我们需要删除更新`org3`不需要的所有头、元数据和签名，以便使用`jq`工具将锚节点信息加入进去：

```bash
# 转换成 config.json 文件
configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json
```

### 8.2 将锚节点信息加入到配置文件

使用`jq`工具，将要添加的`org3`锚节点信息添加到`config.json`文件中，并保存为`modified_anchor_config.json`文件：

```bash
jq '.channel_group.groups.Application.groups.Org3MSP.values += {"AnchorPeers":{"mod_policy": "Admins","value":{"anchor_peers": [{"host": "peer0.org3.example.com","port": 11051}]},"version": "0"}}' config.json > modified_anchor_config.json
```

### 8.3 计算更新差量并生成更新配置交易文件

接着，就是计算更新差量，再封装差量，生成配置更新交易文件，具体命令如下：

```bash
# 转换 config.json 为 protobuf 格式
configtxlator proto_encode --input config.json --type common.Config --output config.pb

# 转换 modified_anchor_config.json 为 protobuf 格式
configtxlator proto_encode --input modified_anchor_config.json --type common.Config --output modified_anchor_config.pb

# 计算两个 protobuf 格式化配置之间的差值
configtxlator compute_update --channel_id $CHANNEL_NAME --original config.pb --updated modified_anchor_config.pb --output anchor_update.pb

# 再次使用 configtxlator 命令转换 anchor_update.pb 为 anchor_update.json
configtxlator proto_decode --input anchor_update.pb --type common.ConfigUpdate | jq . > anchor_update.json

# 将更新配置包装再封装一层，恢复先前剥离的标头，将其输出到 anchor_update_in_envelope.json 文件
echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat anchor_update.json)'}}}' | jq . > anchor_update_in_envelope.json

# 现在重新整合了更新配置文件，我们需要将其转换为 protobuf，以便它可以正确签名并提交给orderer节点进行更新
configtxlator proto_encode --input anchor_update_in_envelope.json --type common.Envelope --output anchor_update_in_envelope.pb
```

### 8.4 提交更新配置交易

更新配置交易文件生成成功后，我们就可以使用如下命令来更新配置交易了：

```bash
peer channel update -f anchor_update_in_envelope.pb -c $CHANNEL_NAME -o orderer.example.com:7050 --tls --cafile $ORDERER_CA
```

`orderer`节点接收配置更新请求后会使用新的配置新生成一个配置块，当其他组织的节点收到更新配置块时，将会处理这些配置更新。

### 8.5 检查节点日志，查看是否配置成功

我们可以使用如下命令查看一个节点的日志：

```bash
docker logs -f peer0.org1.example.com
```

可以看到，其中包含了如下内容，说明新组织的锚节点更新配置成功

```bash
2019-08-26 22:35:31.413 UTC [gossip.privdata] StoreBlock -> INFO 10f [mychannel] Received block [8] from buffer
2019-08-26 22:35:31.427 UTC [gossip.gossip] JoinChan -> INFO 110 Joining gossip network of channel mychannel with 3 organizations
2019-08-26 22:35:31.427 UTC [gossip.gossip] learnAnchorPeers -> INFO 111 Learning about the configured anchor peers of Org1MSP for channel mychannel : [{peer0.org1.example.com 7051}]
2019-08-26 22:35:31.428 UTC [gossip.gossip] learnAnchorPeers -> INFO 112 Learning about the configured anchor peers of Org2MSP for channel mychannel : [{peer0.org2.example.com 9051}]
2019-08-26 22:35:31.428 UTC [gossip.gossip] learnAnchorPeers -> INFO 113 Anchor peer with same endpoint, skipping connecting to myself
2019-08-26 22:35:31.428 UTC [gossip.gossip] learnAnchorPeers -> INFO 114 Learning about the configured anchor peers of Org3MSP for channel mychannel : [{peer0.org3.example.com 11051}]
```

## 9. 参考

- [Fabric学习笔记(八) - cli动态添加Org](https://segmentfault.com/a/1190000013521785)
- [动态加入组织到 Channel](https://pinvondev.github.io/blog/BlockChain/2018/08/27/动态加入组织到-channel/%20Or%20/blog/BlockChain/动态加入组织到-channel/)
- [区块链之Fabric（三）动态增删机构](http://wenchao.wang/2018/04/09/区块链之Fabric（三）动态增删机构/)
- [Fabric组织动态加入](http://lessisbetter.site/2019/08/01/fabric-new-org/)
- [官方文档：Adding an Org to a Channel](https://hyperledger-fabric.readthedocs.io/en/latest/channel_update_tutorial.html)
