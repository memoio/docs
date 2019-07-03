# mefs使用说明

+ mefs有三类角色: user, keeper, provider

user使用存储空间，需要为provider的存储、keeper的管理进行支付
provider负责存储数据
keeper负责管理：运行挑战，触发支付

## 账户申请与注册

1. 每个角色通过私钥（private key）唯一标识，对外用公钥（public key）作为名字，私钥用密码（password）加密后存放于keystore目录下；
2. 角色需要在链上注册，mefs启动的时候会根据所属角色启动不同的服务，一个地址只运行为一个角色。

## 安装与启动

### 安装

使用Docker启动

+ Dockerfile内容

```docker

FROM limcos/environment_construction:latest

MAINTAINER suzakinishi <ccyansnow@gmail.com>

# download mefs

RUN wget -P /usr/local/bin/ http://97.64.124.20:8000/mefs    \
 && chmod 777 /usr/local/bin/mefs

EXPOSE 4001
EXPOSE 5001
```

+ 镜像

```docker
docker build -t <image> .
```

+ 运行容器

```docker
docker run -itd -v <datadir>:/root/.mefs <image> -p 5001:5001
```

### 初始化

mefs会根据输入的私钥和密码进行初始化，默认初始化的目录为$HOME/.mefs，可以通过export MEFS_PATH=<datadir>的方式设置初始化的目录。

```shell
mefs init --sk=<private key> --pwd=<password>
eg: mefs init --sk=0x8a1539557a547f87edef7a4d4dcf12735db77fb37b3bc879e1ee354d837d5c23 --pwd=111111
```

参数解释：

```shell
--sk：用户的私钥，以0x开头的66位十六进制字符串，可用命令mefs create得到
--pwd：用户密码，由数字、字母组成的任意字符串
```

### 启动实例

使用密码即可启动服务

```shell
mefs daemon --pwd=<password>
eg: mefs daemon --pwd=111111
```

参数解释：

```shell
--pwd：用户密码，此密码即初始化时设置的密码
```

### 启动用户
在启动mefs实例后，启动用户。

```shell
mefs lfs start <addr>
```

参数解释：

```shell
addr：用户地址；为空时，启动本地用户；
```

## 使用LFS

mefs为每一个用户提供了一个专属的加密存储空间（LFS），每个存储空间包含多个桶（bucket），桶是用户用于存储对象（object）的容器，每个桶包含多个对象（object），我们可以把对象想象成文件。桶的冗余策略可以在创建的时候指定（存储在该桶中的所有对象都使用该种冗余策略），对象的数据使用对称加密方式加密。

user可以通过命令行，网络（http），以及网关（gateway）的方式进行数据的操作。

### 命令行（cli）

#### 桶（bucket）操作

+ 创建桶

命令描述：create_bucket根据BucketName名字创建桶，每个桶可以设置不同的冗余策略，冗余策略为多副本（multiple replicas）或者纠删码（Reed-Solomon Codes）,可以调整数据块和校验块的个数来决定冗余水平。默认使用3个数据块和2个校验块的纠删码，可以容忍两个块的丢失。

```shell
mefs lfs create_bucket <BucketName> --policy=<redundancy> --dc=<data count> --pc=<parity count> --addr=<public key>
```

参数解释：

```shell
BucketName: 桶的名字，最小3字节最大256字节；
--policy：冗余策略，--policy=0表示使用多副本，--policy=1表示使用纠删码，默认是使用纠删码；
--dc：数据块的个数，默认是3；
--pc：校验块的个数，默认是2；当使用多副本策略的时候，实际的数据块为1，校验块为dc+pc-1
--addr: user的地址，默认为空，表示用户为本地节点地址。
```

结果输出：

```shell
Method: Create Bucket    // 命令名称
BucketName: <BucketName> // 创建的桶的名字
--BucketID: <BucketID>  // 桶的内部ID
--Ctime: <Ctime> // 创建时间
--Policy: <Policy> // 冗余策略
--DataCount: <DataCount> // 数据块个数
--ParityCount: <ParityCount>  // 校验块个数
```

+ 桶列表

命令描述：list_buckets显示出此用户创建的所有的桶，包含每个桶的名字(BucketName)，创建时间(Ctime)，冗余策略(Policy)和冗余参数(DataCount、ParityCount)

```shell
mefs lfs list_buckets --addr=<public key>
```

参数解释：

```shell
--addr: user的地址，默认为空，表示用户为本地节点地址。
```

结果输出：

```shell
Method: List Buckets
BucketName: <BucketName>
--BucketID: <BucketID>
--Ctime: <Ctime>
--Policy: <Policy>
--DataCount: <DataCount>
--ParityCount: <ParityCount>
BucketName: <BucketName>
--BucketID: <BucketID>
--Ctime: <Ctime>
...
```

+ 桶信息

