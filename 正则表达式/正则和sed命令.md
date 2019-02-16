## 处理以下文件内容,将域名取出并进行计数排序,如处理:  
http://www.baidu.com/index.<a target="_blank" href="http://www.2cto.com/kf/qianduan/css/" class="keylink" style="border:none; padding:0px; margin:0px; color:rgb(51,51,51); text-decoration:none; font-size:14px">html</a>  
http://www.baidu.com/1.html  
http://post.baidu.com/index.html  
http://mp3.baidu.com/index.html  
http://www.baidu.com/3.html  
http://post.baidu.com/2.html  
得到如下结果:  
域名的出现的次数 域名  
3 www.baidu.com  
2 post.baidu.com  
1 mp3.baidu.com  
[root@localhost shell]# cat file | sed -e ' s/http:\/\///' -e ' s/\/.*//' | sort | uniq -c | sort -rn  
3 www.baidu.com  
2 post.baidu.com  
1 mp3.baidu.com  
[root@codfei4 shell]# awk -F/ '{print $3}' file |sort -r|uniq -c|awk '{print $1"\t",$2}'  
3 www.baidu.com  
2 post.baidu.com  
1 mp3.baidu.com  
## 用grep结合sed取出网卡的ip地址  
[root@jie1 ~]# ifconfig | grep -B1 "inet addr" |grep -v "\-\-" |sed -n -e 'N;s/eth[0−9].*\n.*addr:[0−9]{1,3}\.[0−9]{1,3}\.[0−9]{1,3}\.[0−9]{1,3}.*/\1 \2/p'  

## -e和;的区别


替换空行 ^\n