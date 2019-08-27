# Fabric 中动态加入节点

 >注：测试 `Fabric` 版本为 `1.4.2`；测试网络配置为`first-network`

## 1. 启动测试网络

首先我们需要进入`first-network`文件夹下，启动测试网络：

```bash
# 进入 first-network 文件夹
cd /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/first-network/
# 启动网络
./byfn.sh up
```

## 2. 修改`crypto-config.yaml`文件

网络启动完成后，我们就可以来测试添加一个节点到网络中了，首先我们需要修改`crypto-config.yaml`文件，添加一个新节点，然后在为这个新节点使用`cryptogen`工具生成其组织关系证书，这只是在测试网络中这样做，如果是在生产环境中，将会使用`Fabric CA`来生成新节点的组织关系证书。

修改的`crypto-config.yaml`文件位置如下：

```yaml
# ---------------------------------------------------------------------------
# Org2: See "Org1" for full specification
# ---------------------------------------------------------------------------
- Name: Org2
  Domain: org2.example.com
  EnableNodeOUs: true
  Template:
    Count: 3 # 将此处改为 3，表示向组织2中新增加一个peer节点（peer2.org2.example.com）
  Users:
    Count: 1
```

## 3. 生成新节点的组织关系证书

我们知道`crypto-config.yaml`配置文件是用来生成各种组织关系证书的，我们使用这个文件和`ctyptogen`就能为新节点生成他的组织关系证书，但是使用的参数不一样，具体命令如下：

```bash
# 使用 extend 参数表示扩展原先的组织关系证书，也就是增加新的组织关系证书
cryptogen extend --config=./crypto-config.yaml
```

命令执行完后没有任何输出，我们可以使用命令`tree crypto-config -L 4`来查看`crypto-config`的目录结构，可以看到在`org2.example.com`的`peers`目录下多了一个`peer2.org2.example.com`目录。

## 4. 修改`docker-compose-base.yaml`文件

生成新节点的组织关系证书后，我们接下来需要的为新节点编写`YAML`启动配置文件，首先修改`docker-compose-base.yaml`文件，在最后新增新节点的配置，具体新增内容如下：

```yaml
peer2.org2.example.com:
    container_name: peer2.org2.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer2.org2.example.com
      - CORE_PEER_ADDRESS=peer2.org2.example.com:11051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:11051
      - CORE_PEER_CHAINCODEADDRESS=peer2.org2.example.com:11052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:11052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer2.org2.example.com:11051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.example.com:9051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ../crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls:/etc/hyperledger/fabric/tls
        - peer2.org2.example.com:/var/hyperledger/production
    ports:
      - 11051:7051
```

## 5. 编辑新节点的`docker-compsoe`启动文件

下一步就是为新节点新建一个`docker-compose`启动文件，用来启动新节点，我们可以先使用如下命令新建一个`YAML`文件：

```bash
vim docker-compose-peer2org2.yaml
```

然后再在文件中添加如下配置内容：

```yaml
version: '2'

volumes:
  peer2.org2.example.com:

networks:
  byfn:
  
services:
  peer2.org2.example.com:
    extends:
      file:   base/docker-compose-base.yaml
      service: peer2.org2.example.com
    container_name: peer2.org2.example.com
    networks:
      - byfn
```

## 6. 启动新节点

证书文件和配置文件都准备就绪后，我们就可以启动该新节点了，启动命令如下：

```bash
docker-compose -f docker-compose-peer2org2.yaml up -d
```

执行完成后，我们使用`docker ps`命令会发现后台多了一个`peer2.org2.example.com`容器。

## 7. 进入`cli`客户端容器，将新节点加入通道

新节点已经启动了，但是它和我们第一步启动起来的网络没有任何关系，我们需要将新节点加入到我们的通道中来，具体步骤如下：

首先使用如下命令今天`cli`客户端容器：

```bash
docker exec -it cli bash
```

然后添加入下的环境变量，将客户端与`peer2.org2`连接：

```bash
export CHANNEL_NAME=mychannel
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
export CORE_PEER_ADDRESS=peer2.org2.example.com:11051
export ORDERER_CA=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

从`orderer`节点上把通道配置`block`拉取下来：

```bash
# 把创世区块拉取下来
peer channel fetch oldest mychannel.block -c mychannel -o orderer.example.com:7050 --tls --cafile $ORDERER_CA
# 相当于下面的命令，第 0 块就是创世区块
peer channel fetch 0 mychannel.block -c mychannel -o orderer.example.com:7050 --tls --cafile $ORDERER_CA
```

我们可以看到执行完成后，会有如下输出：

```bash
2019-08-26 17:59:26.909 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-08-26 17:59:26.912 UTC [cli.common] readBlock -> INFO 002 Received block: 0
```

最后执行如下命令，将新节点`peer2.org2.example.com`加入到通道`mychannel`中来：

```bash
peer channel join -b mychannel.block -o orderer.example.com:7050
```

执行完成之后，会有如下输出：

```bash
2019-08-26 17:59:35.260 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2019-08-26 17:59:35.303 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

## 8. 在新节点上安装链码并调用链码

新节点加入测试网络后，我们就可以在该节点上安装链码，执行各种操作了。

首先执行下面命令安装链码：

```bash
peer chaincode install -n mycc -v 1.0 -p github.com/chaincode/chaincode_example02/go/
```

安装成功后，会有如下输出：

```bash
2019-08-26 18:03:53.009 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2019-08-26 18:03:53.010 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2019-08-26 18:03:53.489 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK" >
```

然后执行下面命令，进行转账操作：

```bash
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:9051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["invoke","a","b","10"]}'
```

最后执行下面命令查询`a`账户余额，结果为`80`，说明新节点成功加入了网络，并且调用链码各种操作正在。

```bash
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```
