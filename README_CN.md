3. [![Logo](https://img.shields.io/badge/KuCoin-KuMex-yellowgreen?style=flat-square)](https://github.com/Kucoin-academy/Guide)
[![GitHub stars](https://img.shields.io/github/stars/Kucoin-academy/best-practice.svg?label=Stars&style=flat-square)](https://github.com/Kucoin-academy/best-practice)
   [![GitHub forks](https://img.shields.io/github/forks/Kucoin-academy/best-practice.svg?label=Fork&style=flat-square)](https://github.com/Kucoin-academy/best-practice)
   [![GitHub issues](https://img.shields.io/github/issues/Kucoin-academy/best-practice.svg?label=Issue&style=flat-square)](https://github.com/Kucoin-academy/best-practice/issues)
   
   [![](https://img.shields.io/badge/lang-English-informational.svg?longCache=true&style=flat-square)](README_EN.md)
   [![](https://img.shields.io/badge/lang-Chinese-red.svg?longCache=true&style=flat-square)](README_CN.md)
   
   # KuCoin API最佳实践
   
   ## 前言
   
   在量化交易中，如何科学高效的接入交易所的API并且优雅的处理异常情况是非常重要的。
   
   此文将会提供接入KuCoin的最佳实践，你可以在本文的基础上加入你自己的用法，如果你对此文有任何的疑问或问题，非常欢迎你提出ISSUE。
   
   ## 正文
   
   对于高频交易者（每秒下单超过10个或者对行情和订单的维护时间延迟要求在毫秒级别）有两个非常重要的问题。
   
   1. 如何最快最准确的获取买卖盘
   2. 如何最快最准确的获取自己的订单信息
   
   KuCoin的服务器设置在**AWS 日本东京**，我们同时也提供**privatelink**的方式让你能够更快更稳定的连接KuCoin，如有需求请发送邮件至**newapi@kucoin.plus**或添加KuCoin官方API微信：**API_KuCoin**
   
   **注意：KuCoin官方API微信仅提供技术支持，不会向用户索要任何个人或账户信息，不会要求用户支付任何费用**。
   
   ### 如何最快最准确的获取买卖盘
   
   1. REST 请求方式（不推荐）
   
      直接调用REST接口即可获取，使用方法：以KuCoin为例 https://docs.kucoin.com/#get-ticker
   
      **优点**：
   
      * 调用简单；
      * 适合一些低频或对行情及时性不敏感的策略。
   
      **缺点**：
   
      * 延时高；
      * 有接口限频，无法频繁调用。
   
      **最佳使用场景**：
   
      * 非高频策略，如一些定投策略和主动性策略；
      * 在Websocket出现问题时，风险控制系统可以使用此方式来获取行情进行进一步的操作。
   
   2. Websockt Level-2推送（推荐）
   
      通过KuCoin提供Level-2 Websocket 5档和50档的全Level-2买卖盘推送，频率大约是100ms推送一次。
   
   3. Websockt Level-3推送（推荐）
   
      通过收取全市场的成交数据，在本地内存构建买卖盘，直接读取本地内存的数据。
   
   ### 如何最快最准确的维护订单
   
   1. REST 请求方式（不推荐）
   
      * 直接调用REST接口即可获取，使用方法: 以KuCoin为例 https://docs.kucoin.com/api/v1/orders/{order-id} 
   
      * REST请求是滞后的，在你查询到订单信息之后，订单还有可能继续成交，所以如果你是高频交易者只推荐你在一些补救逻辑和风险控制中使用REST请求。
   
   2. Websockt 私有平台推送（推荐）
   
      * 通过订阅私有频道来获取自己的订单信息，私有频道会实时的推送你自己的所有订单的全量信息，你无需频繁的通过REST请求来查询你自己的订单，节省限频的同时，实时性也更好。
   
   3. Websockt level-3推送（推荐）：
   
      * 如果你是高频交易者，我们强烈推荐你使用公共的Level-3来维护你自己的订单。Level-3的推送直接来源于撮合引擎，是所有消息中速度最快的，而且保证了完全的准确。
      * Level-3是流式线性的消息推送，每一条消息都有sequence，并且sequence一定是递增的，所以通过sequence你就能够确定你是否准确的收到了每一条消息。
   
   ### 我们先举一个简单的RECEIVE消息作为例子
   
   ```
    {
      "topic": "/spotMarket/level3v2:BTC-USDT",
      "subject": "received",
      "data": {
          "symbol": "BTC-USDT",                        // symbol
          "sequence": 3262786900,                 // sequence
          "orderId": "5f0186a2d765040006f0b1fa",  // order ID
          "clientOid": "a4ba662f77180b69f186aa578aa52cb6", // optional, for you to identify your order
          "ts": 1545914149935808589
       }
    }
   ```
   
   通常来讲Level-3一定会有topic、subject、data三个字段。**其中topic代表频道，subject代表消息类型，data里面是具体的消息内容**。
   
   我们先看一个流程图，  
   
   <img src="/Users/arthur/www/academy/best-practice/img/flow.png" alt="image-20200713141210773" style="zoom:50%;" />
   
   这个流程图很清晰的讲述了一个订单下单之后可能收到的Level-3消息以及各种情况。
   
   当你下单之后，**通常你一定会收到一个RECEIVE消息**，KuCoin系统内部从收到下单到发出消息的延迟**不会大于10ms**。
   
   **假设你到KuCoin的网络延迟为20ms，理论上你从下单到收到RECEIVE的间隔时间应该在30ms左右，由于RECEIVE消息的返回通常会快于下单请求本身的返回，所以你需要下单的时候附带clientOid，并且在本地维护两个key-value数据结构，第一个是clientOid作为key，订单对象作为value，用于你维护本地订单，各种语言都提供这样的数据结构，例如redis，memcache这样缓存系统也提供key-value结构**。
   
   你在BTC-USDT交易对下了一个clientOid为a4ba662f77180b69f186aa578aa52cb6的订单，数量为1，价格为9000的买单，在下单之前，你本地就应该有这样一个数据结构，
   
   ``` 
   {
   "a4ba662f77180b69f186aa578aa52cb6": {
       "orderId": "",
       "size": "1",
       "status": "",
       "price": "9000",
       "side": "buy",
       "symbol": "BTC-USDT",
       "dealSize": "0"
       }
   }
   ```
   
   当你收到RECEIVE消息之后
   
   ```
    {
      "topic": "/spottMarket/level3v2:BTC-USDT",
      "subject": "received",
      "data": {
          "symbol": "BTC-USDT",                        // symbol
          "sequence": 3262786900,                 // sequence
          "orderId": "5f0186a2d765040006f0b1fa",  // order ID
          "clientOid": "a4ba662f77180b69f186aa578aa52cb6", // optional, for you to identify your order
          "ts": 1545914149935808589
       }
    }
   ```
   
   通过clientOid，你可以判断这条RECEIVE消息是属于你自己的订单，此时你可以更新你本地的数据结构。如果你使用的是key-value结构，时间复杂度应该是O(1)，**如果在下单一定时间（最多1s）之后没有收到RECEIVE消息并且你收到的sequence是连续的，那么你的订单一定是下单失败了**，因此你需要做一些额外的处理。如果你已经收到RECEIVE消息了，代表是一定下单成功的。（**如果你使用的是私有频道，没有收到下单的推送，不一定是下单失败，还有可能是漏消息，但是公有频道由于sequence一定是连续的，所以在一定时间内没有收到下单推送，可以确定是下单失败，无需再用REST去补救查询**）。
   
   ```
   {
   "a4ba662f77180b69f186aa578aa52cb6": {
       "orderId": "5f0186a2d765040006f0b1fa",
       "size": "1",
       "status": "RECEIVE",
       "price": "9000",
       "side": "buy
       "symbol": "BTC-USDT",
       "dealSize": "0"
       }
   }
   ```
   
   由于相关订单的消息之后都会用orderId来关联，所以这里推荐你在本地再维护一个orderId为key的结构，类似这样：
   
   ```
   {
   "5f0186a2d765040006f0b1fa":
       {
       "orderId": "5f0186a2d765040006f0b1fa",
       "size": "1",
       "status": "RECEIVE",
       "price": "9000",
       "side": "buy
       "symbol": "BTC-USDT",
       "dealSize": "0",
       "clientOid":"a4ba662f77180b69f186aa578aa52cb6"
       }
   }
   ```
   
   
   之后只需要更新这个结构即可
   
   在收到RECEIVE消息之后，根据你订单的情况你还会收到OPEN、MATCH、UPDATE、DONE等各种类型的消息，其中
   
   * OPEN代表订单挂到了买卖盘之中；
   * MATCH代表成交；
   * UPDATE代表订单修改；
   * DONE代表订单结束，
   
   你只需要按照收到的消息更新你本地的订单信息即可。
   
   注意:
   
   1. MATCH会同时推送takerOrderId和makerOrderId，你可以自行判断你的成交方向；
   
   2. postOnly不代表你一定是maker只能保证你是maker费率成交。
   
      假设你postOnly 9000买入，此时买卖盘有一个9000的隐藏卖单，你也会直接成交，如果吃完隐藏单，并且这个价位没有非隐藏单的卖单，你剩下的单子会直接挂在买卖盘上面。这个MATCH事件你的方向是taker，收取maker手续费。
   
   3. 撤单之后一定会收到DONE消息，收到DONE消息一定代表撤单成功，无须REST再次查询状态或重复撤单，因为Level-3的推送变更远远快于REST查询。
   
   ### 其他问题
   
   1. 现货如何处理账务余额的变动
   
      你可以通过订阅我们的余额推送其中包含tradeId，能够和成交事件进行关联，你可以精确的知道你的撤单或者成交之后的每一笔钱的入账时间，能够再次被使用的时间。通常账务的延迟应该是在50ms左右。
   
   2. 期货仓位的处理
   
      你可以通过订阅我们的仓位推送来处理期货的仓位。
   
   3. 现货自成交的处理
   
      KuCoin支持自成交保护参数，你可以通过传入参数来规避自成交可能带来的问题。注意，只有taker方向订单的自成交参数是有效的。