# 野火专业版IM服务单聊消息性能测试

## 环境准备
准备如下服务器资源：

| 用途 | 配置 | 数量 | 公网IP | 入访端口 | 出访端口 |
| ------ | ------ | ------ | ----- | ----- | ---- |
| IM服务 | 8C32G | 1 | 需要 |  80/1883/8888/18080 |  无 |
| 数据库 | 8C16G | 1 | 不需要 | 内网互通 |  内网互通 |
| 发送压测机 | 4C8G | 1 | 需要 | 无 |   80/1883/18080 |
| 接收压测机 | 4C8G | 1 | 需要 | 无 |   80/1883/18080 |

建议直接购买云服务器和云MySQL进行压测，云服务都是经过优化的，可以直接压测使用。如果是自建服务，需要从网上找一下系统优化方法优化系统配置（至少openfiles改成10万以上）。

测试过程中要用到手机连接接收消息，所以需要用到外网，需要尽可能用最大的网络带宽。建议使用云服务器的按量计费方式，按流量计费可以选择把带宽设置为最大。

操作系统使用CentOS，其它Linux系统也行，不建议使用非Linux系统。

## 安装依赖工具
野火IM服务只依赖Java，CentOS使用下述命令安装，其它系统请自行查找方法。
```
yum install java-1.8.0-openjdk-headless.x86_64
```

## 部署IM服务
购买完云服务器后，就得到了云服务器的公网IP地址，联系野火来获取此公网IP的试用软件包。把软件包上传到IM服务器，解压，然后分别配置做如下配置：
1. ```config/wildfirechat.conf```
```
server.ip ip地址为IM服务的公网IP
#使用mysql数据库
embed.db 0
## 各种限频的大小改为1000000
http.admin.rate_limit 1000000
client.request_rate_limit 1000000
## 消息队列大小改为10万，这样当手机离线后发送10万条以内消息，上线后可以都收到。
message.max_queue 100000
## 下面这个配置是关掉的，要打开
netty.epoll true
```
2. ```config/c3p0.xml```
把数据库信息填写正确

3. ```bin/wildfirechat.sh```
```
JAVA_OPTS="$JAVA_OPTS -Xmx24G"
JAVA_OPTS="$JAVA_OPTS -Xms24G"
```

修改完这些之后，进入bin目录执行 nohup ./wildfirechat.sh 2>&1 &，这样IM服务就部署完成了。