命令描述：若BucketName名字的桶存在，head_bucket显示此桶的创建时间，冗余策略和冗余参数；若不存在，返回桶不存在。

```shell
mefs lfs head_bucket <BucketName> --addr=<public key>
```

参数解释：

```shell
BucketName：桶的名字
--addr: user的地址，默认为空，表示用户为本地节点地址
```

结果输出：

```shell
Method: Head Bucket
BucketName: <BucketName>
--BucketID: <BucketID>
--Ctime: <Ctime>
--Policy: <Policy>
--DataCount: <DataCount>
--ParityCount: <ParityCount>
```

+ 删除桶（bucket）

命令描述：若BucketName名字的桶存在，delete_bucket删除此桶；若不存在，返回桶不存在。只有在桶内为空的时候才会删除，否则返回桶不为空。

```shell
mefs lfs delete_bucket <BucketName> --addr=<public key>
```

参数解释：

```shell
BucketName：桶的名字
--addr: user的地址，默认为空，表示用户为本地节点地址
```

结果输出：

```shell
Method: Delete Bucket
BucketName: <BucketName>
--BucketID: <BucketID>
--Ctime: <Ctime>
--Policy: <Policy>
--DataCount: <DataCount>
--ParityCount: <ParityCount>
```

#### 对象（object）操作

+ 上传文件

命令描述：put_object向BucketName桶内上传一个名为ObjectName的对象；若桶不存在，返回桶不存在；若对象已存在，则返回对象已存在。

```shell
mefs lfs put_object <ObjectName> <BucketName> --addr=<public key>
```

参数解释：

```shell
ObjectName：上传的数据；
BucketName：桶的名字；
--addr: user的地址，默认为空，表示用户为本地节点地址
```

结果输出：

```shell
Method: Put Object
ObjectName: <ObjectName>  // 对象的名字
--ObjectSize: <ObjectSize> // 上传的对象的大小
--Ctime: <Ctime>           // 此对象的创建时间
--Dir: <Dir>               // 是否是目录，true为目录，false为文件
--LatestChalTime:<LatestChalTime>  // 最近一次，此object被挑战的时间
```

+ 下载文件

命令描述：get_object从BucketName桶内下载一个名为ObjectName的对象；若桶不存在，返回桶不存在；若对象不存在，则返回对象不存在。

```shell
mefs lfs get_object <BucketName> <ObjectName> --o=<OutputName> --addr=<public key>  
```

参数解释：

```shell
ObjectName：下载的对象名称；
BucketName：桶的名字；
--o: 默认下载对象的名字为ObjectName，若设置此参数，则为OutputName
--addr: user的地址，默认为空，表示用户为本地节点地址
```

结果输出：

下载的文件

+ 文件列表

命令描述：list_objects列出BucketName桶内所有的对象，包括对象大小，创建时间，是否是目录，最近被挑战的时间。

```shell
mefs lfs list_objects <BucketName> --addr=<public key>
```

参数解释：

```shell
BucketName：桶的名字；
--addr: user的地址，默认为空，表示用户为本地节点地址
```

结果输出：

```shell
Method: List Object
ObjectName: <ObjectName>
--ObjectSize: <ObjectSize>
--Ctime: <Ctime>
--Dir: <Dir>
--LatestChalTime:<LatestChalTime>
ObjectName: <ObjectName>
--ObjectSize: <ObjectSize>
--Ctime: <Ctime>
......
```

+ 文件信息

命令描述：head_object显示BucketName桶内ObjectName对象的大小，创建时间，最近被挑战的时间；

```shell
mefs lfs head_object <BucketName> <ObjectName> --addr=<public key>
```

参数解释：

```shell
ObjectName：下载的对象名称；
BucketName：桶的名字；
--addr: user的地址，默认为空，表示用户为本地节点地址
```

结果输出：

```shell
Method: Head Object
ObjectName: <ObjectName>
--ObjectSize: <ObjectSize>
--Ctime: <Ctime>
--Dir: <Dir>
--LatestChalTime:<LatestChalTime>
```

+ 删除文件

命令描述：delete_object从BucketName桶内删除ObjectName对象。

```shell
 mefs lfs delete_object <BucketName> <ObjectName> --addr=<public key>
```

参数解释：

```shell
ObjectName：下载的对象名称；
BucketName：桶的名字；
--addr: user的地址，默认为空，表示用户为本地节点地址
```

结果输出：

```shell
Method: Delete Object
ObjectName: <ObjectName>
--ObjectSize: <ObjectSize>
--Ctime: <Ctime>
--Dir: <Dir>
--LatestChalTime:<LatestChalTime>
```

#### 角色操作

+ 列出对应的keeper

命令描述：list_keepers列出与此user签署UpKeeping合约的keeper

```shell
mefs lfs list_keepers  
```

输出为对应的keeper id

