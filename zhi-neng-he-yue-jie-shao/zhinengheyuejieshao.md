# 智能合约介绍

##一个简单的智能合约
让我们从最简单的示例开始吧。你现在还不明白，没关系，当你继续往后深入的时候自然就会明白的。

###存储

```solidity
pragma solidity ^0.4.0;

contract SimpleStorage {
    uint storedData;

    function set(uint x) public {
        storedData = x;
    }

    function get() public constant returns (uint) {
        return storedData;
    }
}
```

第一行告诉我们代码是基于Solidity0.4.0版本编写的，你也可以用0.4.x的任意版本号(译者注：版本号用于表示功能迭代或Bug修复)，但是不能高于0.5.0。这用于保证对新版本编辑器的兼容。用关键字`pragma`是因为通常编译指令告诉编译编译器如何对待代码(eg. [pragma one](https://en.wikipedia.org/wiki/Pragma_once))。

一个智能合约是用Solidity语言编写的，是一个在以太坊网络上的一个特殊地质，它包含了一系列的代码（包含若干函数）以及数据（状态数据）。这一行`uint storedData;`生命了一个类型是`uint`(无符号256位整形)名字叫做`storedData`的状态变量。你可以把这个想成一个保存在数据库中的槽，它可以被查询或者通过函数去管理。在以太坊环境中，这是自有的智能合约。在这个例子中，`set`和`get`方法用于修改和获取变量的值。

为了访问这个状态变量，通常在其他语言中需要加上`this.`，在这里不需要加上前缀`this.`。

在这个智能合约中，目前还没有什么功能，除了存储了一个任何人可以访问而且没有一个可行的方法去阻止你去改变的数字。当然，每个人都可以调用`set`方法去设置一个不同的值来覆盖掉之前的值，但是这个数字的历史记录还是保存在区块中。稍后，我们将会严格限制数据的访问来确保只有你能修改。

###提示
所有的识别符（合约名字，函数名字以及变量名字）都需要是严格符合ASCII编码的。string类型的变量可以保存UTF-8编码的字符串。

###警告
一定要小心Unicode编码的文本，看起来一样的文本可能有不一样的code points或者被编码成不同的byte数组。

##货币示例

下面的智能合约是一个最简单的加密数字货币的实现。这可以凭空造出货币，但是只有合约的所有者才能这么做（这里就没有必要去实现不同的发行计划。）。在将来人们不需要通过用户名和密码注册一个用户来转账，只需要一个以太坊的秘钥对。

```solidity
pragma solidity ^0.4.21;

contract Coin {
    // The keyword "public" makes those variables
    // readable from outside.
    address public minter;
    mapping (address => uint) public balances;

    // Events allow light clients to react on
    // changes efficiently.
    event Sent(address from, address to, uint amount);

    // This is the constructor whose code is
    // run only when the contract is created.
    function Coin() public {
        minter = msg.sender;
    }

    function mint(address receiver, uint amount) public {
        if (msg.sender != minter) return;
        balances[receiver] += amount;
    }

    function send(address receiver, uint amount) public {
        if (balances[msg.sender] < amount) return;
        balances[msg.sender] -= amount;
        balances[receiver] += amount;
        emit Sent(msg.sender, receiver, amount);
    }
}
```

这个合约向我们介绍了一些新的概念，让我们一个个过一下。

这行代码`address public minter;`声明了一个可以公开访问的address类型的状态变量。`address`是一个长度为160bit无法进行算数运算的值。它用于存储一个合约的地址或者属于外部用户的秘钥串。关键字`public`会自动生成一个函数，你可以在外部的合约中通过这个函数访问这个变量的当前值。编译器生成的代码跟一下的代码是等价的：

```solidity
function minter() returns (address) { return minter; }
```

当然如果你像上面这样增加一个一样的函数，上面的函数是不会再工作了，因为你有一个名字一样的函数和变量，你很可能已经想明白了，编译器替你做了这些工作。

下一行代码`mapping (address => uint) public balances;`声明了一个公开的状态变量，它是一个更加复杂的数据类型。这个类型将address映射到无符号证书。mapping类型可以被看成是一个[哈希表](https://en.wikipedia.org/wiki/Hash_table)，你可以认为它是被虚拟初始化的，这个上面的任何key的值全部被初始化0。这仅仅只是一个类比，因为不可能获取一个包含所有key的列表，也不可能列出所有值。所以在你往maping添加值或者在你使用的上下文中，也和上面说的是一样的。在这个例子中自动生成的`get`方法将会稍微负责一点。它将会跟下面的代码比较相似：

```solidity
function balances(address _account) public view returns (uint) {
    return balances[_account];
}
```

正如你看到的，你将会很容易通过这个函数查到单个用户的余额。

这一行代码`event Sent(address from, address to, uint amount);`声明了一个在函数`send`最后一行调用的"event"。用户接口(比如应用服务器)可以没有额外消耗的监听在区块上发出的事件。事件一旦提交，监听方会马上收到`from`,`to`和`amount`，这样跟踪交易就很容易了。为了监听这个事件，你需要这样做：

```js
Coin.Sent().watch({}, '', function(error, result) {
    if (!error) {
        console.log("Coin transfer: " + result.args.amount +
            " coins were sent from " + result.args.from +
            " to " + result.args.to + ".");
        console.log("Balances now:\n" +
            "Sender: " + Coin.balances.call(result.args.from) +
            "Receiver: " + Coin.balances.call(result.args.to));
    }
})
```

观察自动生成的函数`balances`在用户接口中被调用了。

函数`Coin`是构造函数，只会在合约创建的时候调用，之后不会被调用。这将会永远保存了合约的创建者：`msg`（`tx`和`block`）是可以访问到区块的一些属性的全局魔法变量。`msg.sender`是函数调用的(外部)地址。

最终，函数`mint`和`send`产生了合约的结果而且能被用户和合约调用。`mint`只能被合约创建者调用，其他人调用将什么都不做。另一方面，`send`可以被任何人(拥有一些钱)用于将一些钱转给其他人。注意如果你用这个合约向一个地址转账，你无法通过区块链浏览器看到任何信息，因为你发送钱以及余额变动被保存在这个特殊的货币合约中。利用事件来创建一个"区块链浏览器"会相对容易。

##区块链基础

对于程序员来说，区块链作为一个概念不是特别难理解。这是因为其中最复杂的（挖矿，哈希，椭圆曲线密码学，P2P网络，.etc）提供了一些特性。一旦你理解了上面这些概念，你不必担心如何底层的技术，就像你在使用亚马逊的AWS时候，你不必去理解它内部是如何工作的。

###交易

一个区块是一个公开的，基于交易的数据库。这意味着任何人加入网络之后，都可以读取这些数据库中的数据。如果你想改变数据库中的一些东西，你必须要创建一个被所有其他人所接受的所谓的交易。交易这个词暗示着你想要去做的改变（假设你想同时改变两个值），要么完全不生效，那么完全生效。此外，当交易被记录在数据库中，没有其他交易可以去改变它（译者注：交易）。

作为一个例子，想象一张电子货币中的表格，这里面罗列了所有的账号的余额。如果要从一个账号转账到另一个账号，事务性的数据库保障了如果一个账号中被减掉的金额，会被加到另一个账号。如果由于某些原因，无法向目标账户增加金额，那么源账号的金额也不会被修改。

此外，一笔交易始终被发送方加密签名。这直接保护了某些在数据库中的改动。在电子货币中，一个简单的检查将会保证只有持有了秘钥的账号才可以从这里面转账。

###区块

在比特币小组中，一个主要的实现障碍是“双花攻击”：当两个在网络中的交易都去置空一个账号，这是否会产生一个冲突？

一个抽象的回答是你无须关心这个。交易的顺序会替你做出选择，交易将被打包进“区块”中，然后将会在被执行，而且分布式的保存在加入的节点中。如果有两笔冲突的交易，最终是第二名的交易会被拒绝，也不会被打包到区块中。

这些区块按照时间形成了一个线性序列，这就是“区块链”这个词产生的原因。区块每隔一段固定的就会被加入（译者注：区块链中）,对于以太坊来说，这个时间间隔大概是17秒。

作为“订单选择机制”（“挖矿”）的一部分，随着时间迁移，区块会被回退，除了链的“顶端”.随着越来越多的区块被添加进来，区块被回退的可能就越小。所以随着你等待的时间越长，回退你的交易或者从区块链中移除你的交易的可能性就越小。

##以太坊虚拟机

###概要

以太坊虚拟机或者EVM是以太坊中的智能合约的运行时环境。这不仅仅只是一个沙盒而且是完全孤立的，这意味着在EVM中运行的代码无法访问网络，文件系统以及其他进程。甚至智能合约访问其他的智能合约也是有限制的。

###账号

在以太坊中有两类有同样地址空间的账号：`外部账号`是被一对公私钥控制的（i.e. 人），`合约账号`是受保存在这个账号中的控制的。

外部账号的地址是决定于公钥的，同时合约账号的地址（这是由创建者的地址以及这个地址上的交易数量，所谓的“随机数”）是在合约生成的时候生成的。

不管账号中是否保存了代码，对于EVM来说，这两类账号是一样的。

每个账号都有一个持久化的键值存储，一个256bit映射到256bit的存储。

此外，每个账号都有一个以太币余额，发起带有以特币的交易将会改变账户余额。

###交易

交易是一个从一个账号发送到另一个账号（可以是相同的地址也可以是特殊的0地址）的消息。它可以包含二进制数据以及以太币。

如果目标地址含有代码，代码将会被执行，然后payload将会被作为输入数据。

如果交易的目标地址是零账号（地址是`0`），这个交易将会创建一个新合约。就像上面已经提到过的，合约的地址不是零地址，这决定于创建者的地址以及这个地址上发送的交易数（随机数）。创建合约的payload被编译成EVM的字节码然后被执行。执行的输出将永远被保存在合约的代码中。这意味着为了创建一个合约，你没必要发送真正的代码，而是返回了那个代码。

