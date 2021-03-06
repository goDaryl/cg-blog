# 详述银行卡支付方式

> **博主说**：在众多的支付方式中，银行卡支付是一种比较常见的支付方式， 其包括线下支付和线上支付两种，线下支付就是常见的 POS 机支付；线上支付主要为网银支付和快捷支付等。本文主要讲述了银行卡支付的几种方式以及对接银行接口时需要着重注意的一些点。


　　说说大家比较熟悉的银行卡支付，它分为线上支付和线下支付两种形式。线下支付就是通常说的 POS 收单，这里不介绍这个内容。对线上支付，按照卡的类别，分为贷记卡支付，也叫 motopay、ePOS，即信用卡支付；和借记卡支付。按照支付形态，又分为认证支付、网银支付、快捷支付几种形态。银行卡网银支付要求银行卡必须开通在线支付功能，而快捷支付并不需要开通在线支付功能。主要利用支付验证要素（卡号、密码、手机号、CVN2、CVV2 等），结合安全认证（例如短信验证码），让持卡人完成互联网支付。

## 1 认证支付

　　指用户在绑卡时，将卡信息提供给电商。这样在支付时，用户无需再输入这些信息，由电商在服务器侧保留用户的账户信息，比如身份证号、卡号、手机号。在用户支付时，无需再输入这些内容，最多就提供个密码或者校验码，就可以完成支付。这基本不会打断用户的使用体验，所以也是电商喜欢的支付方式。但认证支付最让人诟病的就是安全性。一方面需要向电商暴露个人信息，一旦被窃取，资金就容易被盗走。还有在手机上执行支付，一旦手机丢失，窃取者就可以轻而易举的使用或者转移资金。

## 2 快捷支付

　　快捷支付和认证支付类似，不同点在于绑卡之后，有些银行接口会返回`token`，后续使用`token`来作为支付凭证，无需提供卡号信息，这样电商也不需要本地保留卡号了。目前主要是银联有提供`token`接口。

## 3 网银支付

　　相对来说，网银支付要安全很多。网银支付是由银联或者银行提供支付界面，用户必须在页面上输入卡号，密码等验证信息才可以执行支付。大部分银行还要求用户使用 U 盾或者其它安全硬件。但安全和易用永远是个矛盾。网银使用会打断用户体验，增加用户使用难度。对使用硬件加密的支付，不可能天天带着 U 盘跑。另外网银主要用在 Web 端，在手机端，嵌入网银页面，还是比较难看的。

## 4 支付流程
走一个具体的例子看看吧！比如用户在电商系统中买了 200 块钱的东西，然后通过浦发银行卡做结算，用的是快捷支付。这个过程是：

 -  **第 1 步**：用户在交易界面上，提交订单到交易系统中；交易系统确认订单无误后，请求支付系统进行结算。这是在交易系统做的，后面工作就进入支付系统。
 - **第 2 步**：用户被引导到收银台页面， 让用户确认交易金额，选择支付方式，调用支付系统接口。
 - **第 3 步**：支付系统接收到支付请求，验证请求的各个字段是否有问题，确认无误后，调用支付网关执行支付。
 - **第 4 步**：支付网关请求浦发银行的快捷支付接口执行支付。
 - **第 5 步**：支付网关接收到支付结果报文后，对结果报文做解析，获取结果，并将结果告知交易系统。这可以通过 URL 或者 RPC 调用来实现。
 - **第 6 步**：商城系统收到支付结果后，开始执行后续操作。如果是支付成功，则开始准备出库。这一步在交易系统中处理，这里不做介绍。

网银支付和快捷相比，就是在第 4 步之前插入一个步骤，将用户导航到网银页面输入支付信息，后续步骤是一样的。在资金流上也是相同的。 而在第 5 步获取返回结果上，一般银行就直接同步返回，银联是分为同步和异步返回。同步告知操作成功或者失败，异步告知扣款成功或者失败。同步操作和异步操作都需要调用方提供一个回调的 URL 地址，银联会将参数附加在这个地址上。通过解析这些参数可以得到执行结果。异步操作一般有 2-3 秒的延迟，取决于网络，以及该交易处理的复杂度。

## 资金流

　　上一节说的是支付的信息流，那资金流应该是怎么走的？ 在第 3 步，会触发资金流，资金从用户个人账户上转移到电商公司的账户。当然，银行也不是活雷锋，这一笔交易是要收手续费的。资金是实时到账的，手续费一般是按月结算。有按交易笔数计费的，但大部分还是按照交易金额来收费。

　　同行快捷支付是比较简单的场景，让我们来逐步增加难度。如果支付系统没有对接浦发银行，那对浦发卡，就得走其它支付方式：银联或者第三方支付。

