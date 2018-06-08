# OpenSourceWeatherStation 开源气象站
## 前言

从去年（2017年）下半年开始，我们所面向的行业开始向慢慢向农业靠拢，回顾这个历史有种冥冥之中注定的感觉。团队成员基本都是农村走出来的，小时候下地干农活没少干，说起农业感觉很亲切。因为上学，我们离农业离田地越来越远，小时候对农业的各种不理解也在深埋心底。

近些年来，我们团队的主要聚焦方向就是做设备的联网化（其实就是物联网的一部分，我不情愿来蹭这个热词），服务过的行业客户挺多的，对于农业一直在想是否能做点什么，但想归想，没行动。

涉足农业以后，我们深刻体会了中国的农业落后的现实，各类高上大的词汇现代农业、精准农业、农业自动化、农业信息化在仅仅停留在各类媒体和宣传口号中，现实生产中依然很残酷。据我们所知道的，我国主要粮食产区倒是实现了机械化规模化种植，这与近年来土地流转加速以及这些产区地理位置有关。

"靠天吃饭"是农业的真实写照，可见气象气候对于农业的影响之大，所以我们得先要了解气象。

开始想着看能不能找到共享气象数据，于是通过搜索引擎找到了[中国气象在数据交易平台](http://data.cma.cn/Market/Detail/code/A.0012.0001/type/1.html)，心想这下省事了，可是一看说明，全国只有2170个台站，这意味着基本上一个县一个地面站都不到，而且还是小时级别的数据，在云南这种”一山有四季，十里不同天“，一个县一个站小时级别的数据就是然并卵的。然后我们想着自己买来在需要的农场或基地装吧，淘宝一搜， 这好几千好几万的价格吓到我们了。

![在线气象站价格](https://i.loli.net/2018/06/08/5b1a4c50c45be.jpg)

研究了一下这些在线气象站，大概算了下材料成本，于是我们决定，自己做一个，并且开源！大家如果感兴趣可以下载我们的工程文件自行制作，或者可以找我们联系买齐所有材料，自己根据需求开发和组装。

## 需求

根据我们的业务需求，我们梳理下来，需要监测下面几类气象相关数据：

- 温度
- 湿度
- 气压
- 光照强度（Lux值）
- 紫外线强度
- 二氧化碳浓度
- 风速
- 风向

同时这些数据为了便于对温室内和大气中的数据进行比较，这些数据需要是大气中监测一组，温室内监测一组。

除此之外，还有下面一些开发需求：

- 无线通信
- 不能使用外接电源供电
- 要能够持续稳定运行
- 需要有支架
- 支架要运输和安装方便
- 安装时不用进行开挖或水泥打基础
- 方便扩展更多传感器
- 防水性能好
- 能通过WEB或微信小程序看实时和历史数据
- 数据的采集频率可调整

## 方案

### 开发平台选择

根据业务需要，梳理下来，这个东西使用Arduino来做就再合适不过了。支持的传感器和模块众多，上面的数据都有相应的模块可以完成测试和采集。

#### 为什么不用树莓派？

- 树莓派性能太富余了，这意味着更大的功耗量，更大的功耗意味着需要更大的供电系统，更大的供电系统意味着成本高
- 树莓派在面向硬件的开发时，资源不如Arduino多
- 树莓派比Arduino贵多了
- 树莓派运行着操作系统，但我们这个应用场景用不上

#### 选择Arduino for STM32

因为涉及多种数据的采集和缓存，还需要负责网络连接和数据收发，对于树莓派、PC、Android等平台的硬件来说，小菜一碟，但是对于常见的Arduino板，比如Uno或Mega，常用的MCU为Atmel 328p和Atmel 2560，这两个MCU无论从RAM和ROM上来讲都难以支持我们的应用，所以需要一个性能强劲一点的MCU，正好我们发现了[Arduino for STM32](http://www.stm32duino.com/)，这就相当于即可以享受STM32的高硬件扩展性又可以享受丰富的Arduino生态资源，在经过评估以后，我们认为`Arduino for STM32`可以用于进行气象站的开发。

### 传感器选择

选到最后的组合见下表：

| 参数     | 传感器        |
| ------ | ---------- |
| 温度     | SHT20      |
| 湿度     | SHT20      |
| 气压     | BMP180     |
| 光照强度   | TSL2561    |
| 紫外线强度  | GUVA-S12SD |
| 二氧化碳浓度 | CCS811     |
| 风速     | 外购         |
| 风向     | 外购         |

风速风向说起来原理并不复杂，但是因为涉及到的结构和校准问题，因此，我们就外购了，不过话说回来，风速风向这两个传感器占据我们这个项目的不小成本比例。

### 传感器与主板通信

设计上面表中的各类传感器使用兼容性适用性好的Modbus RTU通过RS-485标准介质传输。外购的风速和风向传感器已经是封装好的，已经支持Modbus RTU并留出了RS-485电缆接头。

除了风速风向以外，其他传感器均需我们自己设计RS-485介质传输的PCB进行封装。

### 供电

根据我们的业务需求，气象站要放野外使用，我们不能使用市电进行供电，因此只能通过风能、太阳能、地热能之类的新能源了。在我国光伏产能过剩的大背景下，光伏发电是不二之选，同时，光伏发电涉及的充电控制，电能储存等部分我们以前是有积累的，所以最终供电方案选成了

> 40W光伏板 + 7.4V 锂聚合物电池

### 数据传输

气象站在野外工作，自然也不适合使用WIFI或LAN进行数据通信。GPRS通信是理想选择。自2017年共享单车广泛投放以来，市场上一时出现了许多的GPRS模组方案，我们在之前的项目中用过SIMCom的SIM868，在尝试了不少方案后，我们选定了SIMCom的SIM868模组，经过我们的产品和应用实际验证，[SIM868](https://item.taobao.com/item.htm?spm=a1z10.3-c-s.w4002-16618966406.73.7eb569c4rJHuPV&id=567100266868)方案可靠、功耗低、功能丰富价格合适。而且数据流量费也挺划算的了，找移动开一个10元的套餐就能满足我们的数据流量需求。

 SIM868通过TCP/IP协议将数据传输至服务器端，服务器再对数据进行解析、存储和处理。

### 安装支架

最开始我们想着使用木头搭个架子来使用，因为我们有较齐全的木工工具，还有一些木材库存，但是回想起以前搭木架子的经历，瞬间对自制支架的牢固程度失去了自信，于是还是让同事设计个稳定的金属架子吧！

### 数据存储和呈现

因为之前有一些工作基础，所以这部分沿用以前的技术，服务器端可用的平台、工具、框架很多，都能满足要求，WEB端和小程序开发也不是难点，这个部分是纯软件工作， 要做的人按各自熟悉的套路来做即可。

## 开发过程

### 主板

选定了Arduino for STM32后，设计围绕STM32展开，我们选了`STM32F103RCT6`作为主控MCU。

因为使用GPRS进行数据传输，网络难免会遇上不可用或连接困难的时间，所以为了保证数据不丢失，我们加上了一块EEPROM用作数据暂存，EEPROM的型号为`AT24C1024`，1M存储空间。

传感器数据与时间是一一对应的，所以RTC也给配上了，型号为`DS3231MZ+`，本身`STM32F103RCT6`内置了RTC的，再加一块`DS3231MZ+`现在可以作为时间驱动的中断源。为了保证RTC时钟不停摆，我们给它加上了纽扣电池供电。

供电部分比较麻烦，虽说是太阳能供电，但是也需要考虑在紧急情况下的驱动，所以外接12V接口留了，光伏电池板输出不稳定，所以充电部分占了不小比重。在用电方面，三个供电电压，分别是5V、3.3V和4V。5V和3.3V常用，4V则需要着重说一下，4V是供给GPRS模组使用的，我们用的是[MIC29302WU](https://item.taobao.com/item.htm?spm=a230r.1.14.91.ae235aa5PR6cNz&id=554761942043)，如果给GPRS的供电能力不足，会导致GPRS模组运行不稳定，这个坑我们以前是踩过的，详见分享：[关于GPRS模块的电源设计经验分享](http://www.twiyale.cn/index.php/zh/news/9-gprs.html)；因涉及充电，并且充电电流较大，所以在PCB上充电部分和其他PCB接壤的地方挖了一些槽，方便散热。

 GPRS模组如上面方案所说使用的就是`SIM868`了，同时把SIM868的GSM和GPS外接天线引出连至IPEX座；SIM卡槽使用贴片封装的。

与传感器使用RS485通信，所以板载RS485转换芯片[ADM2483](http://www.analog.com/en/products/interface-isolation/isolation/isolated-rs-485/adm2483.html#product-overview)，这个部分前期设计处理上兼容性有些问题，在开始的试制设备中外购了一块RS485模块外挂在主板上使用；RS485总线上扩展了8路接口；RS485与MCU 的UART间设置隔离电路。

除了RS485作为有线扩展外，我们还预留了LoRa模块位，这部分现在还没用上。

连接接口方面相对于传统的Arduino开发板还是有不少区别的，首先在下载接口改为Micro-USB，这样的话，涉及供电连接的使用螺钉连接端子，开关使用了一个类似`WAGO 2060`的端快速连接端子，面板按钮、指示灯、RS485等使用贴片小端子。

原理图下载链接：[主板原理图](https://shhoo.cn/img/article/weatherstation/GPRS-ADM2483-V2.0.pdf)

PCB设计完成的3D图：

![PCB 3D图](https://i.loli.net/2018/06/08/5b1a4c1b2c0ce.png)

焊接最后的主板图：

![焊接好的主板](https://i.loli.net/2018/06/08/5b1a4c1606add.jpg)

### 多合一传感器（温度+相对湿度+气压+光照强度+紫外强度+二氧化碳浓度）

因为找到有堆叠式百叶箱防辐射外壳在卖，就是下面这种：

![百叶箱防辐射壳](https://i.loli.net/2018/06/08/5b1a4c193b3cb.jpg)

这种外壳能够多层装载，这意味着我们可以在上面装多块电路板，所以我们把温度、相对湿度、气压、光照强度、紫外强度、二氧化碳浓度这些传感器的PCB设计成可以装入这个壳里面。

我们使用Atmel 328P作为MCU，制成两块板，分别放于百叶箱壳中的上下两层，中间使用排线连接，主板通过RS-485与气象站主控板通信连接。

[传感器主板原理图下载链接](https://shhoo.cn/img/article/weatherstation/Multi-Sensor.pdf)。

[传感器板原理图下载链接](https://shhoo.cn/img/article/weatherstation/FourInOneSensorBoard.pdf)。

PCB设计完成的3D图：

![传感器主板](https://i.loli.net/2018/06/08/5b1a4c16e4ade.jpg)

![传感器板](https://i.loli.net/2018/06/08/5b1a4c1a0651f.jpg)

做好的样板照片就不发了，在装壳之前我们还刷了层防水漆，刷时得注意不能刷到传感器的感应孔里。

### 风速风向传感器

我们前后一共试用过两家的风速风向传感器，第一家是金属壳的，结果是还没到现场风向杯就掉了，修不好了，而且这家的风速风向都是侧出线的，这对于防水和安装有一些不便。

![风杯掉了](https://i.loli.net/2018/06/08/5b1a4bef10acd.jpg)

后面选了一家塑料的，底出线或下出线都可以，装上效果不错！

![新风速](https://i.loli.net/2018/06/08/5b1a4bedca569.jpg)

### 外壳和接插件

户外使用，防水防尘是对外壳的最基本要求，按我们以前的经验，如果自己开塑料模具来做，周期长，如果不是大批量生产，价格上并没有优势，而且气象站并不像手机之类的消费电子产品，对外观审美有要求，这就让我们有了很多的选择空间，我们最后选的是公壳。这个公壳防水的，有防水胶条和胶条槽。

![外壳防水胶条](https://i.loli.net/2018/06/08/5b1a4c318d888.jpg)

好处很明显，严丝合缝的，不用怎么担心防护问题，但需要自己在壳上开孔，不过我们有台钻，各种电动工具，所以这个问题不大。

![台钻](https://i.loli.net/2018/06/08/5b1a4beb6c18d.jpg)

![电动工具们](https://i.loli.net/2018/06/08/5b1a4bec2dbec.jpg)

开孔需要和航插配套，我们先后选过两种航插，综合价格、性能和易用性，我们选了下面这种。

![选定的航插](https://i.loli.net/2018/06/08/5b1a4beb7f7c4.jpg)

### 太阳能电池板和电池

我们大概初略算了一下，按照连续15天阴雨仍然可用的目标，我们配了一片40W的单晶硅光伏电池板。

![40W光伏板](https://i.loli.net/2018/06/08/5b1a4be98e088.jpg)

第一套气象站，我们配的是7.4V，14Ah的锂离子电池。第二套时发现这个容量有点大，所以把电池减成了7.4V 10Ah的聚合物锂电池。中顺芯家的锂电池不错，量足，价格能够接受。

![7.4V 10Ah锂离子电池](https://i.loli.net/2018/06/08/5b1a4c14bf442.jpg)

我们给光伏电池板接了一个2芯航插头，用这个航插头与主板盒上的航插母座连接。

### 安装支架

在使用木头搭架子的方案被否掉后，我们机械团队自行设计了架子，架子还在不断改进中，为的是方便运输和组装。

![支架设计图](https://i.loli.net/2018/06/08/5b1a4b932d41b.jpg)

支架设计高度2.5米，主杆可加长为更高的架子。主杆使用外径47mm，1.5mm壁厚不锈钢管，横杆使用外径38mm，1mm壁厚不锈钢管，主杆和横杆连接使用高强度抱箍连接，光伏电池板使用不锈钢板折弯焊接成框架，再通过横杆连接至主杆，气象站主板安装于光伏板背后，这样光伏板还起到一定的挡雨作用，光伏板的角度可以根据安装地的纬度进行调节；支架主杆下部设置有三角支架，三角支架可以折叠以方便运输，三角支架上有固定盘，上面开孔，使用地钉敲住固定在地上。总装图是下面的样子。

![气象站总装图](https://i.loli.net/2018/06/08/5b1a4b8e5632d.jpg)

### 嵌入式开发

这个部分工作因为使用了Arduino IDE，所以开发工作快了不少，分成了2大部分，一部分是气象站主板，另一部分是多合一传感器。

气象站主板使用Arduino for STM32的库，我们用下来感觉兼容性还不错。主板的作用是收集各个传感器的数据并将数据暂存或通过GSM网络上传至服务器。多合一传感器板的开发基于Arduino UNO板进行，因为本身我们多合一传感器板就是Arduino UNO的变体。开发过程不做过多赘述，就是常规的嵌入式开发。需要源码的可在我们Github仓库Clone。

### 服务器端及数据呈现软件开发

这部分就是纯软件工作了，现在各种平台，各种框架都能做这个，大概的架构是下面的这样的：

![软件架构](https://i.loli.net/2018/06/08/5b1a4b8588e28.jpg)

微信小程序免除了同时开发和维护IOS和Android平台应用程序的不便和高投入，而且微信小程序的接口用着还不错，性能和原生是有差距的，但能够接受。WEB端因为使用了Vue.js前端框架，所以开发速度也还不错。

这个部分开发坑不大，不多赘述。



## 组装

### 室内组装测试

我们在放到现场之前会先在室内组装测试，没问题拆了再拉到现场组装。

用的不锈钢管

![不锈钢管](https://i.loli.net/2018/06/08/5b1a4b8ec2eb0.jpg)

高强度不锈钢抱箍

![高强度不锈钢抱箍](https://i.loli.net/2018/06/08/5b1a4b8cea610.jpg)

室内组装好的支架

![室内组装好的支架](https://i.loli.net/2018/06/08/5b1a4ba95e250.jpg)

室内安装各个传感器

![室内安装各个传感器](https://i.loli.net/2018/06/08/5b1a4be9e7f1a.jpg)

室内总装完成

![室内总装完成](https://i.loli.net/2018/06/08/5b1a4be9e7f1a.jpg)

### 现场安装

运到现场安装，因为支架可以折叠，所以我们一行4个人再加所有设备材料一辆车就拉下了。

4个人+材料+设备一辆车就拉下了，第一套去了现场回来，成泥车了！

![车](https://i.loli.net/2018/06/08/5b1a4b2020875.jpg)

到现在为止我们一共装了两套，第一套装时正好是今年年初冬天，那是很冷的，外面零下2度，去的路上，路两边都是冰棱子，湿度还挺大的，这对于云南来讲是很冷了，这种天气少见，所以我们也没有常备厚衣服厚鞋子，在现场干活，那叫一个酸爽，冻死了！

车外零下2度，车内零上24度，幸福！

![车内暖和](https://i.loli.net/2018/06/08/5b1a4b1c0a1bc.jpg)

![路边树上的冰棱](https://i.loli.net/2018/06/08/5b1a4b1a51e60.jpg)

第一套现场组装过程

![第一套现场组装1](https://i.loli.net/2018/06/08/5b1a4baa4026d.jpg)

![第一套现场组装](https://i.loli.net/2018/06/08/5b1a53e36ed86.jpg)

注意到上面图中的风速风向传感器没有？是侧出线的，这个传感器装上用了一小阵就坏了，于是我们又重选了一种，再次到现场给换上，这次就挑了个晴天了，下面是换上后的照片。

![换上新的风速风向传感器]https://i.loli.net/2018/06/08/5b1a4ba9b0451.jpg)

有了第一套踩的坑，第二套安装就快多了。

![第二套安装](https://i.loli.net/2018/06/08/5b1a4b91b3ae2.jpg)

装在温室大棚内的百叶箱

![装在温室大棚内的百叶箱](https://i.loli.net/2018/06/08/5b1a4bd3a9b59.jpg)

第二套装好后

![第二套](https://i.loli.net/2018/06/08/5b1a4ba902d04.jpg)

安装时发现了吃玫瑰花的羊——鲜花羊

![鲜花羊](https://i.loli.net/2018/06/08/5b1a4bac77c79.jpg)



## 运行结果

从今年年初到现在我们一共在两个地方装了两套，从数据中我们发现了一些好玩的现象。

第一个现象是关于设备的GSM网络相对信号强度的(RSSI)的，见下图。

![信号强度](https://i.loli.net/2018/06/08/5b1a4b26cbe38.png)

从图中可以看出，信号强度在夜里的强度总体上要强于白天，我们在分析为什么，有的小伙伴认为夜里网络不拥挤，信号好，有的小伙伴认为白天可能有电磁波干扰……所以，大家可以夜里起床看看手机信号是不是要比白天好，哈哈～～

第二个是，判断是过去某天是晴天还是阴天，通过光照强度就比较好判断了，看下面的图。

![光照强度与天气](https://i.loli.net/2018/06/08/5b1a4b22218ea.png)

很明显上面图中4月6日是阴天，和现实情况一样，而4月8日则是大晴天，因为其光照强度曲线平滑，而4月10日是多云天气，因为光照强度因为云层的遮挡而间歇性降低。我们在考虑着使用分类算法对气象站所在地的历史天气进行分类。

从3月份到现在，整体上气温也是不断在升高的。

![气温](https://i.loli.net/2018/06/08/5b1a4b282c12a.png)

太阳能电池板和锂电池配合良好，即使遇上2、3天阴雨天，电池也够用，基本上电池电压都稳定在8V以上，这样看下来，对供电系统来讲真正的挑战似乎还没到来！

![电压](https://i.loli.net/2018/06/08/5b1a4b1cdc785.png)

还有其他一些好玩的现象就不多讲了。

我们的微信小程序也让数据的使用更方便了呢！小程序截图在下面。

![微信小程序](https://i.loli.net/2018/06/08/5b1a4b29272bd.png)



## 总结及开源

这个项目我们是以做产品的标准来做的，从整体周期上来看，大约4个人总的花了3个月左右吧，感觉速度还是可以的，中途一些加工步骤需要等，好在没有踩过太大的坑，算下来最坑的是第一次买的风速风向传感器，其他都在预想的波折范围内。

我们打算把这个项目中涉及的<u>**各类电路板和核心源代码**</u>开源，以方便各位需要使用在线气象站但是又觉得市场上的在线气象站太贵的朋友们自行制作，当然，你们要是懒得制作，也可以找我们买材料自行组装和改造！价格肯定远低于市场价格！

> Clint师傅
>
> Email: 187271113@qq.com
>
> Kady Yang
>
> Email: 625799663@qq.com