## 部署应用服务
在IM服务下载应用服务，地址在[http://static.wildfirechat.cn/app-server-release-0.58.tar.gz](http://static.wildfirechat.cn/app-server-release-0.58.tar.gz`)。下载后解压，修改config目录下的```im.propertie```：
```
im.welcome_for_new_user=
im.new_user_robot_friend=false
im.robot_welcome=
```

其中```im.welcome_for_new_user``` 和 ```im.robot_welcome``` 配置为空，则手机登陆后不会发送欢迎语，避免测试干扰。

修改配置之后，使用命令 ``` java -jar app-0.58.jar ``` 来启动应用服务。等手机登陆成功后再停掉应用服务。

## 连接手机
需要手机登陆测试账户，在测试过程中观察手机消息状态。修改手机客户端中的应用服务和IM服务地址为IM服务器的公网IP地址，然后用SuperCode 66666 登陆。登陆成功后就可以关掉应用服务了。

可以在应用服务日志中找到用户ID或者在IM服务数据库中的```t_user_session```表中找到用户ID。

## 配置发送服务
在专业版IM服务软件包目录下有个```stress_test```目录，目录中有说明下载工具地址，下载测试工具到发送服务器，解压后得到一个可执行程序```wfcstress```和一个配置文件```config.toml```。

修改配置文件，只留下 ***IMConfig*** 和 ***TestSingleMessageConfig*** 配置项，其它都删除，然后配置如下:
```
# IM服务基础配置
[IMConfig]
## Host为专业版授权地址，如果其它地址测测试失败。
Host = "IM服务公网ip地址"
## 如果更改多客户端绑定端口，请把HttpPort改为定制的端口。
HttpPort = 80
AdminPort = 18080
AdminSecret = 123456
## Lite是否轻量模式，在lite为true时，压测工具只发送不会接收消息。如果测试聊天室功能请记得关闭此开关。
Lite = true
## 消息内容
## 较短文本消息内容，12个汉字，长度36B
MessageContent = "这是一个很短的文本内容！"


# 测试单聊消息。测试程序会创建SendUserCount个客户端并建立长链接。每个客户端循环给ReceiveUserCount个接收用户发送单聊消息。接收用户不会创建客户端。
[TestSingleMessageConfig]
## 是否开启此项测试
Enable = true
## 测试发送用户的前缀，用户ID为 ${SendUserPrefix}_i，SendUserPrefix U_ ，则用户ID为：U_0，U_1,U_2...。
## 用户的clientid为 用户id加上 c_, 比如c_U_0, c_U_1, c_U_2...。
SendUserPrefix = "U_"
## 发送消息用户数量，一般这个数字不要太大。
SendUserCount = 200
## 用户id的起始数，如果您有多台电脑，需要确保用户id不能重复。
SendUserStart = 0

## 客户端逐个给接收用户发送消息，此配置为发送消息的间隔，单位为毫秒。
SendMessageInterval = 0

## 目标用户的前缀，用户ID为 ${ReceiveUserPrefix}_i，ReceiveUserPrefix RU_ ，则用户ID为：RU_0，RU_1,RU_2...。注意不能和发送者前缀相同。
ReceiveUserPrefix = "RU_"
## 目标用户的数量。实际目标用户的总数量为ReceiveUserCount + WatchUserList.count。
ReceiveUserCount = 999
## 用户id的起始数，如果您有多台电脑，需要确保用户id不能重复。
ReceiveUserStart = 0
## 观察者的用户id，可以登陆一个或者多个手机客户端，观察消息发送情况。
WatchUserList = ["手机账户的用户id"]

## 是否跳过确认直接开始测试
SkipConfirm = true

## 循环次数。发送消息的总数为：SendUserCount * (ReceiveUserCount + WatchUserList.count) * Loop
Loop = 50
```
上述配置中，host要改为IM服务的公网IP地址，观察者用户id要填写手机账户的用户id，如果有多个，用逗号分开。测试一次发送的消息总数为 200 * (999 + 1) * 50 = 1000万。

## 配置接收服务
同发送服务配置，把测试工具下载一份到接收服务器。解压后修改配置文件```config.toml```，只留下 ***IMConfig*** 和 ***TestLonglinkConfig*** 配置项，其它都删除，然后配置如下:
```
# IM服务基础配置
[IMConfig]
## Host为专业版授权地址，如果其它地址测测试失败。
Host = "IM服务公网ip地址"
## 如果更改多客户端绑定端口，请把HttpPort改为定制的端口。
HttpPort = 80
AdminPort = 18080
AdminSecret = 123456
## Lite是否轻量模式，在lite为true时，压测工具只发送不会接收消息。如果测试聊天室功能请记得关闭此开关。
Lite = true
## 消息内容
## 较短文本消息内容，12个汉字，长度36B
MessageContent = "这是一个很短的文本内容！"

# 测试长链接。测试程序会建立UserCount个客户端并建立长链接，保持ConnDuration时长后，测试结束。
[TestLonglinkConfig]
## 是否开启此项测试
Enable = true
## 测试用户的前缀，测试用户ID为 ${UserPrefix}_i，比如UserPrefix为 U_ ，则测试用户为：U_0，U_1,U_2...。
## 测试用户的clientid为 用户id加上 c_, 比如c_U_0, c_U_1, c_U_2...。
UserPrefix = "RU_"
## 测试用户数量，单台压测机器不要超过6W，建议设置为5W。
UserCount = 999
## 用户id的起始数，如果您有多台电脑，需要确保用户id不能重复。假如有3台压测机，每台5W个测试用户，第一台的范围是0-49999，第二台的范围是50000-99999，第三台是100000-149999，
## 则UserCount都配置为50000，第一台的UserStart设置为0，第二台的UserStart设置为50000，第三台设置为100000。如果有更多台以此类推。
UserStart = 0
## 长链接保持时长，单位是秒，一般保持30分钟即可。
ConnDuration = 1800000
## 是否跳过确认直接开始测试
SkipConfirm = true
```
上述配置中，host设置为IM服务公网IP地址。```UserPrefix```用户前缀设置为发送服务目标用户前缀```RU_```。用户数```UserCount```设置为发送服务目标用户个数999.

## 发送消息测试
发送消息测试只发送消息，消息目标用户都不在线（只有一个观察者可以在线或者后台），测试消息发送能力。在发送服务器上执行：
```
nohup ./wfcstress 2>&1 &
tail -f nohup.out
```
正确情况下会看到先把200个发送者连接成功，等待3秒后就开始发送消息。等待压测程序执行完成后，观察：
1. 查看测试程序最后的输出，看看测试是否成功，看一下发送的条数和发送的时间，从而得出发送的速率。
2. 压测程序结束后5秒钟内，IM服务的CPU压力是否降为接近为0，查看测试过程中CPU的平均利用率和CPU利用率是否平稳。
3. 压测程序结束后5秒钟内，数据库的CPU压力是否降为接近为0，查看测试过程中CPU的平均利用率和CPU利用率是否平稳。
4. 查看手机账户收到消息是否是一万条（200 * 50）。
5. 查看数据库t_messages_X(X的计算方法请参考[消息存储在数据库中的那张表中](https://docs.wildfirechat.cn/faq/server.html#Q_消息存储在数据库中的那张表中))的消息总是是否是一千万。
6. 查看数据库t_user_messages_X表，查看每张表的数据量是不是都是以万为单位（因为用户消息表是用户hash分表，每个用户接收1万条），或者把这128张表加起来看一下是否是2千万。

## 收发消息测试
收发消息测试同时使用2台压测服务器，一台用于发送，另外一台用于接收。先登陆接收服务器，执行命令:
```
nohup ./wfcstress 2>&1 &
tail -f nohup.out
```
查看日志，等待所有用户都连接上以后，再启动发送服务，跟发送消息测试一样，启动发送消息程序。等待结束以后，观察：
1. 查看测试程序最后的输出，看看测试是否成功，看一下发送的条数和发送的时间，从而得出发送的速率。
2. 压测程序结束后5秒钟内，IM服务的CPU压力是否降为接近为0，查看测试过程中CPU的平均利用率和CPU利用率是否平稳。
3. 压测程序结束后5秒钟内，数据库的CPU压力是否降为接近为0，查看测试过程中CPU的平均利用率和CPU利用率是否平稳。
4. 查看手机账户收到消息是否是一万条（200 * 50）。
5. 查看数据库t_messages_X(X的计算方法请参考[消息存储在数据库中的那张表中](https://docs.wildfirechat.cn/faq/server.html#Q_消息存储在数据库中的那张表中))的消息总是是否是一千万（如果发送消息测试完成之后再测试收发测试，消息总是是两千万)。
6. 查看数据库t_user_messages_X表，查看每张表的数据量是不是都是以万为单位（因为用户消息表是用户hash分表，每个用户接收1万条），或者把这128张表加起来看一下是否是2千万。（如果做过发送消息测试还需要考虑之前的数据)。
