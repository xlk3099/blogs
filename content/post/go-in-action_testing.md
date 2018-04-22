---
title: "go实战读书笔记（二十）: 单元测试"
date: 2018-04-22T15:42:11+08:00
draft: false
author: "xlk3099"
categories: ["golang"]
tags: ["Go In Action"]
---


go的单测.

go的测试文件都已`*_test.go`结尾. 不管是单元测试, 还是benchmark测试.
执行go测试文件指令为:

>`go test [测试文件]|[指定路径下所有测试文件]`

# 基础单元测试
---
先看下书中示例:
```go
package listing01

import (
	"net/http"
	"testing"
)

const checkMark = "\u2713"
const ballotX = "\u2717"

// TestDownload validates the http Get function can download content.
func TestDownload(t *testing.T) {
	url := "https://medium.com/tag/golang"
	statusCode := 200

	t.Log("Given the need to test downloading content.")
	{
		t.Logf("\tWhen checking \"%s\" for status code \"%d\"",
			url, statusCode)
		{
			resp, err := http.Get(url)
			if err != nil {
				t.Fatal("\t\tShould be able to make the Get call.",
					ballotX, err)
			}
			t.Log("\t\tShould be able to make the Get call.",
				checkMark)

			defer resp.Body.Close()

			if resp.StatusCode == statusCode {
				t.Logf("\t\tShould receive a \"%d\" status. %v",
					statusCode, checkMark)
			} else {
				t.Errorf("\t\tShould receive a \"%d\" status. %v %v",
					statusCode, ballotX, resp.StatusCode)
			}
		}
	}
}
```

这个测试`TestDownload`用来测试`http.Get`函数: 在给定URL的情况下, 能否返回期待的response.

开头定义了两个常量: `checkMark` 跟`ballotX` 在命令行输出其实是`✓`, `✗`.

定义一个能被go识别的测试函数:

* 测试函数命名必须是公开函数.
* 测试函数必须以`Test`开头.
* 测试函数必须接受一个指向testing.T的指针, 并且不返回任何值.

testing.T这个参数很重要, 这个指针可以用来帮助记录每个测试的输出和状态.
尽管go没有定义测试输出格式, 但作为一个TDD粉, 我也是很中意书中的例子.
例子中使用了 `t.Log` 跟`t.Logf` 来记录测试输出, 让大家更容易理解这个测试具体测什么, 测试结果如何.
比如运行上述测试:
```
=== RUN   TestDownload
--- PASS: TestDownload (1.93s)
	listing01_test.go:16: Given the need to test downloading content.
	listing01_test.go:18: 	When checking "https://medium.com/tag/golang" for status code "200"
	listing01_test.go:26: 		Should be able to make the Get call. ✓
	listing01_test.go:32: 		Should receive a "200" status. ✓
PASS
ok  	command-line-arguments	1.939s
```
从上述输出中, 我们可以很清楚的看到, `TestDownload` 这个测试正在被运行, 这个测试会下载指定内容, 查看能不能successfully 生成一个指定地址的Http.Get, 如果能, 应该返回一个200的response编码.

如果和fail一个testcase? 在go单测中, 可以使用`t.Fatal` `t.Error`来使一个测试失败, 区别在于, `t.Fatal`会停止当前测试,而`t.Error`会继续当前测试.

# 表组测试
---
上面的测试, 我们讲述了一组输出对应组测试结果. 但有时候我们有大量数据来测试同一个函数, 如果因为不同输入而创建不同的测试函数显得冗余. go里面的表组测试可以用来测试一组不同的输入值跟期待的返回结果.
看下面的例子:

```go
package listing08

import (
	"net/http"
	"testing"
)

const checkMark = "\u2713"
const ballotX = "\u2717"

// TestDownload validates the http Get function can download
// content and handles different status conditions properly.
func TestDownload(t *testing.T) {
	var urls = []struct {
		url        string
		statusCode int
	}{
		{
			"http://www.goinggo.net/feeds/posts/default?alt=rss",
			http.StatusOK,
		},
		{
			"http://rss.cnn.com/rss/cnn_topstbadurl.rss",
			http.StatusNotFound,
		},
	}

	t.Log("Given the need to test downloading different content.")
	{
		for _, u := range urls {
			t.Logf("\tWhen checking \"%s\" for status code \"%d\"",
				u.url, u.statusCode)
			{
				resp, err := http.Get(u.url)
				if err != nil {
					t.Fatal("\t\tShould be able to Get the url.",
						ballotX, err)
				}
				t.Log("\t\tShould be able to Get the url.",
					checkMark)

				defer resp.Body.Close()

				if resp.StatusCode == u.statusCode {
					t.Logf("\t\tShould have a \"%d\" status. %v",
						u.statusCode, checkMark)
				} else {
					t.Errorf("\t\tShould have a \"%d\" status. %v %v",
						u.statusCode, ballotX, resp.StatusCode)
				}
			}
		}
	}
}
```

