---
layout: post
title: P4入门（三）--Source Routing脚本分析
categories: P4
description: 对Source Routing的脚本进行一下分析
keywords: P4 SDN Switch Route
---

　　在入门（一）中分析了p4源码，还遗留了一些问题，那么现在从启动、发送、接收脚本开始分析，一点点解开这些问题吧，不太会Python，尽量分析一下哈。


## 启动

### run_demo.sh

　　整个拓扑的启动就是在这个`run_demo.sh`中，来看看这里面都做了什么，代码不多，所以先上代码：

```
THIS_DIR=$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )

source $THIS_DIR/../../env.sh

P4C_BM_SCRIPT=$P4C_BM_PATH/p4c_bm/__main__.py

SWITCH_PATH=$BMV2_PATH/targets/simple_switch/simple_switch

CLI_PATH=$BMV2_PATH/tools/runtime_CLI.py

$P4C_BM_SCRIPT p4src/source_routing.p4 --json source_routing.json
# This gives libtool the opportunity to "warm-up"
sudo $SWITCH_PATH >/dev/null 2>&1
sudo PYTHONPATH=$PYTHONPATH:$BMV2_PATH/mininet/ python topo.py \
    --behavioral-exe $SWITCH_PATH \
    --json source_routing.json \
    --cli $CLI_PATH
```

　　一开始就是获取当前路径，然后执行`env.sh`脚本，这个脚本就是设置了`bmv2`和`p4c-bm`的路径，接下来设置了`P4C_BM_SCRIPT`这个变量，这算是一个解释器吧，将p4代码解释为json文件，接下来定义了交换机模型，使用的`bmv2`中的模型，然后是`cli`的路径。

　　上面定义了这个变量之后，接下来就是将p4源码解释为json文件，最后启动topo.py脚本，将刚刚的参数传入（交换机模型、json文件、cli）。

　　这里有一行`sudo $SWITCH_PATH >/dev/null 2>&1`，根据注释应该是让运行库可以提前先准备一下？`2>&1`是将错误输出重定向到标准输出。

> 关于这个交换机和cli，回头再来分析一下，先继续走下去

### topo.py

　　接下来就来到了这个topo.py的脚本，在这里面将会建立整个拓扑，一起来看看，首先是main函数，一开始先读文件生成拓扑。

```
nb_hosts, nb_switches, links = read_topo()

topo = MyTopo(args.behavioral_exe,
              args.json,
              nb_hosts, nb_switches, links)
```

　　第一行就是通过函数`read_topo()`读取拓扑，文件中储存的拓扑信息如下：

```
switches 3
hosts 3
h1 s1
h2 s2
h3 s3
s1 s2
s1 s3
s2 s3
```

　　这里面的意思就是3个交换机，3个主机，接下来是连接关系，读取之后就获得了交换机个数、主机个数、相互连接关系

#### 建立Topo

　　然后是实例化`Mytopo`类，继承自`Topo`类，其定义如下：

```
class MyTopo(Topo):
    def __init__(self, sw_path, json_path, nb_hosts, nb_switches, links, **opts):
        # Initialize topology and default options
        Topo.__init__(self, **opts)

        for i in xrange(nb_switches):
            switch = self.addSwitch('s%d' % (i + 1),
                                    sw_path = sw_path,
                                    json_path = json_path,
                                    thrift_port = _THRIFT_BASE_PORT + i,
                                    pcap_dump = True,
                                    device_id = i)
        
        for h in xrange(nb_hosts):
            host = self.addHost('h%d' % (h + 1))

        for a, b in links:
            self.addLink(a, b)
```

　　这类中添加了交换机、主机、连接关系，稍微看了一下mininet里面的Topo源码，`addHost`和`addSwitch`实际上都是添加了一个节点，只不过把switch额外标记了一下。

　　`addLink`用于将两个节点进行连接，源码里可选参数还有对应的port口，这里没有设置port口表示不理解后面怎么确定port口的，在调试时看到输出信息，内部是自动给那个端口以1、2、3顺序排序定义port编号的。

　　thrift_port是服务端口，用来后面对交换机进行设置的

