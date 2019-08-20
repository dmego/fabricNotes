# 使用 Chaincode 开发模式调试链码

 `Fabirc` 提供了链码的开发模式，使链码在开发阶段可以进行非常快速的开发、构建、运行、调试。具体步骤如下：

- 将已经编写好的`chaincode` 拷贝到 `/opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/chaincode`目录中

- 新建一个终端，启动 `chaincode-docker-devmode` 开发模式网络

  ```bash
  # 进入 chaincode-docker-devmode 目录
  cd /opt/gopath/src/github.com/hyperledger/fabric/scripts/fabric-samples/chaincode-docker-devmode
  # 启动网络
  docker-compose -f docker-compose-simple.yaml up
  ```

- 新建一个终端，编译链码，启动 `chaincode`

  ```bash
  # 进入 chaincode 容器
  docker exec -it chaincode bash
  cd abs # 进入链码目录
  go build # 编译链码
  
  CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=absc:0 ./abs # 运行 chaincode
  ```

- 新建一个终端，安装，实例化，测试链码

  ```bash
  # 进入 chaincode 容器
  docker exec -it cli bash
  # 安装链码
  peer chaincode install -p chaincodedev/chaincode/sacc -n absc -v 1.0
  # 实例化链码
  peer chaincode instantiate -n absc -v 1.0 -c '{"Args":[]}' -C myc
  # 调用链码
  peer chaincode invoke -n absc -c '{"Args":["addowner", "owner01", "Owner","341412197607041122","1G1BL52P7TR115520"]}' -C myc
  # 调用链码
  peer chaincode invoke -n absc -c '{"Args":["getbyid", "owner01"]}' -C myc
  ```