跟之前的基础单元测试不同的地方是, 我们定义了一个urls的数据切片被将其初始化.
```go
	var urls = []struct {
		url        string
		statusCode int
	}{
		{
			"http://www.goinggo.net/feeds/posts/default?alt=rss",
			http.StatusOK,
		},
		{
			"http://rss.cnn.com/rss/cnn_topstbadurl.rss",
			http.StatusNotFound,
		},
	}
```
urls的切片对象是一个匿名数据结构, 里面包含了两个参数, 用来`测试的url`, 跟期待的`返回response`.

下面的部分跟单元测试基本一致, 除了多了一个迭代结构, 从表组里迭代每一个要测试的url, 看测试函数是否返回表组里对应url的期待response.

# 模仿测试
---
很多时候, 我们写的函数会包含跟服务器, 数据库或者其他services之间的互动. 对于这类函数的测试, 使用真正的服务器, 数据库会给测试带来很多不便, 比如服务器不运行, 返回时间过长. 数据库数据因为测试受污染等等. 搭建测试服务器或者数据库成本又不是那么低廉. 然而我们的关注点是, 我们的函数能不能准确的处理服务器或者数据库返回的值, 其实对能不能链接真正的服务器或者数据库反而不是那么在意. 这时候, 我们需要模仿服务器或者数据库的behavior.

关于服务器模拟. go里提供了`httptest`包, 用来帮助我们模拟服务器返回的response.

书中的例子:
```go
package listing12

import (
	"encoding/xml"
	"fmt"
	"net/http"
	"net/http/httptest"
	"testing"
)

const checkMark = "\u2713"
const ballotX = "\u2717"

// feed is mocking the XML document we except to receive.
var feed = `<?xml version="1.0" encoding="UTF-8"?>
<rss>
<channel>
    <title>Going Go Programming</title>
    <description>Golang : https://github.com/goinggo</description>
    <link>http://www.goinggo.net/</link>
    <item>
        <pubDate>Sun, 15 Mar 2015 15:04:00 +0000</pubDate>
        <title>Object Oriented Programming Mechanics</title>
        <description>Go is an object oriented language.</description>
        <link>http://www.goinggo.net/2015/03/object-oriented</link>
    </item>
</channel>
</rss>`

// mockServer returns a pointer to a server to handle the get call.
func mockServer() *httptest.Server {
	f := func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(200)
		w.Header().Set("Content-Type", "application/xml")
		fmt.Fprintln(w, feed)
	}

	return httptest.NewServer(http.HandlerFunc(f))
}

// Item defines the fields associated with the item tag in
// the buoy RSS document.
type Item struct {
	XMLName     xml.Name `xml:"item"`
	Title       string   `xml:"title"`
	Description string   `xml:"description"`
	Link        string   `xml:"link"`
}

// Channel defines the fields associated with the channel tag in
// the buoy RSS document.
type Channel struct {
	XMLName     xml.Name `xml:"channel"`
	Title       string   `xml:"title"`
	Description string   `xml:"description"`
	Link        string   `xml:"link"`
	PubDate     string   `xml:"pubDate"`
	Items       []Item   `xml:"item"`
}

// Document defines the fields associated with the buoy RSS document.
type Document struct {
	XMLName xml.Name `xml:"rss"`
	Channel Channel  `xml:"channel"`
	URI     string
}

// TestDownload validates the http Get function can download content
// and the content can be unmarshaled and clean.
func TestDownload(t *testing.T) {
	statusCode := http.StatusOK

	server := mockServer()
	defer server.Close()

	t.Log("Given the need to test downloading content.")
	{
		t.Logf("\tWhen checking \"%s\" for status code \"%d\"",
			server.URL, statusCode)
		{
			resp, err := http.Get(server.URL)
			if err != nil {
				t.Fatal("\t\tShould be able to make the Get call.",
					ballotX, err)
			}
			t.Log("\t\tShould be able to make the Get call.",
				checkMark)

			defer resp.Body.Close()

			if resp.StatusCode != statusCode {
				t.Fatalf("\t\tShould receive a \"%d\" status. %v %v",
					statusCode, ballotX, resp.StatusCode)
			}
			t.Logf("\t\tShould receive a \"%d\" status. %v",
				statusCode, checkMark)

			var d Document
			if err := xml.NewDecoder(resp.Body).Decode(&d); err != nil {
				t.Fatal("\t\tShould be able to unmarshal the response.",
					ballotX, err)
			}
			t.Log("\t\tShould be able to unmarshal the response.",
				checkMark)

			if len(d.Channel.Items) == 1 {
				t.Log("\t\tShould have \"1\" item in the feed.",
					checkMark)
			} else {
				t.Error("\t\tShould have \"1\" item in the feed.",
					ballotX, len(d.Channel.Items))
			}
		}
	}
}
``` 

