## Nginx URL重写模块

### 摘要
> 这个模块允许使用正则表达式重写URI（需PCRE库），并且可以根据相关变量重定向和选择不同的配置。
> 如果这个指令在server字段中指定，那么将在被请求的location确定之前执行，如果在指令执行后所选择的location中有其他的重写规则，那么它们也被执行。如果在location中执行这个指令产生了新的URI，那么location又一次确定了新的URI。
> 这样的循环可以最多执行10次，超过以后nginx将返回500错误。

##指令
### break

	语法：break
	默认值：none
	使用字段：server, location, if

完成当前设置的规则，停止执行其他的重写指令。

示例：

	if ($slow) {
	  limit_rate  10k;
	  break;
	}

### if

	语法：if (condition) { ... }
	默认值：none
	使用字段：server, location

判断一个条件，如果条件成立，则后面的大括号内的语句将执行，相关配置从上级继承。

可以在判断语句中指定下列值：

- 一个变量的名称；不成立的值为：空字符传""或者一些用“0”开始的字符串。
- 一个使用=或者!=运算符的比较语句。
- 使用符号~*和~模式匹配的正则表达式：
- ~为区分大小写的匹配。
- ~*不区分大小写的匹配（firefox匹配FireFox）。
- !~和!~*意为“不匹配的”。
- 使用-f和!-f检查一个文件是否存在。
- 使用-d和!-d检查一个目录是否存在。
- 使用-e和!-e检查一个文件，目录或者软链接是否存在。
- 使用-x和!-x检查一个文件是否为可执行文件。

正则表达式的一部分可以用圆括号，方便之后按照顺序用`$1-$9`来引用。

示例配置：

	if ($http_user_agent ~ MSIE) {
	  rewrite  ^(.*)$  /msie/$1  break;
	}
	 
	if ($http_cookie ~* "id=([^;] +)(?:;|$)" ) {
	  set  $id  $1;
	}
	 
	if ($request_method = POST ) {
	  return 405;
	}
	 
	if (!-f $request_filename) {
	  break;
	  proxy_pass  http://127.0.0.1;
	}
	 
	if ($slow) {
	  limit_rate  10k;
	}
	 
	if ($invalid_referer) {
	  return   403;
	}
	 
	if ($args ~ post=140){
	  rewrite ^ http://example.com/ permanent;
	}

内置变量`$invalid_referer`用指令`valid_referers`指定。
### return

	语法：return code
	默认值：none
	使用字段：server, location, if

这个指令结束执行配置语句并为客户端返回状态代码，可以使用下列的值：204，400，402-406，408，410, 411, 413, 416与500-504。此外，非标准代码444将关闭连接并且不发送任何的头部。

### rewrite

	语法：rewrite regex replacement flag
	默认值：none
	使用字段：server, location, if

按照相关的正则表达式与字符串修改URI，指令按照在配置文件中出现的顺序执行。

`注意`重写规则只匹配相对路径而不是绝对的URL，如果想匹配主机名，可以加一个if判断，如：

	if ($host ~* www\.(.*)) {
	  set $host_without_www $1;
	  rewrite ^(.*)$ http://$host_without_www$1 permanent; # $1为'/foo'，而不是'www.mydomain.com/foo'
	}

可以在重写指令后面添加`标记`。
如果替换的字符串以`http://` 开头，请求将被重定向，并且不再执行多余的`rewrite`指令。

标记可以是以下的值：
- last - 完成重写指令，之后搜索相应的URI或location。
- break - 完成重写指令。
- redirect - 返回302临时重定向，如果替换字段用http://开头则被使用。
- permanent - 返回301永久重定向。

`注意`如果一个重定向是相对的（没有主机名部分），nginx将在重定向的过程中使用匹配`server_name`指令的“Host”头或者server_name指令指定的第一个名称，如果头不匹配或不存在，如果没有设置`server_name`，将使用本地主机名，如果你总是想让nginx使用“Host”头，可以在`server_name`使用“*”通配符（查看http核心模块中的server_name）。例如：

	rewrite  ^(/download/.*)/media/(.*)\..*$  $1/mp3/$2.mp3  last;
	rewrite  ^(/download/.*)/audio/(.*)\..*$  $1/mp3/$2.ra   last;
	return   403;

