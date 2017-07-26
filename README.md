## Dockerで、orderer, peer, caを動かしてchaincodeを動かすテストを実施します。

#### 前提条件
- Mac OS X

#### 事前準備
- gitコマンドのインストール
- GO 1.8以上のインストール
- curlコマンドのインストール(brew install curl)
- Dockerのインストール

#### 1. Dockerで、orderer, peerを動かして、コマンドベースでchaincodeをインストールして、初期化、invoke,queryします。

byfn.sh -m upとコマンドを実行し、証明書とチェーン用genesisファイルの生成から、docker-compose-cli.yamlを読み込みorderer,peerのdockerコンテナを実行し、cliコンテナ上でテストコマンドを実行してchaincodeが一通り動くかのテストの実行まで、全自動でテストを行います。

1. /tmpディレクトリにfabric-testをクローンして、バイナリファイルやdockerイメージを導入します。

```
cd /tmp
git clone https://github.com/david3080/fabric-test.git
cd fabric-test/
curl -sSL https://goo.gl/iX9dek | bash
export PATH=$PWD/bin:$PATH # ダウンロードしたbinフォルダ下のpeerコマンドなどにパスを通します。
```

2. /opt/gopathにfabricをcloneします。fabric1.0.0ではこれをしないと1でダウンロードしたpeerコマンドを実行するときcore.configがないと怒られます。どうせchaincodeのサンプルを動かすのでここでクローンしておきましょう。

```
cd /opt/gopath/src/github.com/hyperledger
git clone http://github.com/hyperledger/fabric
```

3. first-networkディレクトリに移動して、"byfn.sh -m up"を実行し、テストスクリプトを実行します。

```
cd /tmp/fabric-test/first-network/
./byfn.sh -m up  # yを押下して実行。ENDと出たら成功。Ctrl+Cで脱出。
```

4. 証明書とチェーンのgenesisの初期化をします。

```
./byfn.sh -m down
```

#### 2. Dockerで、orderer, peer, caを動かして、コマンドベースでchaincodeをインストールして、初期化、invoke,queryします。

byfn.shコマンドを使って証明書とチェーン用genesisファイルを作成し、docker-compose-ca.yamlを読み込みorderer,peer,caのdockerコンテナを実行します。そのあと、Mac上のコマンドラインからコマンドベースでchaincodeが一通り動くかテストします。

1. "byfn.sh -m generate"を実行して、crypto-configフォルダ（証明書）とchannel-artifactsフォルダ下のファイル（genesis）を作成します。

```
./byfn.sh -m generate
```

2. docker-composeを実行します。

```
docker-compose -f docker-compose-ca.yaml up -d
```

3. peerコマンド実行のための初期設定として、以下の環境変数を~/.bash_profileなどに設定します。source ~/.bash_profileを実行し、設定した環境変数を反映させることも忘れないでください。

```
export GOPATH=/opt/gopath
export CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
export CORE_LOGGING_LEVEL=DEBUG
export CORE_PEER_ADDRESS=peer0.org1.example.com:7051
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_TLS_CERT_FILE=/tmp/fabric-test/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt
export CORE_PEER_TLS_KEY_FILE=/tmp/fabric-test/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key
export CORE_PEER_TLS_ROOTCERT_FILE=/tmp/fabric-test/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/tmp/fabric-test/first-network/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
```

4. peerコマンド実行のための初期設定として、/etc/hostsを修正して、orderer.example.comとpeerN.orgN.example.comのIPアドレスを127.0.0.1に設定します。

```
127.0.0.1 orderer.example.com
127.0.0.1 peer0.org1.example.com
127.0.0.1 peer1.org1.example.com
127.0.0.1 peer0.org2.example.com
127.0.0.1 peer1.org2.example.com
127.0.0.1 ca.org1.example.com
127.0.0.1 ca.org2.example.com
```
 
5. channelを作成してjoinします。

```
peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls true --cafile /tmp/fabric-test/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

peer channel join -b mychannel.block
```

6. $GOPATH下にあるサンプルchaincodeである "github.com/.../chaincode_example02" をインストールします。

```
peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02
```

7. chaincodeを初期化します。

```
peer chaincode instantiate -o orderer.example.com:7050 --tls true --cafile /tmp/fabric-test/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a", "100", "b","200"]}' -P "OR ('Org1MSP.member','Org2MSP.member')"
```

8. chaincodeをinvokeします。

```
peer chaincode invoke -o orderer.example.com:7050 --tls true --cafile /tmp/fabric-test/first-network/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'
```

9. chaincodeをqueryします。

```
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```
 
10. 初期化してやり直したい時は"./byfn.sh -m down"を実行して、1からやり直ししましょう。
