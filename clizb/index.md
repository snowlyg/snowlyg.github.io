# 不同语言之间数据压缩/解压实现


<!--more-->
文章源码地址：[https://www.github.com/snowlyg/learns](https://www.github.com/snowlyg/learns)

# 不同语言之间数据压缩/解压实现

* 正常的时候，使用一种编程语言实现数据的压缩、解压都十分简单。

#### golang 有如下 5 个基础包
- [compress/gzip](https://golang.org/pkg/compress/gzip)
- [compress/bzip2](https://golang.org/pkg/compress/bzip2)
- [compress/flate](https://golang.org/pkg/compress/flate)
- [compress/lzw](https://golang.org/pkg/compress/lzw)
- [compress/zlib](https://golang.org/pkg/compress/zlib)
- 
#### golang 第三方包使用 c++ 的 zlib 包，本次主要使用这个。
- [chindeo/czlib](https://www.github.com/chindeo/czlib)

#### php 有如下 6 个压缩、解压格式
- [bzip2](https://www.php.net/manual/en/book.bzip2.php)
- [lzf](https://www.php.net/manual/en/book.lzf.php)
- [phar](https://www.php.net/manual/en/book.phar.php)
- [rar](https://www.php.net/manual/en/book.rar.php)
- [zip](https://www.php.net/manual/en/book.zip.php)
- [zlib](https://www.php.net/manual/en/book.zlib.php)

#### nodejs 主要使用 zlib 压缩、解压格式
- [zlib](http://nodejs.cn/api/zlib.html)

三种语言都有 `zlib` 方式的压缩、解压方式，这样选择 `zlib` 方式来做不同编程语言间的压缩，解压方式比较合适。

#### data.json 文件
```json
{
    "fdafddsf":"e324r324e324e",
    "fd111afddsf":"e324r324e324e"
}
```

#### golang 实现 `zlib` 
```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"os"
	"path"

	"github.com/chindeo/czlib"
)

func main() {
	// 提取文件内容
	f, err := os.Open("./data.json")
	if err != nil {
		fmt.Printf("os open %v\n", err)
		return
	}

	// 压缩内容
	var b bytes.Buffer
	w, err := czlib.NewWriterLevel(&b, -1)
	if err != nil {
		fmt.Printf("flate new writer %v\n", err)
		return
	}
	defer w.Close()
	_, err = io.Copy(w, f)
	if err != nil {
		fmt.Printf("zlib new reader %v", err)
		return
	}
	w.Flush()

	// 将压缩后内容写入文件
	_, err = writeBytes("./data.zip", b.Bytes())
	if err != nil {
		fmt.Printf("WriteBytes %v\n", err)
		return
	}

	// 解压内容
	r, err := czlib.NewReader(&b)
	if err != nil {
		fmt.Printf("NewReader %v\n", err)
		return
	}
	defer r.Close()
	_, err = io.Copy(os.Stdout, r)
	if err != nil {
		fmt.Printf("io copy %v\n", err)
		return
	}
}

func writeBytes(filePath string, b []byte) (int, error) {
	os.MkdirAll(path.Dir(filePath), os.ModePerm)
	fw, err := os.Create(filePath)
	if err != nil {
		return 0, err
	}
	defer fw.Close()
	return fw.Write(b)
}

```
- 执行 `go run main.go` 得到 `data.zip` 文件


#### nodejs 实现 `zlib` 
- compress.js
```js
const { createDeflate } = require('zlib');
const { pipeline } = require('stream');
const {
  createReadStream,
  createWriteStream
} = require('fs');

const gzip = createDeflate();
const source = createReadStream('data.json');
const destination = createWriteStream('data.zip');

pipeline(source, gzip, destination, (err) => {
  if (err) {
    console.error('发生错误:', err);
    process.exitCode = 1;
  }
});
```

- decompress.js
```js
const { createInflate } = require('zlib');
const { pipeline } = require('stream');
const {
  createReadStream,
  createWriteStream
} = require('fs');


const gunzip = createInflate();
const sourceUnzip = createReadStream('data.zip');
const destinationUnzip =  createWriteStream('data.txt');

pipeline(sourceUnzip, gunzip, destinationUnzip, (err) => {
  if (err) {
    console.error('发生错误:', err);
    process.exitCode = 1;
  }
});
```
- 执行 `node compress.js` 得到 `data.zip` 文件
- 执行 `node decompress.js` 解压 `data.zip`  文件


#### php 实现 `zlib` 
```php
<?php
// Perform GZIP compression:
$compressed = file_get_contents("data.json");
$deflateContext = deflate_init(ZLIB_ENCODING_DEFLATE);
$compressed = deflate_add($deflateContext, $compressed, ZLIB_NO_FLUSH);
$compressed .= deflate_add($deflateContext, NULL, ZLIB_FINISH);

file_put_contents("data.zip",$compressed);

// Perform GZIP decompression:
$inflateContext = inflate_init(ZLIB_ENCODING_DEFLATE);
$uncompressed = inflate_add($inflateContext, $compressed, ZLIB_NO_FLUSH);
$uncompressed .= inflate_add($inflateContext, NULL, ZLIB_FINISH);
echo $uncompressed;
?>
?>
```
- 执行 `php index.php` 得到 `data.zip` 文件

目前已经实现了三种语言各自的压缩，解压数据。
因为实现比较简单就没有贴语言间交叉压缩，解压数据的代码了。具体源码参考在：[learns/zlib](https://github.com/snowlyg/learns/tree/master/zlib)
·
#### PHP 解压 GO 压缩文件
- decompress_with_go_compress .php 
```php
<?php
// Perform GZIP compression:
$compressed = file_get_contents("../go/data.zip");
$inflateContext = inflate_init(ZLIB_ENCODING_DEFLATE);
$uncompressed = inflate_add($inflateContext, $compressed, ZLIB_NO_FLUSH);
$uncompressed .= inflate_add($inflateContext, NULL, ZLIB_FINISH);
file_put_contents("go_data.json",$uncompressed);
echo $uncompressed;
?>
```

#### PHP 解压 NODEJS 压缩文件
- decompress_with_nodejs_compress .php 
```php
<?php
// Perform GZIP compression:
$compressed = file_get_contents("../nodejs/data.zip");
$inflateContext = inflate_init(ZLIB_ENCODING_DEFLATE);
$uncompressed = inflate_add($inflateContext, $compressed, ZLIB_NO_FLUSH);
$uncompressed .= inflate_add($inflateContext, NULL, ZLIB_FINISH);
file_put_contents("go_data.json",$uncompressed);
echo $uncompressed;
?>
```

#### NODEJS 解压 PHP 压缩文件
- decompress_with_php_compress.js 
```js
const { createInflate } = require('zlib');
const { pipeline } = require('stream');
const {
  createReadStream,
  createWriteStream
} = require('fs');


const gunzip = createInflate();
const sourceUnzip = createReadStream('../php/data.zip');
const destinationUnzip =  createWriteStream('php_data.txt');

pipeline(sourceUnzip, gunzip, destinationUnzip, (err) => {
  if (err) {
    console.error('发生错误:', err);
    process.exitCode = 1;
  }
});
```

#### NODEJS 解压 GO 压缩文件
- decompress_with_go_compress.js 
```js
const { createInflate } = require('zlib');
const { pipeline } = require('stream');
const {
  createReadStream,
  createWriteStream
} = require('fs');


const gunzip = createInflate();
// console.log(gunzip)
const sourceUnzip = createReadStream('../go/data.zip');
const destinationUnzip =  createWriteStream('go_data.txt');
pipeline(sourceUnzip, gunzip, destinationUnzip, (err) => {
  if (err) {
    console.error('发生错误:', err);
    process.exitCode = 1;
  }
});
```

#### GO 解压 PHP 压缩文件
```go
package main

import (
	"fmt"
	"io"
	"os"

	"github.com/chindeo/czlib"
)

func main() {

	// 提取文件内容
	f, err := os.Open("../php/data.zip")
	if err != nil {
		fmt.Printf("os open %v\n", err)
		return
	}

	// 解压内容
	r, err := czlib.NewReader(f)
	if err != nil {
		fmt.Printf("NewReader %v\n", err)
		return
	}
	defer r.Close()
	_, err = io.Copy(os.Stdout, r)
	if err != nil {
		fmt.Printf("io copy %v\n", err)
		return
	}
}

```

#### GO 解压 NODEJS 压缩文件
```go
package main

import (
	"fmt"
	"io"
	"os"

	"github.com/chindeo/czlib"
)

func main() {

	// 提取文件内容
	f, err := os.Open("../nodejs/data.zip")
	if err != nil {
		fmt.Printf("os open %v\n", err)
		return
	}

	// 解压内容
	r, err := czlib.NewReader(f)
	if err != nil {
		fmt.Printf("NewReader %v\n", err)
		return
	}
	defer r.Close()
	_, err = io.Copy(os.Stdout, r)
	if err != nil {
		fmt.Printf("io copy %v\n", err)
		return
	}

}

```

这次 `go` 使用的一个第三方包 [chindeo/czlib](https://www.github.com/chindeo/czlib)，这个包参考了一个开源库修改而来，主要是调用 c++ 的 [zlib](http://www.zlib.net/zlib.html) 库。相对于官方的几个压缩库，这个库可以设置更多的参数，能够更方便的兼容其他语言的压缩解压方法。

**特别是在其他语言使用的压缩解压方法因为一些原因已经固定下来不太好修改的情况下，往往需要其他语言修改对应的参数去匹配相应的压缩设置**
