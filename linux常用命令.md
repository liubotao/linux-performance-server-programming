## Linux常用命令

### 打包命令

    $ tar [-cxtzjvfpPN] 文件与目录
    
参数说明

*  -c : 建立一个压缩文件的参数指令(create)
*  -x : 解压一个文件的参数指令
*  -t : 查看tarfile里面的文件
*  -z : 是否具有gzip的属性
*  -j : 是否具有bzip2的属性
*  -v : 压缩的过程是否显示文件名
*  -f : 使用档名
*  -p : 使用原文件的原来属性
*  -P : 可以使用绝对路径来解压
*  -N : 比后面接的日期还要新的才打包

示例1:将整个/etc目录下的文件全部打包

    $ tar -cvf /tmp/etc.tar /etc  仅打包,不压缩
    $ tar -zcvf /tmp/etc.tar.gz /etc 打包,以gzip压缩
    $ tar -jcvf /tmp/etc.tar.bz2 /etc 打包后,以bzip2压缩
    
示例2:查看/tmp/etc.tar.gz文件内的文件
 
     $ tar -ztvf /tmp/etc.tar.gz
     
示例3:将/tmp/etc.tar.gz文件解压缩在/usr/local/src底下

    $ cd /usr/local/src
    $ tar -zxvf /tmp/etc.tar.gz 
    
示例4:tar

    $ 解压 tar xvf FileName.tar
    $ 打包 tar cvf FileName.tar DirName  
    
示例5:gz

    解压1：gunzip FileName.gz
    解压2：gzip -d FileName.gz
    压缩：gzip FileName
    
示例6:tar.gz和tgz
   
    解压：tar zxvf FileName.tar.gz
    压缩：tar zcvf FileName.tar.gz DirName
    
示例7:bz2

    解压1：bzip2 -d FileName.bz2
    解压2：bunzip2 FileName.bz2
    压缩： bzip2 -z FileName
    
示例8:bz

    解压1：bzip2 -d FileName.bz
    解压2：bunzip2 FileName.bz

### 网络命令行(CURL)

英文全称为Client URL Library Functions 使用 URL 语法传输数据的命令行工具,客户端向服务器请求资源的工具

示例1:查看网站CSS是否开启GZIP压缩

    curl -I -H "Accept-Encoding: gzip, deflate" "xxx/style.css"

### 排序和去重

    cut -d' ' -f1 ~/bash_histroy|sort -d|uniq -c|sort -nr|head
    
参数说明
  
*  cut -d' ' : -f1 从~/.bash_history文件分隔符(-d' ')剪出第一行
*  sort -d  : 按字段序(-d)排序剪出来的第一行
*  uniq -c : 对文本去重,并统计次数(-c)
*  sort -nr : 按数字排序(-n)并按降序(-r)排列,没有-r默认是升序排列
*  head : 输出显示文件前面部门的内容,默认线上前10行的内容