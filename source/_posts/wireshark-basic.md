---
title: Wireshark网络分析
date: 2020-02-22 20:25:49
tags: network; wireshark
---

### Wireshark基本使用

一个包称为帧更准确

#### 减小包的大小

为了减小抓包的数据大小，可以对抓包进行设置

1. 只抓包头。一般能抓到包的大小为1514字节，启用了Jumbo Frame之后可达9000字节以上。大多数情况只需要IP或TCP的头就足够了，具体应用数据都是加密的，一般不需要。`Capture-->Options `中设置`Limit each packet to `为80字节，这样TCP、网络层、数据链路层的信息都有了。如果还要看应用层的信息，可以适当调大到200字节

   新版本的wireshark中可以在`Capture-->Input`中的对应网络接口上设置Snaplen(B)的大小

   使用Tcpdump抓eth0上的每个包的前80个字节，并把结果保存到tcpdump.cap文件中`tcpdump -i eth0 -s 80 -w /tmp/tcpdump.cap` 

2. 只抓必要的包。让wireshark在**抓包时过滤**掉不需要的包。在`Capture-->Options-->Input`的Capture Filter中输入过滤条件。例如只查看ip为192.168.43.101的包可以输入`host 192.168.43.1`

   `tcpdump -i eth0 host 192.168.43.1 -w /tmp/tcpdump.cap`

   需要注意如果自己关注的包可能被过滤掉，例如NAT设备把关注的ip地址改掉了

#### 显示过滤 Display Filter

显示过滤可以在主界面上直接输入过滤条件

1. 协议过滤 

   已经定义好的协议直接输入协议名称即可。对与nfs挂载失败可以使用`portmap || mount`进行过滤

2. 地址过滤

   `ip.addr == 192.168.1.104 && tcp.port == 443`

   选择一个包后，可以右键选择follow，再选择一个这个包的协议，可以自动过滤出相关的包。

3. 使用系统右键功能 

   选择一个关注的数据包后，可以右键后，选择`Prepare as filter`,系统会自动提示当前提取的过滤条件，选择select之后，就会填入过滤条件输入框中。`Apply as filter`则是直接应用这个过滤

   右键列表中还有其他的filter可以使用

4. 对过滤后的包保存

   `File -> Export Specified Packets`，在对话框中可以选择勾选当前显示的包

#### 技巧

1. 标记数据包，在每个关注的操作之前发一个指定数据长度的ping命令，这样知道这个操作的数据包的范围，只需要找到这些ping的特殊的ip地址和对应的数据段的大小，就把所有的数据包分割开了

   ```shell
   ping 192.168.43.1 -n 1 -l 1
   操作1执行
   ping 192.168.43.1 -n 1 -l 2
   操作2执行
   ping 192.168.43.1 -n 1 -l 3
   ```

   

1. 设置时间格式

   可以通过`View-->Time display format->Date time of Day`把时间显示为当前系统的时间，而不出相对的时间

   如果分析其他时区的包文件，需要把本机的时区改为和当地的时区一致，这样不用再去进行时区换算 

1. 设置某种类型包的颜色

   可以通过`View-->Coloring Rules`设置每一种包的颜色，方便一下找到，例如默认的icmp的颜色为粉色

1. 待定



### 参考资料

* Wireshark网络分析就是这么简单










