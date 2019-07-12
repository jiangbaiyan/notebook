baiyan

全部视频：https://segmentfault.com/a/1190000018488313

## 回顾while语法的实现
```php
<?php
$a = 1;
while($a){
}
```
 - 在上一篇笔记中我们知道，PHP中的while语法所对应的指令执行过程如下图所示：
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
## include语法的实现
 - 我们在面试中经常会被问到如下知识点:
     - include和require有什么区别？
     - include和include_once有什么区别（require和require_once)同理）
 - 以上两道题的答案相信大家都知道，第一个问题如果文件不存在，include会情况下会发出警告而require会报fatal error并终止脚本运行；而第二个问题中的include_once带有缓存，如果之前加载过这个文件直接调用缓存中的文件，不会去二次加载文件，include_once的性能更好。
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

>  - ZEND_INCLUDE_OR_EVAL_SPEC_CONST_HANDLER：表示include "1.php";语句 
>  - ZEND_ASSIGN_SPEC_CV_CONST_RETVAL_UNUSED_HANDLER：表示$b = 2; 
>  - ZEND_RETURN_SPEC_CONST_HANDLER：表示PHP虚拟机自动给脚本加的返回值
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
{
			case ZEND_INCLUDE_ONCE:
			case ZEND_REQUIRE_ONCE: { //此处带有缓存
					zend_file_handle file_handle;
					zend_string *resolved_path;

					resolved_path = zend_resolve_path(Z_STRVAL_P(inc_filename), (int)Z_STRLEN_P(inc_filename));
					if (resolved_path) {
						if (zend_hash_exists(&EG(included_files), resolved_path)) {
							goto already_compiled;
						}
					} else {
						resolved_path = zend_string_copy(Z_STR_P(inc_filename));
					}

					if (SUCCESS == zend_stream_open(ZSTR_VAL(resolved_path), &file_handle)) {

						if (!file_handle.opened_path) {
							file_handle.opened_path = zend_string_copy(resolved_path);
						}

						if (zend_hash_add_empty_element(&EG(included_files), file_handle.opened_path)) { //加入缓存的哈希表中
							zend_op_array *op_array = zend_compile_file(&file_handle, (type==ZEND_INCLUDE_ONCE?ZEND_INCLUDE:ZEND_REQUIRE));
							zend_destroy_file_handle(&file_handle);
							zend_string_release(resolved_path);
							if (Z_TYPE(tmp_inc_filename) != IS_UNDEF) {
								zend_string_release(Z_STR(tmp_inc_filename));
							}
							return op_array;
						} else {
							zend_file_handle_dtor(&file_handle);
already_compiled:
							new_op_array = ZEND_FAKE_OP_ARRAY;
						}
					} else {
						if (type == ZEND_INCLUDE_ONCE) {
							zend_message_dispatcher(ZMSG_FAILED_INCLUDE_FOPEN, Z_STRVAL_P(inc_filename));
						} else {
							zend_message_dispatcher(ZMSG_FAILED_REQUIRE_FOPEN, Z_STRVAL_P(inc_filename));
						}
					}
					zend_string_release(resolved_path);
				}
				break;
			case ZEND_INCLUDE:
			case ZEND_REQUIRE:
				new_op_array = compile_filename(type, inc_filename);  //关键调用
				break;
	...
	return new_op_array;
}
```
 - 在ZEND_INCLUDE_ONCE分支中可以观察到，如果是include_once或者require_once的case，会先去缓存中查找，那么这个缓存是怎么实现的呢？最容易想到的就是**哈希表**，key为文件名，value为文件内容，这样就可以直接从缓存中读取文件，不用再次加载文件了，提高效率。
 - 我们回到主题include语法，在ZEND_INCLUDE分支中我们可以看到，这里又调用了一个新的函数compile_filename()，它返回一个新的op_array。因为include是包含另一个外部文件，而op_array是一个脚本的指令集，所以需要新创建一个op_array，存储另外一个文件的指令集，我们继续跟进compile_filename()：
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
 - 我们可以看到，它最终调用了zend_compile函数。我们是不是对它很熟悉呢？没错，它就是PHP脚本编译的入口。随后，通过调用这个函数，就可以对引入的外部脚本1.php进行词法分析和语法分析等编译操作了。
 - 现在思考一个问题，这个函数返回一个op_array，是引入的新的外部脚本1.php的op_array，那么原来的旧脚本2.php的op_array的状态和数据应该如何存储呢？
 - 答案是继续往zend_execute_data栈中添加。当include脚本执行完成之后，出栈即可。同递归的原理一样，递归也是借助栈，当你不断递归的时候，数据不断入栈，到最后的递归终止条件的时候，逐步出栈即可，所以递归是非常慢的，效率极低。
## 其他
### PHP脚本的执行流程
 - 我们之前讲过，PHP脚本的执行入口为main函数（我们代码层面无法看到，是虚拟机帮助我们加的）。从main函数进去之后，PHP脚本的执行总共有5大阶段：
 - CLI模式（command line interface，即命令行模式。如在命令行下执行脚本：php 1.php）：

>  - php_module_startup：模块初始化
>  - php_request_startup：请求初始化
>  - php_execute_script：执行脚本
>  - php_request_shutdown：请求关闭
>  - php_module_shutdown：模块关闭

 - CLI模式下，运行一次就会直接退出，并不常驻内存，接下来看一下我们使用的最多的FPM模式，它常驻内存。一次请求到来，PHP-FPM就要对其进行处理，所以在 php_request_startup、php_execute_script、php_request_shutdown三个阶段会进行死循环，让PHP-FPM常驻内存，才能不断地处理一个个到来的请求。但是这样会有一个问题，每一个请求到来的时候，都会重新进行词法解析、语法解析......效率是非常低的。为了解决这个问题，PHP中我们常说的opcache就要粉墨登场了。
 - opcache：把之前解析过的opcode缓存起来
### 初探nginx+php-fpm架构
 - 在LNMP架构下，前端的请求发来，先会通过nginx做代理，然后通过fastcgi协议，转发给上游的php-fpm，由php-fpm真正地处理请求。
 - 我们知道，nginx是多进程架构的反向代理web服务器，由一个master进程和多个worker进程组成：
 - **master进程**：管理所有worker进程（如worker进程的创建、销毁）
 - **worker进程**：负责处理客户端发来的请求
 - 当杀死master进程的时候，worker进程依然存在，可以为客户端提供服务
 - 当杀死worker进程的时候（且当前没有其他worker进程），master进程就会再创建worker进程，保证nginx服务正常运行
 - 下一篇文章我们就即将讲解fastcgi协议，逐步揭开nginx+php-fpm架构通信的神秘面纱