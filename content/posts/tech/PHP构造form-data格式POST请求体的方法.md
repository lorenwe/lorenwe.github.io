---
title: "PHP构造multipart/form-data格式POST请求体的方法"
date: 2024-09-25T14:47:27+08:00
lastmod: 2024-09-25T14:47:27+08:00
author: ["Sulv"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: ""
weight:
slug: ""
draft: false # 是否为草稿
comments: true # 本页面是否显示评论
reward: true # 打赏
mermaid: false #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---


### 引言

最近在尝试基于 PHP 做一个反向代理 HTTP 的程序，其中一个需求是将程序收到的HTTP请求还原回 [RFC2616](https://www.ietf.org/rfc/rfc2616.txt) 的原始格式。

在处理的过程中遇到的问题主要在请求体的处理上。利用PHP的封装协议机制，我们可以通过读取 [php://input](https://www.php.net/manual/zh/wrappers.php.php) 访问原始的POST信息。但这种方式有一个局限，对于 multipart/form-data 的请求来说，为了支持文件上传的操作，PHP会预先把请求体中的文件暂存到临时文件夹，并把参数解析到变量 $_POST 和 $_FILES 中， php://input 获取原始请求的功能也随之失效。

Stack Overflow 上的相关问题给出的 解决办法 是修改服务器配置，把发到 PHP 脚本的 Content-Type: multipart/form-data; boundary=xxxx 修改为其它格式，使其不经过PHP的 form-data 解析；或是把 php.ini 配置关于POST数据解析的 [enable_post_data_reading = Off](https://www.php.net/manual/en/ini.core.php#ini.enable-post-data-reading) 选项关闭。然而这两种方法并不非常具有普遍性，在某些PHP配置文件不可控的共享主机的环境下并不适用。

于是引出了本文讨论的话题 — 如何重新组装 multipart/form-data 格式的原始 POST 请求体。

### multipart/form-data 格式

在POST请求中，一般表单会通过 application/x-www-form-urlencoded 格式上传，但此格式的数据仅支持文本格式，不支持二进制文件的上传。为了支持表单 POST 文件上传，[RFC1867](https://www.ietf.org/rfc/rfc1867.txt) 定义了 multipart/form-data 的数据格式，实现了通过POST请求上传表单的内容以及二进制文件数据，关于数据的形态，参考 四种常见的 POST 提交数据方式 | JerryQu 的小站 。

[RFC1867](https://www.ietf.org/rfc/rfc1867.txt) 对于 multipart/form-data 的数据格式主要在MIME [RFC1521](https://www.ietf.org/rfc/rfc1521.txt) 7.2.1 小节定义的。另外，在MIME 标准 Media Types 部分 [RFC2046](https://www.ietf.org/rfc/rfc2046.txt) 的 5.1.1 节中，对于 multipart-body 的格式有一个较为清晰的 BNF 范式的语法定义，简短总结如下（来自 [Stack Overflow](https://stackoverflow.com/questions/27993445/is-this-a-well-formed-multipart-form-data-request)） ：

```
multipart-body := [preamble CRLF]
                  dash-boundary CRLF
                  body-part *encapsulation
                  close-delimiter
                  [CRLF epilogue]

dash-boundary := "--" boundary

body-part := MIME-part-headers [CRLF *OCTET]

encapsulation := delimiter
                 CRLF body-part

delimiter := CRLF dash-boundary

close-delimiter := delimiter "--"
```

### 还原 multipart/form-data 的代码

写代码前搜索前人的经验，在 SegmentFault 看到了一位[前辈的实现](https://segmentfault.com/q/1010000002604013)，参考前辈的代码，以及 RFC2046 的 BNF 语法定义，写了以下代码：

``` php
// 还原 rfc1867, rfc2046 格式的FormData
function getFormData() {
  // body-part array
  $body = array();

  // 普通参数
  foreach ($_POST as $key => $value) {
    $body_part = "Content-Disposition: form-data; name=\"$key\"\r\n";
    $body_part .= "\r\n$value";
    $body[] = $body_part;
  }

  // 上传文件处理
  foreach ($_FILES as $key => $value) {
    $body_part = "Content-Disposition: form-data; name=\"$key\"; filename=\"{$value['name']}\"\r\n";
    $body_part .= "Content-type: {$value['type']}\r\n";
    $body_part .= "\r\n".file_get_contents($value['tmp_name']);
    $body[] = $body_part;
  }

  // 提取boundary
  $boundary = substr($_SERVER['CONTENT_TYPE'], strpos($_SERVER['CONTENT_TYPE'], "=") + 1);
  // multipart-body
  $multipart_body = "--$boundary\r\n";
  // 拼接各个域
  $multipart_body .= implode("\r\n--$boundary\r\n", $body);
  // 最后一个不同的 boundary
  $multipart_body .= "\r\n--$boundary--";

  return $multipart_body;
}
```

### 数组类型参数的支持

以上代码在大多数情况下工作正常，但未考虑到请求参数的类型为数组的情况。

在PHP解释器源码的测试用例中，我们可以找到许多数组类型参数的测试，部分摘录如下：

```
a[]=1
a[]=1&a[]=1
a[]=1&a[0]=5
a[a]=1&a[b]=3
a[]=1&a[a]=1&a[b]=3
a[][]=1&a[][]=3&b[a][b][c]=1&b[a][b][d]=1
a=1&b=ZYX&c[][][][][][][][][][][][][][][][][][][][][][]=123&d=123&e[][]][]=3
```

```
Content-Type: multipart/form-data; boundary=---------------------------20896060251896012921717172737
-----------------------------20896060251896012921717172737
Content-Disposition: form-data; name="file[]"; filename="file1.txt"
Content-Type: text/plain-file1

1
-----------------------------20896060251896012921717172737
Content-Disposition: form-data; name="file[2]"; filename="file2.txt"
Content-Type: text/plain-file2

2
-----------------------------20896060251896012921717172737
Content-Disposition: form-data; name="file[]"; filename="file3.txt"
Content-Type: text/plain-file3

3
-----------------------------20896060251896012921717172737--
```

在PHP源码的 main/php_variables.c 中的 php_register_variable_ex 函数中，我们可以看到相关的处理：

``` c
/* 99-110行 */
/* ensure that we don't have spaces or dots in the variable name (not binary safe) */
for (p = var; *p; p++) {
    if (*p == ' ' || *p == '.') {
        *p='_';
    } else if (*p == '[') {
        is_array = 1;
        ip = p;
        *p = 0;
        break;
    }
}
var_len = p - var;

/* 229-235行 */
ip++;
if (*ip == '[') {
    is_array = 1;
    *ip = 0;
} else {
    goto plain_var;
}
```

可见，在还原POST数据的时候，我们还需要考虑到参数为数组的情况。

这里通过一个简单的 DFS 算法深度优先遍历数组，生成类似 a[0], a[1][1] 的字符串来实现：

``` php
<?php

$arr = [
  'key1' => [
    '_key1' => 23333,
    '_key2' => 66666,
  ],
  'key2' => "hahah",
  "test",
];

var_dump($arr);

function dfs(&$node, $prefix, &$result) {
  if (!is_array($node)) {
    $result[$prefix] = $node;
  } else {
    foreach ($node as $key => $value) {
      dfs($value, "{$prefix}[{$key}]", $result);
    }
  }
}

dfs($arr, "arr", $result);

foreach ($result as $key => $value) {
  echo "$key = $value\n";
}
```

运行结果：

```
array(3) {
  ["key1"]=>
  array(2) {
    ["_key1"]=>
    int(23333)
    ["_key2"]=>
    int(66666)
  }
  ["key2"]=>
  string(5) "hahah"
  [0]=>
  string(4) "test"
}
arr[key1][_key1] = 23333
arr[key1][_key2] = 66666
arr[key2] = hahah
arr[0] = test
```

至于 $_FILES 数组，这里有一个反直觉的情况，具体在文档中也有人提出： [PHP: POST method uploads - Manual](http://php.net/manual/en/features.file-upload.post-method.php#91479)

简单地说，当表单中文件域的key为数组形式时，拿到的 $_FILES 数组类似如下的格式：

```
array(1) {
  ["key"]=>
  array(5) {
    ["name"]=>
    array(2) {
      [0]=>
      string(8) "test.txt"
      [1]=>
      array(1) {
        [0]=>
        string(8) "test.txt"
      }
    }
    ["type"]=>
    array(2) {
      [0]=>
      string(10) "text/plain"
      [1]=>
      array(1) {
        [0]=>
        string(10) "text/plain"
      }
    }
    ["tmp_name"]=>
    array(2) {
      [0]=>
      string(14) "/tmp/phpKHCoSt"
      [1]=>
      array(1) {
        [0]=>
        string(14) "/tmp/phpSgtRHe"
      }
    }
    ["error"]=>
    array(2) {
      [0]=>
      int(0)
      [1]=>
      array(1) {
        [0]=>
        int(0)
      }
    }
    ["size"]=>
    array(2) {
      [0]=>
      int(8)
      [1]=>
      array(1) {
        [0]=>
        int(8)
      }
    }
  }
}
```

假设我的目标是 key[1][0] 的 name 属性，在PHP中我们需要通过 $_FILES["key"]["name"][1][0] 来访问，而在 $_FILES["key"]["name"] 中，后面的索引的层级并不确定的，我们也不能简单地指定 [1][0] 来访问 $_FILES["key"]["name"][1][0]。所以这里得有一些 hack 来优化一下这个过程，这里我实现了一个 query_multidimensional_array 函数，具体看最终的代码。

### getFormData() 代码实现

以下是整个函数的完整实现：

``` php
// 还原 rfc1867, rfc2046 格式的FormData
function getFormData() {
  // body-part array
  $body = array();

  // 普通参数
  foreach ($_POST as $key => $value) {
    if (!is_array($value)) {
      $body_part = "Content-Disposition: form-data; name=\"$key\"\r\n";
      $body_part .= "\r\n$value";
      $body[] = $body_part;
    } else {
      // 数组的情况处理 如 param1[]=xxxx
      $result = array();
      convert_array_key($value, $key, $result);
      foreach ($result as $k => $v) {
        $body_part = "Content-Disposition: form-data; name=\"$k\"\r\n";
        $body_part .= "\r\n$v";
        $body[] = $body_part;
      }
    }
  }

  // 上传文件处理
  foreach ($_FILES as $key => $value) {
    if (!is_array($value['type'])) {
      $body_part = "Content-Disposition: form-data; name=\"$key\"; filename=\"{$value['name']}\"\r\n";
      $body_part .= "Content-type: {$value['type']}\r\n";
      $body_part .= "\r\n".file_get_contents($value['tmp_name']);
      $body[] = $body_part;
    } else {
      // 文件key是数组的情况 如 file1[]=xxxx
      $result = array();
      convert_array_key($value['type'], "", $result);
      foreach ($result as $k => $v) {
        $filename = query_multidimensional_array($value['name'], $k);
        $type = query_multidimensional_array($value['type'], $k);
        $tmp_name = query_multidimensional_array($value['tmp_name'], $k);
        $body_part = "Content-Disposition: form-data; name=\"{$key}{$k}\"; filename=\"{$filename}\"\r\n";
        $body_part .= "Content-type: {$type}\r\n";
        $body_part .= "\r\n".file_get_contents($tmp_name);
        $body[] = $body_part;
      }
    }
  }

  // 提取boundary
  $boundary = substr($_SERVER['CONTENT_TYPE'], strpos($_SERVER['CONTENT_TYPE'], "=") + 1);
  // multipart-body
  $multipart_body = "--$boundary\r\n";
  // 拼接各个域
  $multipart_body .= implode("\r\n--$boundary\r\n", $body);
  // 最后一个不同的 boundary
  $multipart_body .= "\r\n--$boundary--";

  return $multipart_body;
}

// 直接访问多维数组元素
// query: [0][0] -> $array[0][0]
function query_multidimensional_array(&$array, $query) {
  $query = explode('][', substr($query, 1, -1));
  $temp = $array;
  foreach ($query as $key) {
    $temp = $temp[$key];
  }
  return $temp;
}

// DFS将数组变为一维形式
function convert_array_key(&$node, $prefix, &$result) {
  if (!is_array($node)) {
    $result[$prefix] = $node;
  } else {
    foreach ($node as $key => $value) {
      convert_array_key($value, "{$prefix}[{$key}]", $result);
    }
  }
}
```

至此，在PHP脚本中，只需调用 getFormData() ，即可获得 multipart/form-data 请求的原始数据，通过以下代码可以实现一键获取请求原始POST Body。

需要注意的是，若数组类型参数是 a[] 这种形式，经过本函数还原后会补充具体的下标，比如说这里的 a[] 会被处理成 a[0] ，a[][] 则为 a[0][0]。从而导致了 POST Body 长度发生变化，若结果需要用于发包等操作，我们需要重新计算 Content-Length ，避免请求出现问题。

``` php
if (@$_SERVER['CONTENT_TYPE'] && strpos($_SERVER['CONTENT_TYPE'], "multipart/form-data") !== false) {
  $body = getFormData();
  $content_length = strlen($body);
} else {
  $body = file_get_contents('php://input');
}
```
### PHP中以multipart/form-data上传文件流

上传类:

```php
class UploadPart
{   
    protected static $url;
    protected static $delimiter;
    protected static $instance;

    public function __construct() {
        static::$url = '上传地址';
        static::$delimiter = uniqid(); // 生成
    }

    public function putPart($param) {
        $post_data = static::buildData($param);
        $curl = curl_init(static::$url);
        curl_setopt($curl, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($curl, CURLOPT_POST, true);
        curl_setopt($curl, CURLOPT_POSTFIELDS, $post_data);
        curl_setopt($curl, CURLOPT_HTTPHEADER, [
            "Content-Type: multipart/form-data; boundary=" . static::$delimiter,
            "Content-Length: " . strlen($post_data)
        ]);
        $response = curl_exec($curl);
        curl_close($curl);
        $info = json_decode($response, true);
        if (!is_array($info['Msg']) && $info['Msg'] == $param['filesize']) {
            $param['offset'] = $param['filesize'];
            $param['upload'] = '';
            return $this->putPart($param);
        }
        return $response;
    }

    private static function buildData($param){
        $data = '';
        $eol = "\r\n";
        $upload = $param['upload'];
        unset($param['upload']);

        foreach ($param as $name => $content) {
            $data .= "--" . static::$delimiter . "\r\n"
                . 'Content-Disposition: form-data; name="' . $name . "\"\r\n\r\n"
                . $content . "\r\n";
        }
        // 拼接文件流
        $data .= "--" . static::$delimiter . $eol
            . 'Content-Disposition: form-data; name="upload"; filename="' . $param['filename'] . '"' . "\r\n"
            . 'Content-Type:application/octet-stream'."\r\n\r\n";

        $data .= $upload . "\r\n";
        $data .= "--" . static::$delimiter . "--\r\n";
        return $data;
    }
    
    public static function getInstance() {
        if(!static::$instance){
            static::$instance = new static();
        }
       return static::$instance;
    }

}
```

使用方法

``` php
$fields = array(
    'type' => 'video',
    'filename' => '1407.png',
    'filesize' => 58701,
    'offset' => 0,
    'filetype' => '.acc',
    'originName' => '1407.png',
    'upload'=>file_get_contents('0407.png')
);
$part = UploadPart::getInstance()->putPart($fields);
```

### 参考

- [RFC1521](https://www.ietf.org/rfc/rfc1521.txt)
- [RFC1867](https://www.ietf.org/rfc/rfc1867.txt)
- [RFC2046](https://www.ietf.org/rfc/rfc2046.txt)
- [PHP: POST 方法上传 - Manual](https://www.php.net/manual/zh/features.file-upload.post-method.php)
- [PHP: 上传多个文件 - Manual](https://www.php.net/manual/zh/features.file-upload.multiple.php)
- [PHP文件上传源码分析(RFC1867) | 风雪之隅](https://www.laruence.com/2009/09/26/1103.html)
- [深入理解PHP原理之文件上传 | 风雪之隅](https://www.laruence.com/2008/11/07/586.html)
- [四种常见的 POST 提交数据方式 | JerryQu 的小站](https://imququ.com/post/four-ways-to-post-data-in-http.html)
- [php - Get raw post data - Stack Overflow](https://stackoverflow.com/questions/1361673/get-raw-post-data)
- [http - Is this a well formed multipart/form-data request? - Stack Overflow](https://stackoverflow.com/questions/27993445/is-this-a-well-formed-multipart-form-data-request)
- [http - 请问哪份 RFC 文档定义了 multipart/form-data 的格式？ - SegmentFault 思否](https://segmentfault.com/q/1010000002604013)