+ 列出对应的provider

命令描述：list_providers列出给此user存储数据的provider

```shell
mefs lfs list_providers
```

输出为对应的provider id

#### 其他

+ 刷新元数据

命令描述：fsync手动刷新lfs的元数据，此命令在keeper上执行。元数据包括SuperBlock、BucketInfo、ObjectInfo

```shell
mefs lfs fsync
```

输出为flush success

+ 查询余额

命令描述：show_balance查询对应地址的余额

```shell
mefs lfs show_balance --addr=<public key>
```

参数解释：

```shell
--addr: user的地址（以0x开头），默认为空，表示用户为本地节点地址，即查询本地节点的余额
```

输出为对应的金额

+ 查询使用的储存空间

命令描述：show_storage查询用户的使用空间，即用户的所有bucket中共存储了多少数据，单位是kb

```shell
mefs lfs show_storage --addr=<public key>
```
参数解释：

```shell
--addr: user的地址，默认为空，表示用户为本地节点地址
```

输出为相应的空间，格式为两位小数带单位（kb）

### http操作

mefs的命令都可以使用http进行操作

#### 配置

在mefs启动之前，进行如下配置：

``` shell
// mefs api的端口设置,默认为5001
mefs config Addresses.API /ip4/0.0.0.0/tcp/5001

// 跨域访问
mefs config --json API.HTTPHeaders.Access-Control-Allow-Origin '["*"]'
mefs config --json API.HTTPHeaders.Access-Control-Allow-Methods '["PUT", "GET", "POST"]'
```

然后启动mefs即可使用http方式进行操作。

#### example

+ 显示所有bucket的信息

```shell
curl  "http://127.0.0.1:5001/api/v0/lfs/list_buckets?"
```

输出是标准的json格式

```json
{
    "Method": "List Buckets",
    "Buckets": [{
        "BucketName": "<BucketName>",
        "BucketID": "<BucketID>",
        "Ctime": "<Ctime>",
        "Policy": "<Policy>",
        "DataCount": "<DataCount>",
        "ParityCount": "<ParityCount>"
    }, {
        "BucketName": "<BucketName>",
        "BucketID": "<BucketID>",
        "Ctime": "<Ctime>",
        "Policy": "<Policy>",
        "DataCount": "<DataCount>",
        "ParityCount": "<ParityCount>"
    }]
}
```

+ 显示某bucket的所有object的信息

```shell
curl  "http://127.0.0.1:5001/api/v0/lfs/list_objects?arg=<BucketName>"
```

输出是标准的json格式

```json
{
    "Method": "List Objects",
    "Objects": [{
        "ObjectName": "<ObjectName>",
        "ObjectSize": "<ObjectSize>",
        "Ctime": "<Ctime>",
        "Dir": "<Dir>",
        "LatestChalTime": "<LatestChalTime>"
    }, {
        "ObjectName": "<ObjectName>",
        "ObjectSize": "<ObjectSize>",
        "Ctime": "<Ctime>",
        "Dir": "<Dir>",
        "LatestChalTime": "<LatestChalTime>"
    }]
}
```

#### 使用

一个类似于如下的命令：

```shell
mefs rootcmd subcmd arg1 op1=arg2
```

对应的http请求为：

```shell
curl  "http://<ip>:5001/api/v0/api/v0/<rootcmd>/<subcmd>?arg=<arg1>&op1=<arg2>"
```

ip为启动mefs的机器的网络地址，若运行前配置了跨域访问，可以使用外网ip进行访问，否则只能通过127.0.0.1访问。

### 网关模式

多个用户可以共用一个mefs的运行程序

#### user代理启动

在启动mefs后，也可以代理启动其他的用户。

```shell
mefs lfs start <addr> --sk=<private key> --pwd=<password>
```

参数解释：

```shell
addr：用户地址;
--sk：用户的私钥;地址对应的私钥，若何不匹配，以私钥的地址为准；
--pwd：用户密码;
```

#### user代理关闭

在启动mefs后，也可以代理关闭其他的用户。

```go
mefs lfs kill addr --pwd=<password>
```

参数解释：

```shell
addr：用户地址
--pwd：用户密码
```

#### 使用

+ cli

```shell
mefs rootcmd subcmd arg1 op1=arg2 --addr=<public key>
```

+ http

```shell
curl  "http://<ip>:5001/api/v0/api/v0/<roocmd>/<subcmd>?arg=<arg1>&op1=<arg2>&addr=<public>"
```

#### example

用户pk从BucketName桶内获取ObjectName名字的文件

+ cli

```shell
mefs lfs get_object <BucketName> <ObjectName> --addr=<pk>
```

+ http

```shell
curl  "http://127.0.0.1:5001/api/v0/lfs/get_object?arg=<BucketName>&arg=<ObjectName>&addr=<pk>"
```