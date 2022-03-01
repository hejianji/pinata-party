# 合约
打开PinataPartyContract.cdc，编写一下代码：
```
pub contract PinataPartyContract {
  pub resource NFT {
    pub let id: UInt64
    init(initID: UInt64) {
      self.id = initID
    }
  }
}
```
接下来，我们需要创建一个资源接口，我们将用它来定义哪些能力可以提供给其他人
```
pub resource interface NFTReceiver {
  pub fun deposit(token: @NFT, metadata: {String : String})
  pub fun getIDs(): [UInt64]
  pub fun idExists(id: UInt64): Bool
  pub fun getMetadata(id: UInt64) : {String : String}
}
```
接下来，我们需要定义代币收藏品（ Colletion ）接口。把它看成是存放用户所有 NFT 的钱包。
```
pub resource Collection: NFTReceiver {
    pub var ownedNFT: @{UInt64: NFT}
    pub var metadataObjs: {UInt64: { String : String }}

    init () {
        self.ownedNFT <- {}
        self.metadataObjs = {}
    }

    pub fun withdraw(withdrawID: UInt64): @NFT {
        let token <- self.ownedNFT.remove(key: withdrawID)!

        return <-token
    }

    pub fun deposit(token: @NFT, metadata: {String : String}) {
        self.ownedNFT[token.id] <-! token
    }

    pub fun idExists(id: UInt64): Bool {
        return self.ownedNFT[id] != nil
    }

    pub fun getIDs(): [UInt64] {
        return self.ownedNFT.keys
    }

    pub fun updateMetadata(id: UInt64, metadata: {String: String}) {
        self.metadataObjs[id] = metadata
    }

    pub fun getMetadata(id: UInt64): {String : String} {
        return self.metadataObjs[id]!
    }

    destroy() {
        destroy self.ownedNFT
    }
}
```
这个资源里有很多东西，说明一下。首先，有一个变量叫ownedNFT。这个是很直接的，它可以跟踪用户在这个合约中所有拥有的 NFT。

接下来，有一个变量叫metadataObjs。这个有点特殊，因为我们扩展了 Flow NFT 合约功能，为每个 NFT 存储元数据的映射。这个变量将代币 id 映射到其相关的元数据上，这意味着我们需要在设置代币 id 之前，将其设置为元数据。

然后我们初始化变量。定义在 Flow 中的资源中的变量必需初始化。

最后，我们拥有了 NFT Collection 资源的所有可用函数。需要注意的是，并不是所有这些函数大家都可以调用。你还记得在前面，NFTReceiver资源接口中定义了任何人都可以访问的函数。

合约代码就快完成了。因此，在 Collection资源的下面，添加以下内容：
```
pub fun createEmptyCollection(): @Collection {
    return <- create Collection()
}

pub resource NFTMinter {
    pub var idCount: UInt64

    init() {
        self.idCount = 1
    }

    pub fun mintNFT(): @NFT {
        var newNFT <- create NFT(initID: self.idCount)

        self.idCount = self.idCount + 1 as UInt64

        return <-newNFT
    }
}
```
首先，我们有一个函数，在调用时创建一个空的 NFT Collection。这就是第一次与合约进行交互的用户如何创建一个存储位置，该位置映射到定义好的 Collection资源。

之后，我们再创建一个资源（resource）。它很重要的，因为没有它，我们就无法铸造代币。NFTMinter资源包括一个idCount，它是递增的，以确保我们的 NFT 不会有重复的 id。它还有一个功能，用来创造 NFT。

在NFTMinter资源的下方，添加主合约初始化函数
```
init() {
      self.account.save(<-self.createEmptyCollection(), to: /storage/NFTCollection)
      self.account.link<&{NFTReceiver}>(/public/NFTReceiver, target: /storage/NFTCollection)
      self.account.save(<-create NFTMinter(), to: /storage/NFTMinter)
}
```
这个初始化函数只有在合约部署时才会被调用。它有三个作用。

1. 为收藏品（Collection）的部署者创建一个空的收藏品，这样合约的所有者就可以从该合约中铸造和拥有 NFT。
2. Collection资源发布在一个公共位置，并引用在一开始创建的NFTReceiver接口。通过这个方式告诉合约，在NFTReceiver上定义的函数可以被任何人调用。
3. NFTMinter资源被保存在账户存储中，供合约的创建者使用。这意味着只有合约的创造者才能铸造代币。

