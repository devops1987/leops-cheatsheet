```bash


# 链接跟踪表
# -L 表示列表，-o 表示以扩展格式显示
$ conntrack -L -o extended | head
# 统计总的连接跟踪数
$ conntrack -L -o extended | wc -l
# 统计 TCP 协议各个状态的连接跟踪数
conntrack -L -o extended | awk '/^.*tcp.*$/ {sum[$6]++} END {for(i in sum) print i, sum[i]}'

# 统计各个源 IP 的连接跟踪数
conntrack -L -o extended | awk '{print $7}' | cut -d "=" -f 2 | sort | uniq -c | sort -n -r

# time_wait超时时间清理
sysctl net.netfilter.nf_conntrack_tcp_timeout_time_wait

traceroute --tcp -p 80 -n baidu.com # --tcp 表示使用 TCP 协议，-p 表示端口号，-n 表示不对结果中的 IP 地址执行反向域名解析
hping3 -c 3 -S -p 80 baidu.com # -c 表示发送 3 次请求，-S 表示设置 TCP SYN，-p 表示端口号为 80

# -c 表示并发请求数为 5000，-n 表示总的请求数为 10 万
# -r 表示套接字接收错误时仍然继续执行，-s 表示设置每个请求的超时时间为 2s
$ ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30/

# 默认测试时间为 10s，请求超时 2s
$ wrk --latency -c 1000 http://192.168.0.30

# 查看tc规则
tc -s qdisc show dev eth0

# 删除tc规则
tc qdisc del dev eth0 root netem loss 30%

```

## iptables

```bash
# 开启路由转发功能
sysctl -w net.ipv4.ip_forward=1

# 为一个子网配置snat
iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j MASQUERADE
# 为具体的 IP 地址配置 SNAT，并指定转换后的源地址
iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100

# 配置DNAT
iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-DNAT 192.168.0.2

# 双向地址转换
iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100
iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2
```

## Tcpdump

```bash

tcpdump -i ethO  # 指定网络接口，默认是0号接口eth0，any表示所有接口
tcpdump -nn  # 不解析IP地址和端口号的名称
tcpdump -c5 #  限制要抓取网络包的个数
tcpdump -A # 以ASCII格式显示网络包内容(不指定时只显示头部信息)
tcpdump -W fle.pcap # 保存到文件中，文件名通常以.pcap为后缀
tcpdump -e # 输出链路层的头部信息



tcpdump -nn host 35.1 90.27.188 # 主机过滤,host,src host,dst host
tcpdump -nn  192.1 68.0.0 # 网络过滤,net,src net,dst net
tcpdump -nn dst port 80 # 端口过滤,port,portrange, Src port,dst port
tcpdump -nn tcp # 协议过滤ip,ip6, arp, tcp,udp,icmp
tcpdump -nn icmp or udp # 逻辑表达式 and,or,not
tcpdump -nn "tcp[tcpflags] & tcp-syn != 0" # 特定状态的TCP包


# 输出格式
# 时间戳 协议 源地址. 源端口 > 目的地址. 目的端口 网络包详细信息

# 源地址是 10.5.2.3，目的端口是 3389 的数据包
tcpdump -nnvS src 10.5.2.3 and dst port 3389

#　从 192.168 网段到 10 或者 172.16 网段的数据包
tcpdump -nvX src net 192.168.0.0/16 and dat net 10.0.0.0/8 or 172.16.0.0/16

# 监听eth0网卡HTTP 80端口的request和response
tcpdump -i eth0 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'


# 监听eth0网卡HTTP 80端口的request(不包括response)，指定来源域名"example.com"，也可以指定IP"192.168.1.107"
tcpdump -i eth0 -A -s 0 'src example.com and tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'


# 监听本机发送至本机的HTTP 80端口的request和response
tcpdump -i lo -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'


# 监听eth0网卡HTTP 80端口的request和response，结果另存为cap文件
tcpdump -i eth0 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)' -w ./dump.cap
```

## ftrace

