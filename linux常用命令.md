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
 
   
## CURL网络命令行

示例1:查看网站CSS是否开启GZIP压缩
curl -I -H "Accept-Encoding: gzip, deflate" "xxx/style.css"
