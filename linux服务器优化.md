## Linux服务器优化

### 调整Linux的最大文件数

修改/etc/rc.local文件打开数
    
    ulimit -SHn 65535
    
### 调整TCP参数

        $vim ／etc/sysct1.conf
        
	net.ipv4.tcp_fin_timeout = 30
	net.ipv4.tcp_keepalive_time = 1200
	net.ipv4.tcp_syncookies = 1
	net.ipv4.tcp_tw_reuse = 1
	net.ipv4.tcp_tw_recycle = 1
	net.ipv4.ip_local_port_range = 1024 65000
	net.ipv4.tcp_max_syn_baklog = 8192
	net.ipv4.tcp_max_tw_bukets = 5000
	
       $ /sbin/sysctl -p


##备注
  
 1) 截图zabixx里面对502 504监控 以及ES里面的watch脚本  
 2）如果是传统硬盘,inde   1 x数据的时候,一定要查看磁盘I/O使用率
 
     iostat -x 1 5
     
     如果util接近100%则说明产生的I/O请求太多,I/O系统已经满负载,磁盘可能存在瓶颈
     
     如果idle小于70%,I/O的压力比较大,说明读取进程中有较多的wait
     
  3）kafka的pations必须大于消费者的数量,logstash和kafka的pations配置
     
