# tcpdump/Wireshark网络分析

## Q21: 如何使用tcpdump/Wireshark分析网络问题？

**答案：**

```bash
# 1. tcpdump抓包
# 抓取指定端口的TCP包
tcpdump -i eth0 -n 'tcp port 443'

# 抓取WebSocket包
tcpdump -i eth0 -n 'tcp port 443 and tcp[13] & 8 != 0'  # PSH标志

# 保存到文件
tcpdump -i eth0 -w capture.pcap 'tcp port 443'

# 2. Wireshark分析
# 打开pcap文件
wireshark capture.pcap

# 过滤条件：
# - tcp.port == 443
# - websocket
# - tcp.analysis.retransmission  # 重传包
# - tcp.analysis.duplicate_ack    # 重复ACK

# 3. 分析延迟
# Statistics -> IO Graphs
# 查看RTT: Statistics -> TCP Stream Graph -> Round Trip Time
```

[src: raw/ingested/tcpdump_Wireshark网络分析.md]