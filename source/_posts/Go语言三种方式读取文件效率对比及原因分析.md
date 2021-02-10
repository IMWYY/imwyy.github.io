---
title: Go语言三种方式读取文件效率对比及原因分析
date: 2018-10-07
tags: Performance
---

最近有遇到需要用go读取大文件的情况，顺路研究了一下go几种读取文件方式的效率。

## go几种常见的文件io方式

1. 使用os包内的open和read。

   ```go
   fi, err := os.Open(path) // 打开文件
   buf := make([]byte, 1024)
   n, err := fi.Read(buf)   // 读取内容
   ```

2. 使用buffered io

   ```go
   fi, err := os.Open(path)
   r := bufio.NewReader(fi)
   buf := make([]byte, 1024)
   n, err := r.Read(buf)
   ```

3. 使用ioutil包内的方法

   ```go
   fi, err := os.Open(path)
   fd, err := ioutil.ReadAll(fi)
   ```

## 现象（效率对比）

准备了待读取文件信息如下：

```shel
total 720912
-rw-r--r--  1 stephen  staff   2.3K Sep 15 11:59 io_demo.go
-rw-r--r--  1 stephen  staff   336M Sep 15 11:59 test.txt
```

同时io_demo.go文件中的代码如下：

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"io/ioutil"
	"os"
	"time"
)

func readRaw(path string) string {
	start := time.Now()
	fi, err := os.Open(path)
	if err != nil {
		panic(err)
	}
	defer fi.Close()
	defer func() {
		fi.Close()
		fmt.Printf("[readRaw] cost time %v \n", time.Now().Sub(start))
	}()

	var data []byte
	buf := make([]byte, 1024)
	for {
		n, err := fi.Read(buf)
		if err != nil && err != io.EOF {
			panic(err)
		}
		data = append(data, buf[:n]...)
		if 0 == n {
			break
		}
	}
	return string(data)
}

func readWithBufferIO(path string) string {
	start := time.Now()
	fi, err := os.Open(path)
	if err != nil {
		panic(err)
	}
	defer func() {
		fi.Close()
		fmt.Printf("[readWithBufferIO] cost time %v \n", time.Now().Sub(start))
	}()

	r := bufio.NewReader(fi)

	var data []byte

	buf := make([]byte, 1024)
	for {
		n, err := r.Read(buf)
		if err != nil && err != io.EOF {
			panic(err)
		}
		if 0 == n {
			break
		}
		data = append(data, buf[:n]...)
	}
	return string(data)
}

func readWithIOUtil(path string) string {
	start := time.Now()
	fi, err := os.Open(path)
	if err != nil {
		panic(err)
	}
	defer func() {
		fi.Close()
		fmt.Printf("[readWithIOUtil] cost time %v \n", time.Now().Sub(start))
	}()

	fd, err := ioutil.ReadAll(fi)
	return string(fd)
}

func main() {
	file := "test.txt"

	readRaw(file)
	readWithBufferIO(file)
	readWithIOUtil(file)
}
```

用如上代码读取已准备的文件，多次测试用时信息如下（进行了超过10次测试，仅取了两个结果来说明问题）：

```shell
[readRaw] cost time 1.490717874s
[readWithBufferIO] cost time 573.336617ms
[readWithIOUtil] cost time 379.678285ms
```

```shell
[readRaw] cost time 1.45133396s
[readWithBufferIO] cost time 541.944555ms
[readWithIOUtil] cost time 983.909509ms
```

可以看到，**毫无疑问使用os包readRaw读取的方式是最慢的，且相比其他两种方式要慢很多。但是readWithBufferIO和readWithIOUtil 两种方式速度的快慢就很难分伯仲了**。

## 透过现象看本质

既然得到了这个结论，那么我们来看看为什么会这样。

#### 1. 为什么bufferIO会比普通read快？

看bufio源码

```go
// NewReader returns a new Reader whose buffer has the default size.
func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)
}
```

再看NewReaderSize方法

```go
// NewReaderSize returns a new Reader whose buffer has at least the specified
// size. If the argument io.Reader is already a Reader with large enough
// size, it returns the underlying Reader.
func NewReaderSize(rd io.Reader, size int) *Reader {
	// Is it already a Reader?
	b, ok := rd.(*Reader)
	if ok && len(b.buf) >= size {
		return b
	}
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	r := new(Reader)
	r.reset(make([]byte, size), rd)
	return r
}
```

bufferio默认创建一个大小为4096 byte的缓冲区，它的 read 方法执行一次IO系统调用读取4096byte（4K）大小到缓冲区，此后`r.Read(buf)`都会从缓冲区中读。**而普通io每次读/写操作都会执行系统调用**，必然会比bufferIO慢很多，毕竟**每次系统调用都会从执行从用户态到内核态的切换**。

#### 2. 为什么bufferio和ioutil的效率难分伯仲？

来看`ioutil`源码

```go
// MinRead is the minimum slice size passed to a Read call by
// Buffer.ReadFrom. As long as the Buffer has at least MinRead bytes beyond
// what is required to hold the contents of r, ReadFrom will not grow the
// underlying buffer.
const MinRead = 512

// ReadAll reads from r until an error or EOF and returns the data it read.
// A successful call returns err == nil, not err == EOF. Because ReadAll is
// defined to read from src until EOF, it does not treat an EOF from Read
// as an error to be reported.
func ReadAll(r io.Reader) ([]byte, error) {
	return readAll(r, bytes.MinRead)
}

// readAll reads from r until an error or EOF and returns the data it read
// from the internal buffer allocated with a specified capacity.
func readAll(r io.Reader, capacity int64) (b []byte, err error) {
	var buf bytes.Buffer
	// If the buffer overflows, we will get bytes.ErrTooLarge.
	// Return that as an error. Any other panic remains.
	defer func() {
		e := recover()
		if e == nil {
			return
		}
		if panicErr, ok := e.(error); ok && panicErr == bytes.ErrTooLarge {
			err = panicErr
		} else {
			panic(e)
		}
	}()
	if int64(int(capacity)) == capacity {
		buf.Grow(int(capacity))
	}
	_, err = buf.ReadFrom(r)
	return buf.Bytes(), err
}
```

可以看到，`ioutil.ReadAll`最后实现的也是一个带缓冲的IO，**且大小在512byte以上，且使用的是bytes.Buffer，可以根据情况动态的增长。但是的Grow时重新分配buf也会带来一些开销**，所以两者相比就变成了一个权衡，没有绝对占优。

但是ioutil的好处就是方便，`ioutil.ReadAll`或者`ioutil.ReadFile`一行代码就搞定。