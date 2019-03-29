1. **ssh**

1. 客户端-->跳板机-->目标机
```
ssh -A 用户名@跳板机ip
ssh -A 用户名@目标机器ip
```
常用参数：
1. -A 密钥转发
2. -i 指定密钥文件
3. -p 端口号
5. -C：请求压缩所有数据；

2. scp test1.txt supertool@y051:/home/supertool
-P  指明端口
-r  递归复制
-i  指明密钥文件


3. **rsync**

-t  不更新modify time
-z  压缩
-P  断点续传功能，大文件用到
-r  递归传递
-I  强制同步

4. **sz和rz**



