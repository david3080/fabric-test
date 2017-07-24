## Dockerで、orderer, peer, caを動かしてchaincodeを動かすテストを実施します。

#### 前提条件
- Mac OS X

#### 事前準備
- gitコマンドのインストール
- GO 1.8以上のインストール
- curlコマンドのインストール(brew install curl)
- Dockerのインストール

#### 手順
1. /tmpディレクトリにfabric-testをクローンして、バイナリファイルやdockerイメージを導入します。

```
cd /tmp
git clone https://github.com/david3080/fabric-test.git
cd fabric-test/
curl -sSL https://goo.gl/iX9dek | bash
export PATH=$PWD/bin:$PATH # ダウンロードしたbinフォルダ下のpeerコマンドなどにパスを通します。
```

2. $GOPATHにfabricをcloneします。fabric1.0.0ではこれをしないと1でダウンロードしたpeerコマンドを実行するときcore.configがないと怒られます。どうせchaincodeのサンプルを動かすのでここでクローンしておきましょう。

```
cd $GOPATH/src/github.com/hyperledger
git clone http://github.com/hyperledger/fabric
```

3. first-networkディレクトリに移動して、"byfn.sh -m up"を実行し、テストスクリプトを実行します。

```
cd /tmp/fabric-samples/first-network/
./byfn.sh -m up  # yを押下して実行。ENDと出たら成功。Ctrl+Cで脱出。
```

4. "byfn.sh -m generate"を実行して、crypto-configフォルダ（証明書）とchannel-artifactsフォルダ下のファイル（genesis）を作成します。

```
./byfn.sh -m down
./byfn.sh -m generate
```

5. cliコンテナが起動しないようdocker-compose.yamlを編集します。

```
cp docker-compose-cli.yaml docker-compose.yaml
vi docker-compose.yaml
```

ここで、"cli:"以下の行を丸ごと削除します。
```
  cli:
    container_name: cli
    image: hyperledger/fabric-tools
    tty: true
    ...
```

5. docker-composeを実行します。

```
docker-compose -f docker-compose.yaml up -d
```

6. peerコマンド実行のための初期設定として、以下の環境変数を~/.bash_profileなどに設定します。

```
export GOPATH=/opt/gopath
export CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
export CORE_LOGGING_LEVEL=DEBUG
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_CERT_FILE=/tmp/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/tmp/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/tmp/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/tmp/fabric-samples/first-network/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
```

7. peerコマンド実行のための初期設定として、/etc/hostsを修正して、orderer.example.comとpeerN.orgN.example.comのIPアドレスを127.0.0.1に設定します。

```
127.0.0.1 orderer.example.com
127.0.0.1 peer0.org1.example.com
127.0.0.1 peer1.org1.example.com
127.0.0.1 peer0.org2.example.com
127.0.0.1 peer1.org2.example.com
```
 
8. channelを作成してjoinします。

```
peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls true --cafile /tmp/fabric-samples/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

peer channel join -b mychannel.block
```

9. $GOPATH下にあるサンプルchaincodeである "github.com/.../chaincode_example02" をインストールします。

```
peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02
```

10. chaincodeを初期化します。

```
peer chaincode instantiate -o orderer.example.com:7050 --tls true --cafile /tmp/fabric-samples/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "OR ('Org1MSP.member','Org2MSP.member')"
```

11. chaincodeをinvokeします。

```
peer chaincode invoke -o orderer.example.com:7050  --tls true --cafile /tmp/fabric-samples/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem  -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'
```

12. chaincodeをqueryします。

```
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```
 
13. 初期化してやり直したい時は3.からやり直ししましょう。
