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
 - 我们首先看一个例子：
 - 1.php：
```php 
<?php
$a = 1;
```
 - 2.php：
```php
<?php
include "1.php";
$b = 2;
```
 - 那么我们通过gdb 2.php并分析它的op_array，可以得出它的指令，一共有3条：

>  ZEND_INCLUDE_OR_EVAL_SPEC_CONST_HANDLER：表示include "1.php";语句 
> ZEND_ASSIGN_SPEC_CV_CONST_RETVAL_UNUSED_HANDLER：表示$b = 2; 
> ZEND_RETURN_SPEC_CONST_HANDLER：表示PHP虚拟机自动给脚本加的返回值
 - 接下来我们深入第一个include的handler处理函数：
```c

static ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL ZEND_INCLUDE_OR_EVAL_SPEC_CONST_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
	USE_OPLINE
	zend_op_array *new_op_array;

	zval *inc_filename;

	SAVE_OPLINE();
	inc_filename = EX_CONSTANT(opline->op1); //这里的op1就是字符串1.php
	new_op_array = zend_include_or_eval(inc_filename, opline->extended_value);
	...
```
 - 这个handler处理函数中核心为zend_include_or_eval()这个函数，它返回一个新的op_array：
```c
static zend_never_inline zend_op_array* ZEND_FASTCALL zend_include_or_eval(zval *inc_filename, int type) /* {{{ */
{
	zend_op_array *new_op_array = NULL;
	zval tmp_inc_filename;

	...
	} else {
		switch (type) {
			...
			case ZEND_INCLUDE: //include语句
			case ZEND_REQUIRE: //require语句
				new_op_array = compile_filename(type, inc_filename); //核心调用
				break;
	...
	return new_op_array;
}
```
 - 我们可以看到，这里又调用了一个新的函数compile_filename()，它返回一个新的op_array。因为include是包含另一个外部文件，而op_array是一个脚本的指令集，所以需要新创建一个op_array，存储另外一个文件的指令集，我们继续跟进compile_filename()：
```c
zend_op_array *compile_filename(int type, zval *filename)
{
	zend_file_handle file_handle;
	zval tmp;
	zend_op_array *retval;
	zend_string *opened_path = NULL;
	
	...
	retval = zend_compile_file(&file_handle, type); //核心调用
	
	return retval;
}
```
 - 这个函数中还会继续调用zend_compile_file()函数，它是一个函数指针，指向compile_file()函数：
```c
ZEND_API zend_op_array *compile_file(zend_file_handle *file_handle, int type)
{
	...
	zend_op_array *op_array = NULL;

	if (open_file_for_scanning(file_handle)==FAILURE) {
		...
	} else {
		op_array = zend_compile(ZEND_USER_FUNCTION); //核心调用
	}

	return op_array;
}
```
 - 我们可以看到，它最终调用了zend_compile函数。我们是不是对它很熟悉呢？没错，它就是PHP脚本编译的入口。随后，通过调用这个函数，就可以对引入的外部脚本1.php进行词法分析和语法分析了。
 - 现在思考一个问题，这个函数返回一个op_array，是引入的新的外部脚本1.php的op_array，那么原来的旧脚本2.php的op_array的状态和数据应该如何存储呢？