　　先说银联快捷。银联提供的多种接入方式，常说的快捷支付，在银联文档中叫商户侧开通 token 接口。通过这个接口，可以实现同行和跨行资金结算。不管收款行是浦发还是其它行，都可以完成结算。对本地和用户来说，体验是一样的。而在银联侧，后台资金流处理却不一样。了解这个资金流，有助于在异常情况下，了解资金到底跑到哪里了。

　　如果收款行也是浦发银行，银联发报文给浦发，浦发使用内部系统完成两个账户间的转帐，即时完成。

　　如果收款行是他行，比如工行。银联发指令给浦发和工行，分别完成各自账户上资金余额的增减，对个人和电商来说，这笔资金算是落地了。但实际资金流并不是立即发生。银联会在半夜做清结算后处理这笔资金。这个过程就是金融机构之间的清结算了，一般不需要关注。

　　如果使用的是第三方支付，对用户来说，处理的流程和银联一样。但资金流会不一样。 第三方支付在浦发和工行一般都会有落地的托管资金。 发生交易后，一般来说不会产生跨行资金流动。用户在浦发行的钱会被结算到第三方支付在浦发行的托管账户，而在工行的钱，会由第三方支付在工行的账户打到客户账户上。 这就降低了跨行资金流动成本。

　　目前国内主要银行都提供快捷和直联的接口。对电商来说，要对接哪些银行是个需要考虑的问题。

## 银联 token 支付

　　一般来说，大部分银行都提供直联和网银接口，但不需要直接对接所有银行。银联和第三方支付也提供直联接口，可以直接对接国内主要银行。也不是所有银行都被银联支持，这和银联签约的接口有关，需要在对接时咨询银联。从我们使用情况看， 浦发借记卡、邮储银行卡是不支持的。 另外交行、平安（含原深发）、上海银行、浦发、北京银行，上述银行卡需要开通银联在线支付业务。

## 对接银行

　　大部分银行提供的银行卡支付接口，借记卡支付和贷记卡支付是不一样的。但也有几个好心的银行，可以用一套接口同时开通借记卡和贷记卡。点名赞一下这些银行： 宇宙第一大行工商银行和建设银行。其他同学对接中如果也发现借记卡和贷记卡用一个接口的，也请及时告知。 作为国内最保守的软件团队，和银行对接时务必做好足够的准备。在商务谈判完成、拿到银行的接口文档后，需要考虑两个问题：**专线问题**和**加密问题**。

## 卡BIN