> 源码倒是可以仔细看看，但是目前对Python太不熟悉了，

#### 创建mininet

　　接下来就是创建一个mininet实例

```
net = Mininet(topo = topo,
              host = P4Host,
              switch = P4Switch,
              controller = None )
net.start()
```

　　传入的参数就是刚刚建立的拓扑、主机数量、交换机数量，这个controller留待以后再来看看

#### 设置主机

　　创建了mininet并传入拓扑后，对主机进行设置

```
for n in xrange(nb_hosts):
        h = net.get('h%d' % (n + 1))
        for off in ["rx", "tx", "sg"]:
            cmd = "/sbin/ethtool --offload eth0 %s off" % off
            print cmd
            h.cmd(cmd)
        print "disable ipv6"
        h.cmd("sysctl -w net.ipv6.conf.all.disable_ipv6=1")
        h.cmd("sysctl -w net.ipv6.conf.default.disable_ipv6=1")
        h.cmd("sysctl -w net.ipv6.conf.lo.disable_ipv6=1")
        h.cmd("sysctl -w net.ipv4.tcp_congestion_control=reno")
        h.cmd("iptables -I OUTPUT -p icmp --icmp-type destination-unreachable -j DROP")
```

　　遍历每个host，通过get获取，然后进行设置。

　　`ethtool`操作将网卡的硬件offload关闭，offload是网卡的一种机制，将在协议栈中进行的IP分片、TCP分段、重组、checksum校验等操作，转移到网卡硬件中进行，降低系统CPU的消耗，提高处理性能。

> 这里为什么要把这个关闭呢，可能是因为是软件定义网络的原因？或者是因为这个实验的报文格式是自定义的？

　　然后通过`sysctl`将ipv6禁用，将ipv4的拥塞控制设置为reno模式。

> reno模式：TCP Reno implements an algorithm called Fast recovery. A fast retransmit is sent, half of the current CWND is saved as SSThresh and as new CWND, thus skipping slow start and going directly to Congestion Avoidance algorithm

　　`iptables`指令将类型为目标不可达的icmp报文丢弃

#### 设置交换机

　　这里对交换机进行设置

```
for i in xrange(nb_switches):
        cmd = [args.cli, "--json", args.json,
               "--thrift-port", str(_THRIFT_BASE_PORT + i)]
        with open("commands.txt", "r") as f:
            print " ".join(cmd)
            try:
                output = subprocess.check_output(cmd, stdin = f)
                print output
            except subprocess.CalledProcessError as e:
                print e
                print e.output
```

　　这里就是通过thrift_port指定交换机进行设定，建立一个cli子进程，设定json参数，输入指定为commands.txt中的内容。

　　一切准备就绪，执行cli，启动就完成了。

## 接收

　　接收比较简单，所以先拿出来讲，大致就是通过sniff抓取网卡eth0的报文，然后用自定义的解析函数进行解析然后打印内容，解析函数如下

```
def handle_pkt(pkt):
    pkt = str(pkt)
    if len(pkt) < 12: return
    preamble = pkt[:8]
    preamble_exp = "\x00" * 8
    if preamble != preamble_exp: return
    num_valid = struct.unpack("<L", pkt[8:12])[0]
    if num_valid != 0:
        print "received incorrect packet"
    msg = pkt[12:]
    print msg
    sys.stdout.flush()
```

　　这解析函数还是比较好看懂的，先校验前八个字节是不是0，然后8到12字节提取出来是num_valid，判断是不是0，这里转换数字是用小端unsigned long来转换的，最后就是12字节以后的就是内容了，将其打印，大功告成。

## 发送

　　最后是发送了啊，又是百来行，好多啊:joy::joy::joy::joy::joy:

　　read_topo()函数跟上面topo里面是一样的就不说了，`SrcRoute`类继承自Packet，就是自己定义了一个包类型了，里面包括一个8字节的前导码字段，4自己的计数字段。

### main函数

　　一开始就是读入需要传送数据的源主机和目的主机，然后搜索最短路径后不断读入数据生成报文再发送，我发现分析这个好像对p4没什么卵用，就不具体分析了。