但是如果我们将其放入一个名为**/download/**的location中，则需要将`last`标记改为`break`，否则nginx将执行10次循环并返回500错误。

	location /download/ {
	  rewrite  ^(/download/.*)/media/(.*)\..*$  $1/mp3/$2.mp3  break;
	  rewrite  ^(/download/.*)/audio/(.*)\..*$  $1/mp3/$2.ra   break;
	  return   403;
	}
如果替换字段中包含参数，那么其余的请求参数将附加到后面，为了防止附加，可以在最后一个字符后面跟一个`问号`：

	rewrite  ^/users/(.*)$  /show?user=$1?  last;
`注意`：大括号（{和}），可以同时用在正则表达式和配置块中，为了防止冲突，**正则表达式使用大括号需要用双引号（或者单引号）**。例如要重写以下的URL：

	/photos/123456 
为:

	/path/to/photos/12/1234/123456.png 
则使用以下正则表达式（注意引号）：

	rewrite  "/photos/([0-9] {2})([0-9] {2})([0-9] {2})" /path/to/photos/$1/$1$2/$1$2$3.png;

同样，重写只对路径进行操作，而不是参数，如果要重写一个带参数的URL，可以使用以下代替：

	if ($args ^~ post=100){
	  rewrite ^ http://example.com/new-address.html? permanent;
	}

`注意`$args变量不会被编译，与location过程中的URI不同（参考http核心模块中的location）。
### set

	语法：set variable value
	默认值：none
	使用字段：server, location, if
指令设置一个变量并为其赋值，其值可以是文本，变量和它们的组合。

你可以使用`se`t定义一个新的变量，但是不能使用`set`设置$http_xxx头部变量的值。具体可以查看[这个例子](http://www.uk.superiorpapers.com/)

### uninitialized_variable_warn

	语法：uninitialized_variable_warn on|off
	默认值：uninitialized_variable_warn on
	使用字段：http, server, location, if
开启或关闭在未初始化变量中记录警告日志。
事实上，`rewrite`指令在配置文件加载时已经编译到内部代码中，在解释器产生请求时使用。
这个解释器是一个简单的堆栈虚拟机，如下列指令：

	location /download/ {
	  if ($forbidden) {
	    return   403;
	  }
	  if ($slow) {
	    limit_rate  10k;
	  }
	  rewrite  ^/(download/.*)/media/(.*)\..*$  /$1/mp3/$2.mp3  break;
将被编译成以下顺序：

	  variable $forbidden
	  checking to zero
	  recovery 403
	  completion of entire code
	  variable $slow
	  checking to zero
	  checkings of regular expression
	  copying "/"
	  copying $1
	  copying "/mp3/"
	  copying $2
	  copying "..mpe"
	  completion of regular expression
	  completion of entire sequence
`注意`并没有关于limit_rate的代码，因为它没有提及ngx_http_rewrite_module模块，“if”块可以类似"location"指令在配置文件的相同部分同时存在。
如果$slow为真，对应的if块将生效，在这个配置中limit_rate的值为10k。

指令：

	rewrite  ^/(download/.*)/media/(.*)\..*$  /$1/mp3/$2.mp3  break;
如果我们将第一个斜杠括入圆括号，则可以减少执行顺序：

	rewrite  ^(/download/.*)/media/(.*)\..*$  $1/mp3/$2.mp3  break;
之后的顺序类似如下：

	  checking regular expression
	  copying $1
	  copying "/mp3/"
	  copying $2
	  copying "..mpe"
	  completion of regular expression
	  completion of entire code

> 参考链接: ***http://shouce.jb51.net/nginx/StandardHTTPModules/Rewrite.html***