还是先前那个示例, `TestDownload` 用来测试http.Get. 这里我们定义了一个mockServer, mockServer对任意访问的request都会返回我们预先定义好的response `feed`.

这时候测试http.Get函数, 只需要将mockServr.URL 作为其参数即可.

# 测试服务端点
---
比起测试服务器response, 更多的时候我们需要测试自己写的API. 做测试的时候, 我们希望可以直接测试API, 而不需要启动服务器.
httptest也可以帮我们实现这点.

先看一个简单的服务器例子:
```go
package main

import (
	"log"
	"net/http"

	"github.com/goinaction/code/chapter9/listing17/handlers"
)

func main() {
	handlers.Routes()
	log.Println("监听: 启动, 监听端口: 4000.")
	http.ListenAndServe(":4000", nil)
}

```

```go
package handlers

import (
	"encoding/json"
	"net/http"
)
func Routes() {
	http.HandleFunc("/sendjson", SendJSON)
}

func SendJSON(rw http.ResponseWriter, r *http.Request) {
	u := struct {
		Name  string
		Email string
	}{
		Name:  "Bill",
		Email: "bill@ardanstudios.com",
	}

	rw.Header().Set("Content-Type", "applicatoin/json")
	rw.WriteHeader(200)
	json.NewEncoder(rw).Encode(&u)
}

```

这里我们有一个handler包, 里面定义了一个路由, 定义了一个"/sendjson" API, 当用户访问这个API时, 会返回一个JSON字符串, 里面包含了用户的姓名跟邮箱. 

在主函数里, 我们定义该路由一个实例, 开启一个服务器济监听端口`:4000`.

如果我们想要对这个API进行测试, 一个方法是, 一直开启这个服务器, 然后采取之前的测试方式. 但不可能一直开着服务器做测试啊...

这个时候我们可以通过外部对API的网络请求进行测试.
```go
func init() {
	handlers.Routes()
}

func TestSendJSON(t *testing.T) {
	t.Log("Given the need to test the SendJSON endpoint.")
	{
		req, err := http.NewRequest("GET", "/sendjson", nil)
		if err != nil {
			t.Fatal("\tShould be able to create a request.",
				ballotX, err)
		}
		t.Log("\tShould be able to create a request.",
			checkMark)

		rw := httptest.NewRecorder()
		http.DefaultServeMux.ServeHTTP(rw, req)

		if rw.Code != 200 {
			t.Fatal("\tShould receive \"200\"", ballotX, rw.Code)
		}
		t.Log("\tShould receive \"200\"", checkMark)

		u := struct {
			Name  string
			Email string
		}{}

		if err := json.NewDecoder(rw.Body).Decode(&u); err != nil {
			t.Fatal("\tShould decode the response.", ballotX)
		}
		t.Log("\tShould decode the response.", checkMark)

		if u.Name == "Bill" {
			t.Log("\tShould have a Name.", checkMark)
		} else {
			t.Error("\tShould have a Name.", ballotX, u.Name)
		}

		if u.Email == "bill@ardanstudios.com" {
			t.Log("\tShould have an Email.", checkMark)
		} else {
			t.Error("\tShould have an for Email.", ballotX, u.Email)
		}
	}
}
```

这个例子, 我们没有启动任何HTTP服务器, 但是通过httptest.NewRecorder() 创建了一个ResponseWriter. 并通过调用`http.DefaultServeMux.ServeHTTP` 模拟了外部对`/sendjson` API的调用请求.

go test 提供了许多其他功能, 比如写完测试, 要查看测试覆盖率, 可以使用 `go test -cover [对应测试文件]`
感兴趣的小伙伴可以通过`go  test -help` 了解更多.