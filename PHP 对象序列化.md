## 1. 对象序列化

所有php里面的值都可以使用函数`serialize()`来返回一个包含字节流的字符串来表示。`unserialize()`函数能够重新把字符串变回php原来的值。 序列化一个对象将会保存对象的所有变量，但是不会保存对象的方法，只会保存类的名字。

为了能够`unserialize()`一个对象，这个对象的类必须已经定义过。如果序列化类A的一个对象，将会返回一个跟类A相关，而且包含了对象所有变量值的字符串。 如果要想在另外一个文件中反序列化一个对象，这个对象的类必须在反序列化之前定义，可以通过包含一个定义该类的文件或使用函数`spl_autoload_register()`来实现。

	<?php
	// classa.inc:
	  
	  class A {
	      public $one = 1;
	    
	      public function show_one() {
		  echo $this->one;
	      }
	  }
	  
	// page1.php:

	  include("classa.inc");
	  
	  $a = new A;
	  $s = serialize($a);
	  // 把变量$s保存起来以便文件page2.php能够读到
	  file_put_contents('store', $s);

	// page2.php:
	  
	  // 要正确反序列化，必须包含下面一个文件
	  include("classa.inc");

	  $s = file_get_contents('store');
	  $a = unserialize($s);

	  // 现在可以使用对象$a里面的函数 show_one()
	  $a->show_one();
	?>

可以在对象上使用 `__sleep()` 和 `__wakeup()` 方法对序列化/反序列化事件挂载钩子。 使用 __sleep() 也能够让你仅序列化对象的某些属性

## 2.魔术方法

### 2.1 `__sleep()` 和 `__wakeup()`

	public __sleep(): array

	public __wakeup(): void

`serialize()` 函数会检查类中是否存在一个魔术方法 `__sleep()`。如果存在，该方法会先被调用，然后才执行序列化操作。此功能可以用于清理对象，并返回一个包含对象中所有应被序列化的变量名称的数组。如果该方法未返回任何内容，则 null 被序列化，并产生一个 E_NOTICE 级别的错误。

	注意:

	__sleep() 不能返回父类的私有成员的名字。这样做会产生一个 E_NOTICE 级别的错误。可以用 Serializable 接口来替代

`__sleep()` 方法常用于提交未提交的数据，或类似的清理操作。同时，如果有一些很大的对象，但不需要全部保存，这个功能就很好用。

与之相反，`unserialize()` 会检查是否存在一个 `__wakeup()` 方法。如果存在，则会先调用 __wakeup 方法，预先准备对象需要的资源。

`__wakeup()` 经常用在反序列化操作中，例如重新建立数据库连接，或执行其它初始化操作。

**示例 #1 Sleep 和 wakeup**

	<?php
	class Connection 
	{
	    protected $link;
	    private $server, $username, $password, $db;
	    
	    public function __construct($server, $username, $password, $db)
	    {
		$this->server = $server;
		$this->username = $username;
		$this->password = $password;
		$this->db = $db;
		$this->connect();
	    }
	    
	    private function connect()
	    {
		$this->link = mysql_connect($this->server, $this->username, $this->password);
		mysql_select_db($this->db, $this->link);
	    }
	    
	    public function __sleep()
	    {
		return array('server', 'username', 'password', 'db');
	    }
	    
	    public function __wakeup()
	    {
		$this->connect();
	    }
	}
	?>


2.2 `__serialize()` 和 `__unserialize()`

	注意:

	此特性自 PHP 7.4.0 起可用。


	public __serialize(): array

	public __unserialize(array $data): void

`serialize()` 函数会检查类中是否存在一个魔术方法 `__serialize()`。如果存在，该方法将在任何序列化之前优先执行。它必须以一个代表对象序列化形式的 `键/值` 成对的关联数组形式来返回，如果没有返回数组，将会抛出一个 TypeError 错误

	注意:
	如果类中同时定义了 __serialize() 和 __sleep() 两个魔术方法，则只有 __serialize() 方法会被调用。 __sleep() 方法会被忽略掉。<br>如果对象实现了 Serializable 接口，接口的 serialize() 方法会被忽略，做为代替类中的 __serialize() 方法会被调用。
	如果类中同时定义了 __unserialize() 和 __wakeup() 两个魔术方法，则只有 __unserialize() 方法会生效，__wakeup() 方法会被忽略。

