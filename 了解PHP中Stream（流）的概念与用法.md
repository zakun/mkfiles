## 了解PHP中Stream（流）的概念与用法

### 概述
> [参考链接](http://www.nowamagic.net/librarys/veda/detail/2587)

> Stream是PHP开发里最容易被忽视的函数系列（SPL系列，Stream系列，pack函数，封装协议）之一，但其是个很有用也很重要的函数。Stream可以翻译为“流”，在Java里，流是一个很重要的概念。
> 
> 流(stream)的概念源于UNIX中管道(pipe)的概念。在UNIX中，管道是一条不间断的字节流，用来实现程序或进程间的通信，或读写外围设备、外部文件等。根据流的方向又可以分为输入流和输出流，同时可以在其外围再套上其它流，比如缓冲流，这样就可以得到更多流处理方法。
> 
> PHP里的流和Java里的流实际上是同一个概念，只是简单了一点。由于PHP主要用于Web开发，所以“流”这块的概念被提到的较少。如果有Java基础，对于PHP里的流就更容易理解了。其实PHP里的许多高级特性，比如SPL，异常，过滤器等都参考了Java的实现，在理念和原理上同出一辙。

### example#1 查找固定条件的文件

比如下面是一段PHP SPL标准库的用法（遍历目录，查找固定条件的文件）：

	class RecursiveFileFilterIterator extends FilterIterator
	{
	    // 满足条件的扩展名
	    protected $ext = array('jpg','gif');

	    /**
	     * 提供 $path 并生成对应的目录迭代器
	     */
	    public function __construct($path)
	    {
		parent::__construct(new RecursiveIteratorIterator(new RecursiveDirectoryIterator($path)));
	    }

	    /**
	     * 检查文件扩展名是否满足条件
	     */
	    public function accept()
	    {
		$item = $this->getInnerIterator();
		if ($item->isFile() && in_array(pathinfo($item->getFilename(), PATHINFO_EXTENSION), $this->ext))
		{
		    return TRUE;
		}
	    }
	}


	// 实例化
	foreach (new RecursiveFileFilterIterator('D:/history') as $item)
	{
	    echo $item . PHP_EOL;
	}


我们可以通过几个例子先来了解stream系列函数的使用。

### example#2 socket来抓取数据

下面是一个使用socket来抓取数据的例子：

	$post_ =array (
	    'author' => 'Gonn',
	    'mail'=>'gonn@nowamagic.net',
	    'url'=>'http://www.nowamagic.net/',
	    'text'=>'使用socket来抓取数据的例子');

	$data=http_build_query($post_);
	$fp = fsockopen("baidu.com", 80, $errno, $errstr, 5);

	$out="POST http://baidu.com/news/1/comment HTTP/1.1\r\n";
	$out.="Host: example.org\r\n";
	$out.="User-Agent: Mozilla/5.0 (Windows; U; Windows NT 6.1; zh-CN; rv:1.9.2.13) Gecko/20101203 Firefox/3.6.13"."\r\n";
	$out.="Content-type: application/x-www-form-urlencoded\r\n";
	$out.="PHPSESSID=082b0cc33cc7e6df1f87502c456c3eb0\r\n";
	$out.="Content-Length: " . strlen($data) . "\r\n";
	$out.="Connection: close\r\n\r\n";
	$out.=$data."\r\n\r\n";

	fwrite($fp, $out);
	while (!feof($fp))
	{
	    echo fgets($fp, 1280);
	}

	fclose($fp);

我们也可以用stream_socket 实现，这很简单，只需要打开socket的代码换成下面的即可：

	$fp = stream_socket_client("tcp://nowamagic.net:80", $errno, $errstr, 3);

### example#3 file_get_contents

再来看一个stream的例子：

`file_get_contents`函数一般常用来读取文件内容，但这个函数也可以用来抓取远程url，起到和curl类似的作用。

	$opts = array (
	    'http'=>array(
	       'method' => 'POST',
	       'header'=> "Content-type: application/x-www-form-urlencoded\r\n" .
			  "Content-Length: " . strlen($data) . "\r\n",
	       'content' => $data)
	);

	$context = stream_context_create($opts);
	file_get_contents('http://baidu.comt/news/1/comment', false, $context);

注意第三个参数，`$context`，即HTTP流上下文，可以理解为套在`file_get_contents`函数上的一根管道。同理，我们还可以创建FTP流，socket流，并把其套在对应的函数在。


### example#4 过滤器流的作用

我们再来看个过滤器流的作用：

	$fp = fopen('c:/test.txt', 'w+');

	/* 把rot13过滤器作用在写入流上 */
	stream_filter_append($fp, "string.rot13", STREAM_FILTER_WRITE);

	/* 写入的数据经过rot13过滤器的处理*/
	fwrite($fp, "This is a test\n");
	rewind($fp);

	/* 读取写入的数据，独到的自然是被处理过的字符了 */
	fpassthru($fp);
	fclose($fp);

	// output：Guvf vf n grfg

在上面的例子中，如果我们把过滤器的类型设置为`STREAM_FILTER_ALL`，即同时作用在读写流上，那么读写的数据都将被`rot13`过滤器处理，我们读出的数据就和写入的原始数据是一致的。

你可能会奇怪`stream_filter_append`中的 `string.rot13`这个变量来的莫名其妙，这实际上是PHP内置的一个过滤器。

使用下面的方法即可打印出PHP内置的流：

	$streamlist = stream_get_filters();
	print_r($streamlist);

输出：

	Array
	(
	    [0] => convert.iconv.*
	    [1] => mcrypt.*
	    [2] => mdecrypt.*
	    [3] => string.rot13
	    [4] => string.toupper
	    [5] => string.tolower
	    [6] => string.strip_tags
	    [7] => convert.*
	    [8] => consumed
	    [9] => dechunk
	    [10] => zlib.*
	    [11] => bzip2.*
	)

自然而然，我们会想到定义自己的过滤器，这个也不难：

	class md5_filter extends php_user_filter
	{
	    function filter($in, $out, &$consumed, $closing)
	    {
		while ($bucket = stream_bucket_make_writeable($in))
		{
		    $bucket->data = md5($bucket->data);
		    $consumed += $bucket->datalen;
		    stream_bucket_append($out, $bucket);
		}

		//数据处理成功，可供其它管道读取
		return PSFS_PASS_ON;
	    }
	}

	stream_filter_register("string.md5", "md5_filter");

注意：过滤器名可以随意取。

之后就可以使用`string.md5`这个我们自定义的过滤器了。

这个过滤器的写法看起来很是有点摸不着头脑，事实上我们只需要看一下`php_user_filter`这个类的结构和内置方法即了解了。

过滤器流最适合做的就是文件格式转换了，包括压缩，编解码等，除了这些“偏门”的用法外，filter流更有用的一个地方在于调试和日志功能，比如说在socket开发中，注册一个过滤器流进行log记录。比如下面的例子：

	class md5_filter extends php_user_filter
	{
	    public function filter($in, $out, &$consumed, $closing)
	    {
		$data="";
		while ($bucket = stream_bucket_make_writeable($in))
		{
		    $bucket->data = md5($bucket->data);
		    $consumed += $bucket->datalen;
		    stream_bucket_append($out, $bucket);
		}

		call_user_func($this->params, $data);
		return PSFS_PASS_ON;
	    }
	}

	$callback = function($data)
	{
	    file_put_contents("c:\log.txt",date("Y-m-d H:i")."\r\n");
	};

这个过滤器不仅可以对输入流进行处理，还能回调一个函数来进行日志记录。

可以这么使用：

	stream_filter_prepend($fp, "string.md5", STREAM_FILTER_WRITE,$callback);

PHP中的stream流系列函数中还有一个很重要的流，就是包装类流 `streamWrapper`。使用包装流可以使得不同类型的协议使用相同的接口操纵数据。