```bash
挂载
$ mount -t debugfs nodev /sys/kernel/debug

目录
/sys/kernel/debug/tracing
支持的跟踪器
/sys/kernel/debug/tracing/available_tracers
的函数和事件
/sys/kernel/debug/tracing/available_filter_functions /sys/kernel/debug/tracing/available_events

设置跟踪函数
echo do_sys_open > set_graph_function
开启函数调用跟踪，并跟踪调用进程
echo function_graph > current_trace
echo funcgraph-proc > trace_options
开启跟踪
echo 1 > tracing_on
关闭跟踪
echo 0 > tracing_on


trace-cmd record -p function_graph -g do_sys_open -O funcgraph-proc ls
trace-cmd report
```


## grep

```bash
grep -i -w vivek /etc/passwd   #搜索大小写任意的 vivek(即不区分大小写的搜索)
grep -E -i -w 'vivek|raj' /etc/passwd　#搜索大小写任意的 vivek 或 raj
```

```
[:alnum:] - 字母数字字符
[:alpha:] - 字母字符
[:blank:] - 空字符: 空格键符 和 制表符
[:digit:] - 数字: '0 1 2 3 4 5 6 7 8 9'
[:lower:] - 小写字母: 'a b c d e f g h i j k l m n o p q r s t u v w x y z'
[:space:] - 空格字符: 制表符、换行符、垂直制表符、换页符、回车符和空格键符
[:upper:] - 大写字母: 'A B C D E F G H I J K L M N O P Q R S T U V W X Y Z'
```

## sed

```bash
sed -n '{/access fail/{g;p}};h' access.log  # 打印匹配行的上一行
# sed 会逐行处理文件， 首先判断该行是否匹配（或包含）access fail关键字，
#　若没有匹配，仅执行h，即将该行以覆盖方式放到后台（一个缓冲区）；
# 若匹配上了，则执行后面的命令（{g;p}），g表示将后台内容挪到前台来（即将上一行覆盖当前行内容），p打印该行。


```

## sort

```bash
sort -te -k2,2n -k1,1n test.txt   # 排序浮点数
8.461754562e-07
8.959458896e-07
8.831553093e-06
8.387280091e-05
8.391373668e-05
8.547354437e-05
8.936111118e-05
# -te将你的数字分成两个字段,用e将尾数与指数分开. -k2,2表示首先按指数排序,然后-k1,1表示接下来用你的尾数排序.

```

## jq

> https://github.com/stedolan/jq/wiki/Cookbook

```bash
echo $json_data | jq -r '.status.phase'
echo $json_data | jq -r '.status.conditions[] | select(.status != "True") | "    type=\(.type), message=\(.message)"'
echo $json_data | jq -r '.spec.selector | to_entries | .[] | "\(.key)=\(.value),"'

jq -cn '[limit(5; range(2)|range(2))]'
printf '%s\n' '{"a":{"b":[{"c":0},{"d":1}]}}' | jq -c 'path(..)'

json='[{"genre":"deep house"}, {"genre": "progressive house"}, {"volume": "wubwubwub"}]'

echo "$json" | jq -c '.[] | select(.genre | . and contains("house"))?'
{"genre":"deep house"}
{"genre":"progressive house"}

cat file.json | jq '.[] | select(.age == 36)'
cat file.json | jq 'map({ _id, email })'
cat file.json | jq 'reduce .[] as $item (0; . + $item.age)'

```

## diff

```bash
diff -u old new | sed "s/^-/$(tput setaf 1)&/; s/^+/$(tput setaf 2)&/; s/^@/$(tput setaf 6)&/; s/$/$(tput sgr0)/"   # 颜色显示
```

## scp

```bash
# Usage
scp <options> source_path destination_path
# 常用选项
-r      # transfer directory 传输目录
-v      # see the transfer details 传输细节
-C      # copy files with compression 压缩传输
-l 800  # limit bandwith with 800   限制传输带宽
-p      # preserving the original attributes of the copied files 保持传输文件的属性
-q      # hidden the output 隐藏输出 比较少用
# 案例
scp file user@host:/path/to/file # 远程拷贝文件到host主机
scp -r /path/to/directory user@host:/path/to/directory # 远程拷贝目录到host主机
scp user@host:/path/to/file /local/path  # 拷贝远程文件到本地
scp -r user@host:/path/ /local/  # 拷贝远程目录到本地

```