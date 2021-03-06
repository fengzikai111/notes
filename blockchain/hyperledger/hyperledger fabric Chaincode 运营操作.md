# `Chaincode` 运营操作

`Chaincode`是一个用[`Go`](https://golang.org/)，[`node.js`](https://nodejs.org/)或[`Java`](https://java.com/en/)编写的程序，它**实现了一个指定的接口**。`Chaincode`运行在与支持对等节点的进程隔离的安全`Docker`容器中。`Chaincode`通过应用程序提交的**交易初始化和管理分类帐状态**。

链码通常处理**网络成员同意认可的业务逻辑**，因此可以将其视为“**智能合约**”。由**链码创建的状态仅限于该链码，并且不能由另一个链码直接访问**。但是，在同一网络中，给定适当的权限，链码可以**调用另一个链码**来访问其状态。

在以下部分中，将通过区块链网络运营商`Noah`的眼睛探索链码。为了诺亚的利益，将专注于**链码生命周期**运营；根据区块链网络中链码的**运营生命周期、打包、安装、实例化和升级**链码的过程。

# `Chaincode` 生命周期

`Hyperledger Fabric API`支持与区块链网络中的**各个节点（对等体，订购者和`MSP`）进行交互**，并且还允许人们**在支持对等节点上打包，安装，实例化和升级**链代码。`Hyperledger Fabric`语言特定的`SDK`抽象出`Hyperledger Fabric API`的细节以促进应用程序开发，尽管它可用于管理链代码的生命周期。此外，可以通过`CLI`直接访问`Hyperledger Fabric API`，将在本文档中使用它。

系统提供了四个命令来管理链码的生命周期：**包、安装、实例化和升级**。在将来的版本中，正在考虑添加停止和启动交易来禁用和重新启用链代码而无需实际卸载它。**成功安装和实例化链码后，链码处于活动状态（正在运行），并可以通过调用交易处理交易，链码可能在安装后随时升级**。

# `Chaincode` 打包

**链码包由3部分组成**：

+ 链码由`ChaincodeDeploymentSpec`或`CDS`定义。`CDS`根据**代码和其他属性（如名称和版本）定义链代码包**
+ **可选**的实例化策略，可以通过用于**认可策略**的相同在语法上描述，并在[认可策略](https://hyperledger-fabric.readthedocs.io/en/latest/endorsement-policies.html)中描述
+ 由**拥有**链码的实体签署的一组签名

**签名用于以下目的**：

+ 建立链码的**所有权**
+ 允许**验证**包的内容
+ 允许**检测**包篡改

根据链码的实例化策略**验证**通道上链码的**实例化交易的创建者**。

## 创建包

包装链码有两种方法：一个用于**何时**要拥有链码的**多个所有者**，因此需要使用**多个身份签名**的链代码包。此工作流程要求最初**创建一个签名**的链代码包（`SignedCDS`），随后将其**串行**传递给**其他所有者以进行签名**。

更简单的工作流程适用于部署仅具有发起`install`交易的节点标识的签名的`SignedCDS`。首先解决更复杂的案例。但是，如果还不需要担心多个所有者，可以跳到下面的安装链代码部分。

要**创建签名的链码包**，请使用以下命令：

```sh
$ peer chaincode package -n mycc -p github.com/hyperledger/fabric/examples/chaincode/go/example02/cmd -v 0 -s -S -i "AND('OrgA.admin')" ccpack.out
```

`-s`选项**创建一个可由多个所有者签名的包**，而不是简单地创建原始`CDS`。指定`-s`时，**如果其他所有者需要签名，则还必须指定`-S`选项**。否则，该过程将创建一个`SignedCDS`，除了`CDS`之外，该策略仅包括实例化策略。

`-S`选项指示进程使用由`core.yaml`中`localMspid`属性的值标识的`MSP`对包进行`signpackage` 。

可选的`-i`选项允许为链码**指定实例化策略**。**实例化策略具有与认可策略相同的格式**，并指定哪些身份可以实例化链代码。在上面的示例中，只允许`OrgA`的管理员实例化链代码。如果**未提供策略，则使用默认策略，该策略仅允许对等方的`MSP`的管理员标识实例化链代码**。

## 包签名

在创建时签署的链码包可以**移交给其他所有者进行检查和签名**。该工作流程支持链码包的带外签名。

[`ChaincodeDeploymentSpec`](https://github.com/hyperledger/fabric/blob/master/protos/peer/chaincode.proto#L78)可以选择由**集体所有者签名**，以创建[`SignedChaincodeDeploymentSpec`](https://github.com/hyperledger/fabric/blob/master/protos/peer/signed_cc_dep_spec.proto#L26)（或`SignedCDS`）。`SignedCDS`包含3个元素：

+ `CDS`包含**源代码**，链码的**名称和版本**。
+ 链码的**实例化策略**，表示为背书认可策略。
+ 通过[背书](https://github.com/hyperledger/fabric/blob/master/protos/peer/proposal_response.proto#L111)认可策略定义的链码**所有者列表**。

> **注意**：此绑定策略是在带外确定的，以便在某些通道上实例化链代码时提供适当的`MSP`主体。如果未指定实例化策略，则**默认策略是该通道的任何`MSP`管理员**。

每个所有者通过将`ChaincodeDeploymentSpec`与该所有者的身份（例如证书）相结合并**签署合并结果**来认可`ChaincodeDeploymentSpec`。链码**所有者**可以使用以下命令**对先前创建的已签名包进行签名**：

```sh
$ peer chaincode signpackage ccpack.out signedccpack.out
```

其中`ccpack.out`和`signedccpack.out`分别是**输入和输出包**。`signedccpack.out`包含使用本地`MSP`签名的程序包的**附加签名**。

## 安装链码

安装交易将链代码的源代码打包成称为`ChaincodeDeploymentSpec`（或`CDS`）的规定格式，并将其安装在将运行该链代码的**对等节点**上。

> **注意**：必须在将运行链码的通道的**每个支持对等节点**上安装链代码。

如果只为`ChaincodeDeploymentSpec`提供安装`API`，它将**默认实例化策略**并包含一个**空的所有者**列表。

> **注意**：`Chaincode`只应安装在**链码拥有成员的对等节点**上，以**保护链码逻辑**与网络上其他成员的**机密性**。那些**没有链码**的成员，**不能成为链码交易的参与者**。也就是说，他们**无法执行**链码。但是，他们仍然**可以验证交易并将其提交到分类帐**。

要安装链代码，请将[`SignedProposal`](https://github.com/hyperledger/fabric/blob/master/protos/peer/proposal.proto#L104)发送到[`System Chaincode`](https://hyperledger-fabric.readthedocs.io/en/latest/chaincode4noah.html#system-chaincode)部分中描述的**生命周期系统链代码（`LSCC`）**。例如，要使用`CLI`安装[`Simple Asset Chaincode`](https://hyperledger-fabric.readthedocs.io/en/latest/chaincode4ade.html#simple-asset-chaincode)部分中描述的`sacc`示例链代码，命令将如下所示：

```sh
$ peer chaincode install -n asset_mgmt -v 1.0 -p sacc
```

`CLI`在内部为`sacc`创建`SignedChaincodeDeploymentSpec`并将其发送到**本地对等方**，后者在`LSCC`上调用`Install`方法。`-p`选项的参数**指定了链代码的路径**，该链代码必须位于用户`GOPATH`的源树中，例如，`$GOPATH/src/sacc`。有关命令选项的完整说明，请参阅`CLI`部分。

请注意，为了在对等方上安装，`SignedProposal`的签名**必须来自对等方的本地`MSP`管理员**。

## 实例化

实例化交易**调用生命周期系统链接（`LSCC`）**来创建和初始化通道上的链代码。这是一个**链码通道绑定过程**：链码可以绑定到**任意数量**的通道，并且可以**单独和独立**地在每个通道上运行。换句话说，无论可以在其上**安装和实例化链代码的其他通道数量**，**状态都与提交交易的通道保持隔离**。

实例化交易的创建者必须满足`SignedCDS`中**包含的链代码的实例化策略**，并且还必须是通道上的写入器，其被配置为通道创建的一部分。这对于通道的安全性非常重要，可以**防止恶意实体部署链代码或欺骗成员在未绑定的通道上执行链代码**。

例如，回想一下默认实例化策略是任何通道`MSP`管理员，因此**链代码实例化的创建者必须是通道管理员的成员**。当**交易提议到达背书人**时，它会根据**实例化策略验证创建者的签名**。在将交易提交到分类帐之前，在**交易验证期间再次执行此操作**(二次验证)。

实例化交易还为通道上的该链代码**设置了认可策略**。认可政策描述了通道成员接受的交易结果的**证明要求**。

例如，使用`CLI`实例化`sacc`链代码并使用`john`和`0`初始化状态，该命令将如下所示：

```sh
$ peer chaincode instantiate -n sacc -v 1.0 -c '{"Args":["john","0"]}' -P "AND ('Org1.member','Org2.member')"
```

> **注意**：认可政策（`CLI`使用波兰表示法），这需要**得到`Org1`和`Org2`成员对所有`sacc`交易的认可**。也就是说，`Org1`和`Org2`都必须在`sacc`上执行`Invoke`的结果，以使交易有效。

成功**实例化后**，链代码在通道上进入**活动状态**，并**准备处理`ENDORSER_TRANSACTION`类型**的任何交易提议。交易在到达认可对等方时同时处理。

## 实例化

可以通过**更改其版本**来随时升级链码，该版本是`SignedCDS`的一部分。其他部分，例如所有者和实例化策略是可选的。但是，**链码名称必须相同**，否则它将被视为完全不同的链码。

在升级之前，**必须在所需的参与者上安装新版本的链代码**。升级是一种类似于实例化交易的交易，它**将新版本的链码绑定到通道**。绑定到**旧版链代码的其他通道仍然使用旧版本运行**。换句话说，**升级交易一次只影响一个通道，即提交交易的通道**。

> **注意**：由于链码的多个版本可能**同时处于活动状态**，因此升级过程**不会自动删除旧版本**，因此用户必须暂时对其进行管理。

实例化交易有一个细微差别：根据**当前的链码**实例化策略**检查升级**交易，而**不是新策略**（如果指定）。这是为了确保只有当前实例化策略中指定的**现有成员才能升级链代码**。

> **注意**：在升级期间，调用链码`Init`函数以执行任何与数据相关的更新或重新初始化它，因此必须注意**避免在升级链代码时重置状态**。

## 停止并开始

请注意，**尚未实现停止和启动生命周期交易**。但是，可以通过**从每个参与者中删除链代码容器**和`SignedCDS`包来**手动停止链码**。这是通过**删除运行支持对等节点的每个主机或虚拟机上的链代码容器**，然后从**每个支持对等节点删除`SignedCDS`来完成**的：

> **注意**：为了从对等节点删除`CDS`，首先需要**输入对等节点的容器**。我们真的需要提供一个可以执行此操作的实用程序脚本。

```sh
docker rm -f <container id>
rm /var/hyperledger/production/chaincodes/<ccname>:<ccversion>
```

停止在以控制方式进行升级的工作流程中非常有用，其中在**发出升级之前可以在所有对等体上的通道上停止链码**。

# `CLI`

> 正在评估为`Hyperledger Fabric`对等二进制文件分发特定于平台的二进制文件的需求。目前，只需从正在运行的`docker`容器中调用命令即可。

要查看当前可用的`CLI`命令，请在正在运行的`fabric-peer`的 `Docker` 容器中执行以下命令：

```sh
$ docker run -it hyperledger/fabric-peer bash
# peer chaincode --help
```

其中显示的输出类似于以下示例：

```sh
Usage:
  peer chaincode [command]

Available Commands:
  install     Package the specified chaincode into a deployment spec and save it on the peer's path.
  instantiate Deploy the specified chaincode to the network.
  invoke      Invoke the specified chaincode.
  list        Get the instantiated chaincodes on a channel or installed chaincodes on a peer.
  package     Package the specified chaincode into a deployment spec.
  query       Query using the specified chaincode.
  signpackage Sign the specified chaincode package
  upgrade     Upgrade chaincode.

Flags:
      --cafile string                       Path to file containing PEM-encoded trusted certificate(s) for the ordering endpoint
      --certfile string                     Path to file containing PEM-encoded X509 public key to use for mutual TLS communication with the orderer endpoint
      --clientauth                          Use mutual TLS when communicating with the orderer endpoint
      --connTimeout duration                Timeout for client to connect (default 3s)
  -h, --help                                help for chaincode
      --keyfile string                      Path to file containing PEM-encoded private key to use for mutual TLS communication with the orderer endpoint
  -o, --orderer string                      Ordering service endpoint
      --ordererTLSHostnameOverride string   The hostname override to use when validating the TLS connection to the orderer.
      --tls                                 Use TLS when communicating with the orderer endpoint
      --transient string                    Transient map of arguments in JSON encoding

Use "peer chaincode [command] --help" for more information about a command.
```

为了便于在脚本应用程序中使用，`peer`命令在命令**失败时始终生成非零返回码**。链码命令示例：

```sh
# 安装 和 实例化 链码
peer chaincode install -n mycc -v 0 -p path/to/my/chaincode/v0
peer chaincode instantiate -n mycc -v 0 -c '{"Args":["a", "b", "c"]}' -C mychannel

# 安装 和 升级 链码
peer chaincode install -n mycc -v 1 -p path/to/my/chaincode/v1
peer chaincode upgrade -n mycc -v 1 -c '{"Args":["d", "e", "f"]}' -C mychannel

# 查询 和 调用 链码交易
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","e"]}'
peer chaincode invoke -o orderer.example.com:7050  --tls --cafile $ORDERER_CA -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'
```

# 系统链码

系统链代码具有相同的编程模型，除了**它在对等进程内运行**而**不是**像普通链代码那样**在孤立的容器中运行**。因此，系统链代码**内置于对等可执行文件中，并且不遵循上述相同的生命周期**。特别是**安装、实例化和升级不适用于系统链代码**。

系统链代码的目的是在**同行和链码之间快速`gRPC`通信成本，并权衡管理的灵活性**。例如，系统链代码**只能使用对等二进制文件**进行升级。它还**必须注册一组[固定的参数](https://github.com/hyperledger/fabric/blob/master/core/scc/importsysccs.go)**，并且**没有认可政策或认可政策功能**。

系统链代码在`Hyperledger Fabric`中用于实现许多系统行为，以便系统集成商可以适当地**替换或修改**它们。

系统链代码的当前列表：

1. `LSCC` **生命周期系统链码**处理上述生命周期请求。

2. `CSCC` **配置系统链码**处理对等端的通道配置。

3. `QSCC` **查询系统链码**提供分类帐查询`API`，例如获取块和交易。

用于**认可和验证**的前系统链代码已被[可插拔交易认可和验证](https://hyperledger-fabric.readthedocs.io/en/latest/pluggable_endorsement_and_validation.html)文档所描述的可插入认可和验证功能所取代。在修改或更换这些系统链码时，尤其是`LSCC`，必须格外小心。