**示例 #2 序列化和反序列化**
	<?php
	class Connection
	{
	    protected $link;
	    private $dsn, $username, $password;

	    public function __construct($dsn, $username, $password)
	    {
		$this->dsn = $dsn;
		$this->username = $username;
		$this->password = $password;
		$this->connect();
	    }

	    private function connect()
	    {
		$this->link = new PDO($this->dsn, $this->username, $this->password);
	    }

	    public function __serialize(): array
	    {
		return [
		  'dsn' => $this->dsn,
		  'user' => $this->username,
		  'pass' => $this->password,
		];
	    }

	    public function __unserialize(array $data): void
	    {
		$this->dsn = $data['dsn'];
		$this->username = $data['user'];
		$this->password = $data['pass'];

		$this->connect();
	    }
	}?>

## 3. 序列化接口

### 3.1 简介

> [序列化接口文档 - 详情](https://www.php.net/manual/zh/class.serializable.php)
自定义序列化的接口。

实现此接口的类将不再支持 `__sleep()` 和 `__wakeup()`。不论何时，只要有实例需要被序列化，serialize 方法都将被调用。它将不会调用 `__destruct()` 或有其他影响，除非程序化地调用此方法。当数据被反序列化时，类将被感知并且调用合适的 unserialize() 方法而不是调用 `__construct()`。如果需要执行标准的构造器，你应该在这个方法中进行处理。

### 3.2 接口摘要

	class Serializable {
		/* 方法 */
		abstract public serialize(): string
		abstract public unserialize(string $serialized): mixed
	}

**示例 #1 Basic usage**

	<?php
	class obj implements Serializable {
	    private $data;
	    public function __construct() {
		$this->data = "My private data";
	    }
	    public function serialize() {
		return serialize($this->data);
	    }
	    public function unserialize($data) {
		$this->data = unserialize($data);
	    }
	    public function getData() {
		return $this->data;
	    }
	}

	$obj = new obj;
	$ser = serialize($obj);

	$newobj = unserialize($ser);

	var_dump($newobj->getData());
	?>
以上例程的输出类似于：

	string(15) "My private data"

**示例 #2 Here's an example how to un-, serialize more than one property:**

	class Example implements \Serializable
	{
	    protected $property1;
	    protected $property2;
	    protected $property3;

	    public function __construct($property1, $property2, $property3)
	    {
		$this->property1 = $property1;
		$this->property2 = $property2;
		$this->property3 = $property3;
	    }

	    public function serialize()
	    {
		return serialize([
		    $this->property1,
		    $this->property2,
		    $this->property3,
		]);
	    }

	    public function unserialize($data)
	    {
		list(
		    $this->property1,
		    $this->property2,
		    $this->property3
		) = unserialize($data);
	    }

	}

**示例 #3 Serializing child and parent classes:**

	<?php
	class MyClass implements Serializable {
	    private $data;
	   
	    public function __construct($data) {
		$this->data = $data;
	    }
	   
	    public function getData() {
		return $this->data;
	    }
	   
	    public function serialize() {
		echo "Serializing MyClass...\n";
		return serialize($this->data);
	    }
	   
	    public function unserialize($data) {
		echo "Unserializing MyClass...\n";
		$this->data = unserialize($data);
	    }
	}

	class MyChildClass extends MyClass {
	    private $id;
	    private $name;
	   
	    public function __construct($id, $name, $data) {
		parent::__construct($data);
		$this->id = $id;
		$this->name = $name;
	    }
	   
	    public function serialize() {
		echo "Serializing MyChildClass...\n";
		return serialize(
		    array(
			'id' => $this->id,
			'name' => $this->name,
			'parentData' => parent::serialize()
		    )
		);
	    }
	   
	    public function unserialize($data) {
		echo "Unserializing MyChildClass...\n";
		$data = unserialize($data);
	       
		$this->id = $data['id'];
		$this->name = $data['name'];
		parent::unserialize($data['parentData']);
	    }
	   
	    public function getId() {
		return $this->id;
	    }
	   
	    public function getName() {
		return $this->name;
	    }
	}

	$obj = new MyChildClass(15, 'My class name', 'My data');

	$serial = serialize($obj);
	$newObject = unserialize($serial);

	echo $newObject->getId() . PHP_EOL;
	echo $newObject->getName() . PHP_EOL;
	echo $newObject->getData() . PHP_EOL;

	?>

This will output:

	Serializing MyChildClass...
	Serializing MyClass...
	Unserializing MyChildClass...
	Unserializing MyClass...
	15
	My class name
	My data
