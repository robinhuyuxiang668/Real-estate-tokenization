Fuji 部署地址：

RealEstateToken 0x0f67ED63b0A5b7D2d407Fbb2bD38c7F7f3f38B4D https://testnet.snowtrace.io/address/0x0f67ED63b0A5b7D2d407Fbb2bD38c7F7f3f38B4D/contract/43113/code

Issuer 0xc97977E483A7D8a6e0b7EA564CE74c52eD114a77 https://testnet.snowtrace.io/address/0xc97977E483A7D8a6e0b7EA564CE74c52eD114a77/contract/43113/code

RwaLending 0x945e3C2B0Eb65306AA8D1Dc0afF32D89a538909A https://testnet.snowtrace.io/address/0x945e3C2B0Eb65306AA8D1Dc0afF32D89a538909A/contract/43113/code

EnglishAuction 0xd840f7b09D76DE59DCc31A7CbF61cFF673227Ed2 https://testnet.snowtrace.io/address/0xd840f7b09D76DE59DCc31A7CbF61cFF673227Ed2/contract/43113/code

1.编写 JavaScript 函数开始，这个函数能帮助我们获取创建 NFT 所需的信息，构建 NFT 的元数据，将代码存储于链上 ，见 FunctionsSource.sol：

房产地址

建造年份

土地面积

居住面积

卧室数量

2.运用工厂模式发行 ERC-1155 通证 ERC1155Core.sol

3.CrossChainBurnAndMintERC1155.sol
继承 ERC1155Core.sol，扩展支持通过 CCIP 实现的跨链转账。

4.RealEstatePriceDetails.sol
辅助合约将用于周期性地使用 Automation 和 Functions 服务来获取实世界资产的价格详情。为了使其正常运作，已经将创建的 JavaScript 脚本并将其内容放入到 1 中的 FunctionsSource.sol

5. RealEstateToken.sol
   编译器优化器设置为 200 次运行，并将 EVM 版本设置为 "Paris"

6.Issuer.sol

7.应用：
1）RwaLending.sol 自动化房地产借贷

ERC1155Receiver
为了让该合约能够接收 ERC-1155 通证，它必须实现 IERC1155Receiver 接口。最重要的检查发生在该接口中的 onERC1155Received 和 onERC1155ReceivedBatch 函数里。具体来说，我们在这里确保 msg.sender 是我们自己的 RealEstateToken.sol。

初始和清算阈值
为了简化代码，在这个例子中的初始和清算阈值被硬编码为 60%和 75%。这意味着如果 Alice 的房产价值 100 美元，她将获得 60 USDC 作为贷款。如果在某个时刻，她的房产价值下降到低于 75 美元，任何人都可以清算其头寸，而她将失去所有的 ERC-1155 通证。但她会保留 USDC。

滑点保护
调用 borrow 函数时，Alice 还需要提供 minLoanAmount 和 maxLiquidationThreshold 的值，以保护自己免受滑点的影响。由于 loanAmount 和 liquidationThreshold 是在链上动态计算的，Alice 需要设定自己可接受的边界值，否则交易将被回滚。

1 美元不等于 1 USDC
通常情况下，1 美元并不等同于 1 USDC。其价值可能在 0.99 美元到 1.01 美元之间波动，因此将美元值硬编码为 1 USDC 是极其危险的。我们必须使用 USDC/USD 的 Chainlink Price Feed。

使用 Chainlink Data Feeds - 精度位数
USDC/USD 的喂价有 8 位小数，而 USDC ERC-20 通证本身有 6 位小数。例如，喂价值可能返回 99993242，这意味着 1 USDC 等于 0.99993242 美元。现在，我们需要将这 100 美元转换为我们实际要发送或接收的 USDC 通证的确切数量。为此，我们使用 getValuationInUsdc 函数。

使用 Chainlink Data Feeds - 价格更新频率
每产生一个新区块就推送价格更新到区块链上是不切实际的。因此，有两种触发条件（可以在 data.chain.link 的"Trigger parameters" 处看到）：价格偏差阈值和心跳周期。这些参数因不同的数据源而异，开发者们需要对此保持关注。基本上，DONs 持续地计算新的价格并将其与最新价格进行比较。如果新价格低于或高于偏差阈值，一个新的报告会被推送到链上。此外，假设心跳周期设为 1 小时，如果前一小时内没有新的报告生成，不管怎样都会推动一个新报告到链上。

这对开发者而言为什么很重要？使用过期价格来进行链上行为是有风险的。因此，开发者必须：

应该使用 latestRoundData 函数而不是 latestAnswer 函数。

将 updatedAt 变量与他们所选的 usdcUsdFeedHeartbeat 进行对比。

作为一种合理性检查，我们同样希望验证返回的 roundId 大于零

2） EnglishAuction.sol 在英式拍卖会上出售细分权益化的 RWA
竞标仅使用原生通证（,任何用户都可以通过存入超过当前最高出价者所存的原生通证数量的方式竞标。除了当前的最高出价者，所有竞标者都可以撤回他们的竞标。 7 天后，任何人都可以调用 endAuction 函数，这将把 ERC-1155 房地产通证转移给最高出价者
