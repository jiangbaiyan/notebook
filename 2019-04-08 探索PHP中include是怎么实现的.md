baiyan

全部视频：https://segmentfault.com/a/1190000018488313

## 回顾while的实现
```php
<?php
$a = 1;
while($a){
}
```
 - 在上一篇笔记中我们知道，PHP中的while语句所对应的指令执行过程如下图所示：
![enter description here](http://baiyanzzz.oss-cn-beijing.aliyuncs.com/2019/7/12/1562893925235.png)
 - 那么现在回答一下上一篇文章结尾提出的问题：do-while是如何实现的呢？
```php
<?php
$a = 1;
do{
	$a = 0;
}while($a);
```
 - 经过gdb调试，其最终的指令如下：
![](http://baiyanzzz.oss-cn-beijing.aliyuncs.com/2019/7/12/1562897928121.png)
 - 第一个ASSIGN对应$a = 1;
 - 第二个ASSIGN对应$a = 0;
 - 第三个JMPNZ对应do-while循环体，注意这个箭头指向$a = 0对应的ASSIGN指令，代表每次循环都要重新执行一次$a = -这个ASSIGN指令
 - 第四个RETURN对应PHP虚拟机自动给脚本添加的返回值
## include语句的实现