# 铸造 NFT
在 pinata-party项目的根目录下创建一个新的目录，我们把它叫做 transactions。创建好文件夹，在里面创建一个名为MintPinataParty.cdc 的新文件。现在，在你的MintPinataParty.cdc文件中，添加以下内容：
```
import PinataPartyContract from 0xf8d6e0586b0a20c7

transaction {
  let receiverRef: &{PinataPartyContract.NFTReceiver}
  let minterRef: &PinataPartyContract.NFTMinter

  prepare(acct: AuthAccount) {
      self.receiverRef = acct.getCapability<&{PinataPartyContract.NFTReceiver}>(/public/NFTReceiver)
          .borrow()
          ?? panic("Could not borrow receiver reference")

      self.minterRef = acct.borrow<&PinataPartyContract.NFTMinter>(from: /storage/NFTMinter)
          ?? panic("could not borrow minter reference")
  }

  execute {
      let metadata : {String : String} = {
          "name": "The Big Swing",
          "swing_velocity": "29",
          "swing_angle": "45",
          "rating": "5",
          "uri": "ipfs://QmRZdc3mAMXpv6Akz9Ekp1y4vDSjazTx2dCQRkxVy1yUj6"
      }
      let newNFT <- self.minterRef.mintNFT()

      self.receiverRef.deposit(token: <-newNFT, metadata: metadata)

      log("NFT Minted and deposited to Account 2's Collection")
  }
}
```
这是一个非常简单的交易代码，这在很大程度上要归功于 Flow 所做的工作，但让我们来看看它。首先，你会注意到顶部的导入语句。如果你还记得，在部署合约时，我们收到了一个账户地址。它就是这里引用的内容。因此，将0xf8d6e0586b0a20c7替换为你部署的账户地址。

接下来我们对交易进行定义。在我们的交易中，我们首先要做的是定义两个参考变量，receiverRef和minterRef。在这种情况下，我们既是 NFT 的接收者，又是 NFT 的挖掘者。这两个变量是引用我们在合约中创建的资源。如果执行交易的人对资源没有访问权，交易将失败。

接下来，我们有一个prepare函数。该函数获取试图执行交易的人的账户信息并进行一些验证。它会尝试 借用两个资源 NFTMinter和 NFTReceiver上的可用能力。如果执行交易的人没有访问这些资源的权限，验证无法通过，这就是交易会失败的原因。

最后是execute函数。这个函数是为我们的 NFT 建立元数据，铸造 NFT，然后在将 NFT 存入账户之前关联元数据。

现在我们差不多可以执行代码发送交易铸造 NFT 了。但首先，我们需要准备好我们的账户。在项目根目录下的命令行中，创建一个新的签名私钥。
```
flow keys generate
```
我们将需要私钥来签署交易，所以我们可以把它粘贴到flow.json文件中。

现在可以发送交易，简单的运行这个命令：
```
flow transactions send ./transactions/MintPinataParty.cdc --signer emulator-account
```

# 验证 token
最后，验证 token 是否在我们的账户中，并获取元数据。做到这一点，我们要写一个非常简单的脚本，并从命令行调用它。

在项目根目录，创建一个名为 scripts的新文件夹。在里面，创建一个名为CheckTokenMetadata.cdc的文件。在该文件中，添加以下内容：
```
import PinataPartyContract from 0xf8d6e0586b0a20c7

pub fun main() : {String : String} {
    let nftOwner = getAccount(0xf8d6e0586b0a20c7)
    // log("NFT Owner")
    let capability = nftOwner.getCapability<&{PinataPartyContract.NFTReceiver}>(/public/NFTReceiver)

    let receiverRef = capability.borrow()
        ?? panic("Could not borrow the receiver reference")

    return receiverRef.getMetadata(id: 1)
}
```
在脚本中，导入部署的合约地址。然后定义一个 main函数（这是脚本运行所需的函数名）。在这个函数里面，我们定义了三个变量：
1. nftOwner：拥有 NFT 的账户。由于使用部署了合约的账户中铸造了 NFT，所以在我们的例子中，这两个地址是一样的。
2. capability：需要从部署的合约中 借用的能力（或功能）。请记住，这些能力是受访问控制的，所以如果一个能力对试图借用它的地址不可用，脚本就会失败。我们正在从NFTReceiver资源中借用能力。
3. receiverRef：这个变量只是简单地记录我们的能力。

我们可以调用（可用的）函数。在这种情况下，我们要确保相关地址确实已经收到了我们铸造的 NFT，然后我们要查看与代币相关的元数据。让我们运行的脚本，看看得到了什么。在命令行中运行以下内容：
```
flow scripts execute ./scripts/CheckTokenMetadata.cdc
```