　　对接银行有一个逃避不了的问题，就是如何根据卡号判断发卡行？这就需要卡 BIN。 BIN 号即银行标识代码的英文缩写。BIN 由 6 位数字表示，出现在卡号的前 6 位，由国际标准化组织（ISO）分配给各从事跨行转接交换的银行卡组织。银行卡的卡号是标识发卡机构和持卡人信息的号码，由以下三部分组成：发卡行标识代码（BIN号）、发卡行自定义位、校验码。目前，国内的银行卡按照数字打头的不同分别归属于不同的银行卡组织，其中以 BIN 号`4`字打头的银行卡属于 VISA 卡组织，以`5`字打头的属于 MASTERCARD 卡组织，以`9`字和`62`、`60`打头的属于中国银联，而`62`、`60`打头的银联卡是符合国际标准的银联标准卡 ，可以在国外使用，这也是中国银联近几年来主要发行的银行卡片。 大部分银行卡号前 6 位即可确定发卡行和卡类型，但也有非标卡需要 6-10 位才可以判断出来。需要维护一个卡 BIN 库。「[银行卡 BIN 库](http://download.csdn.net/detail/qq_35246620/9919946)」是一个比较完整的卡 BIN 库， 为 CSV 格式的，有需要的同学可以点击免费获取。

![LUHN](http://img.blog.csdn.net/20170803144525817)

上述 LUHN 验证方式，是一种验证银行卡号是否合法的算法，其具体步骤为：

 1. 从卡号最后一位数字开始，逆向将奇数位（例如 1、3、5…）相加。
 2. 从卡号最后一位数字开始，逆向将偶数位数字先乘以 2，如果乘积结果为两位数，则将其减去 9，再求和。
 3. 将奇数位总和加上偶数位总和，结果应该可以被 10 整除。

在加卡的时候先做 LUHN，可以排除一部分输错卡号的情况。但要注意，国内有些银行的卡号不符合上述各种规定。

## 短信和身份验证

　　一般绑卡操作第 5 步需要银行下发短信验证码。 短信验证的接口，不同银行还不一样。有些银行是短信和身份验证一起做了；有些银行是可以配置身份验证是否同时发短信。还有些比较奇葩的机构，比如某联，接口中让你传身份信息，但实际上没传也是可以的，也不验证身份信息到底对不对。这在对接渠道时需要特别注意。

此类接口一般包含如下内容：

 - 版本号：当前接口的版本号；
 - 编码方式： 默认都是 UTF-8，指传输的内容的编码方式；
 - 签名和签名方法： 生成报文的签名。 不是所有的字段都需要放到签名中，文档中会说明哪些字段需要签名；
 - 签名算法：生成签名的算法，RSA、RSA128、MD5 等；
 - 商户代码：在渠道侧注册的商户号；
 - 商户订单号：即发送给渠道的订单号；
 - 发送时间：该请求送出的时间；
 - 账号和账号类型：银行卡、存折、IC 卡等支持的账号类型以及对应的账号；
 - 卡的加密信息：如信用卡的 CVN2，有效期等；
 - 开户行信息：开户行所在地以及名称，大部分是不需要的；
 - 身份证件类型和身份证号：可以用于实名验证的证件，指身份证、军官证、护照、回乡证、台胞证、警官证、士兵证等，不同银行可以支持的证件类型不一样，这也不是问题，大部分就是身份证了；
 - 姓名：真实姓名，必须和身份证一致；
 - 手机号：在所在银行注册的手机号。

系统会返回上述数据的验证结果。如果验证通过，则会发短信。但这不是所有的渠道都是这样。哪些字段会参与验证、需不需要发短信，需要注意看接口文档。

## 绑卡接口

　　绑卡接口和发短信接口类似，还需要将用户的卡号，身份证等信息传递过去。在绑卡成功后，会返回一个签约号。这个签约号是后续调用支付，解约等接口所必须的。这里有个问题，已经绑卡的用户，调用绑卡签约接口再绑一次，会出现什么情况？这个和银行实现有关。大部分银行，如农业、浦发、建行等，对绑卡签约接口调用，会首先验证身份信息，如果验证不通过，则不执行后续操作。验证通过后，再检查这个卡在该商户下是否已经绑过了，如果没有绑过，则执行绑卡，否则会提示卡已经绑定过了，不能重复签约。 但工行的实现不一样，他是首先验证这个卡是不是已经绑过了，如果已经绑卡，则不继续验证身份信息。 总之，银行都不支持重复绑卡。

## 银联绑卡

　　银联直联绑卡和银行绑卡类似，但是得注意验证接口，仅验证卡号和姓名，不验证身份证号和手机号。这导致第 5 步无法正常进行。银联只有到第 6 步执行绑卡时才做身份验证。 所以在处理上，还需要做一些调整，来确保和银行的流程的一致。一种处理方法是，对银联，在第 5 步就开始调用银联接口执行绑卡操作，但是在本地标记为预绑卡状态；商户侧发送短信验证码，验证通过后，才将状态设置为绑卡成功。

　　银联网银绑卡处理起来比较麻烦。用户在电商页面上输入卡号，然后被导航到银联页面上去完成绑卡操作，成功后，银联返回一个token作为签约号，用于支持后续操作。这问题就来了，用户可以在银联页面上绑定一个别人的卡，而电商侧是无法知道这个卡的情况的。所以这种方式尽量不要用。

## 实名认证

　　绑卡操作有个不错的副产品，就是实名认证。常说的二要素，三要素，四要素认证，可以通过这个操作完成。 二要素指姓名和身份证号，三要素加上银行卡号，四要素则加上手机号。看起来，似乎银行都应该支持四要素验证，但大部分银行接口仅支持三要素，毕竟手机号还是非常容易变。当然，实名认证，也就是二要素认证，是应用最多的认证了。国内唯一的库是在公安部这，由 NCIIC 负责对外提供接口。可以提供如下功能：

 - 简项核查：返回“一致”、“不一致”、“库中无此号”；
 - 返照核查：返回“一致 + 网纹照片”、“不一致”、“库中无此号”；
 - 人像核查：返回“同一人”、“不同人”、“库中无此号”。

官方接口收费是`5`元/条。 市面上主要的第三方服务提供商有国政通（简项、返照）、诺证通（简项）、IDface（三接口）等，收费简项核查：`0.5~2.0`元、返照核查为`0.8~2.1`元、人像核查`2.0~8.0`元不等。一般都和访问量有关，量大从优。当然，这里也要注意，涉密人员是没法查到相关信息的。 性能上， XX通一般在`200ms`内即可返回结果，普通商用应该是没问题的。有些公司还会额外提供四要素接口，以XX通为例，它号称支持大部分银行卡的四要素认证。但是实现上有点儿懵，居然是实时请求银行的接口，这就导致接口延迟非常高，`1`秒以上的占大部分，甚至`10`秒以上的都不少见，基本无法商用。这种情况下，还不如直接上银联呢！

**需要注意的问题：**

 - 验证短信由谁来发送？
 - 对接的是总行还是分行接口？
 - 是否支持重复绑卡，重复绑卡返回的 token 是一样的还是不同的？
 - 身份验证是三要素还是四要素验证？
 - token 和账号是否相同？


----------

**转载声明**：本文转自个人博客「凤凰牌老熊」，[银行卡支付](http://blog.lixf.cn/essay/2016/10/12/account-3-bank/)。

