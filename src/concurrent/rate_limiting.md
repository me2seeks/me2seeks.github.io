# Rate limiting

## 什么是速率限制？

本质上，速率限制是控制应用程序用户在指定时间范围内可以发出多少请求的过程。您可能想要这样做有几个原因，例如：

- 确保您的应用程序无论收到多少传入流量都能正常运行
- 对某些用户设置使用限制，例如实施有上限的免费层或公平使用政策
- 保护您的应用程序免受`DOS`和其他网络攻击

由于这些原因，速率限制是所有Web应用程序迟早必须集成的东西。但是它是如何工作的呢？这要看情况。开发人员可以在速率限制他们的应用程序时使用几种算法。

### 令牌桶算法

在令牌桶算法技术中，令牌以每次固定的速率添加到桶中，桶是某种存储。应用程序处理的每个请求都会消耗桶中的一个令牌。桶有固定的大小，因此令牌不能无限堆积。当桶用完令牌时，新的请求会被拒绝。

### 漏桶算法

在漏桶算法技术中，请求在到达时被添加到桶中，然后从桶中删除，以便以固定速率进行处理。如果桶满了，其他请求要么被拒绝，要么被延迟。

### 固定窗口算法

固定窗口算法技术跟踪在固定时间窗口内发出的请求数量，例如，每五分钟一次。如果一个窗口中的请求数量超过预定限制，其他请求要么被拒绝，要么被延迟到下一个窗口的开始。

### 滑动窗口算法

与固定窗口算法一样，滑动窗口算法技术跟踪时间滑动窗口上的请求数量。虽然窗口的大小是固定的，但窗口的开始是由用户第一次请求的时间决定的，而不是任意时间间隔。如果窗口中的请求数量超过预设限制，后续请求要么被丢弃，要么被延迟。

每种算法都有其独特的优点和缺点，因此选择合适的技术取决于应用程序的需要。话虽如此，令牌和漏桶变体是最流行的速率限制形式。

## 在Go应用程序中实现速率限制

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type Message struct {
    Status string `json:"status"`
    Body   string `json:"body"`
}

func endpointHandler(writer http.ResponseWriter, request *http.Request) {
    writer.Header().Set("Content-Type", "application/json")
    writer.WriteHeader(http.StatusOK)
    message := Message{
        Status: "Successful",
        Body:   "Hi! You've reached the API. How may I help you?",
    }
    err := json.NewEncoder(writer).Encode(&message)
    if err != nil {
        return
    }
}

func main() {
    http.HandleFunc("/ping", endpointHandler)
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Println("There was an error listening on port :8080", err)
    }
}
```

```go
func rateLimiter(next func(w http.ResponseWriter, r *http.Request)) http.Handler {
    limiter := rate.NewLimiter(2, 4)
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow() {
            message := Message{
                Status: "Request Failed",
                Body:   "The API is at capacity, try again later.",
            }

            w.WriteHeader(http.StatusTooManyRequests)
            json.NewEncoder(w).Encode(&message)
            return
        } else {
            next(w, r)
        }
    })
}
```

```go
func main() {
    http.Handle("/ping", rateLimiter(endpointHandler))
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Println("There was an error listening on port :8080", err)
    }
} 
```

```bash
for i in {1..6}; do curl http://localhost:8080/ping; done
```

## 每客户速率限制

尽管我们当前的中间件可以工作，但它有一个明显的缺陷：应用程序作为一个整体受到速率限制。如果一个用户发出四个请求的突发，所有其他用户将无法访问API。我们可以通过使用一些唯一属性来识别每个用户来单独限制每个用户的速率来纠正这一点。

我们还将存储客户端最后一次发出请求的时间，所以一旦客户端在一定时间后没有发出请求，您可以删除它们的限制器以节省应用程序的内存。最后一个难题是使用互斥锁来保护存储的客户端数据免受并发访问。

以下是新中间件的代码：

```go
 func perClientRateLimiter(next func(writer http.ResponseWriter, request *http.Request)) http.Handler {
    type client struct {
        limiter  *rate.Limiter
        lastSeen time.Time
    }
    var (
        mu      sync.Mutex
        clients = make(map[string]*client)
    )
    go func() {
        for {
            time.Sleep(time.Minute)
            // Lock the mutex to protect this section from race conditions.
            mu.Lock()
            for ip, client := range clients {
                if time.Since(client.lastSeen) > 3*time.Minute {
                    delete(clients, ip)
                }
            }
            mu.Unlock()
        }
    }()

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract the IP address from the request.
        ip, _, err := net.SplitHostPort(r.RemoteAddr)
        if err != nil {
            w.WriteHeader(http.StatusInternalServerError)
            return
        }
        // Lock the mutex to protect this section from race conditions.
        mu.Lock()
        if _, found := clients[ip]; !found {
            clients[ip] = &client{limiter: rate.NewLimiter(2, 4)}
        }
        clients[ip].lastSeen = time.Now()
        if !clients[ip].limiter.Allow() {
            mu.Unlock()

            message := Message{
                Status: "Request Failed",
                Body:   "The API is at capacity, try again later.",
            }

            w.WriteHeader(http.StatusTooManyRequests)
            json.NewEncoder(w).Encode(&message)
            return
        }
        mu.Unlock()
        next(w, r)
    })
}
```

这部分代码做了一些事情：

- 定义名为`client`的结构类型以保存限制器和每个客户端的`lastSeen`时间
- 创建互斥锁和字符串映射以及指向`client`结构的指针称为`clients`
- 创建每分钟运行一次的Go例程并删除`client`结构，其`lastSeen`时间比`clients`映射的当前时间大三分钟

下一部分是匿名函数。函数：

- 从传入请求中提取IP地址
- 检查地图中当前IP地址是否有限制器，如果没有，则添加一个
- 更新IP地址的`lastSeen`时间
- 使用特定于IP地址的限制器以与之前的速率限制器相同的方式对请求进行速率限制

有了这个，您已经实现了将分别限制每个客户端的中间件！最后一步是简单地注释掉`rateLimiter`，并在您的`main`函数中将其替换为`perClientRateLimiter`。

## Rate limiting with Tollbooth

[Tollbooth](https://github.com/didip/tollbooth/)它使用令牌桶算法以及`golang.org`的 `x/time/rate`

注释掉`perClientRateLimiter`并用以下代码替换您的`main`函数：

```go
func main() {
    message := Message{
        Status: "Request Failed",
        Body:   "The API is at capacity, try again later.",
    }
    jsonMessage, _ := json.Marshal(message)

    tlbthLimiter := tollbooth.NewLimiter(1, nil)
    tlbthLimiter.SetMessageContentType("application/json")
    tlbthLimiter.SetMessage(string(jsonMessage))

    http.Handle("/ping", tollbooth.LimitFuncHandler(tlbthLimiter, endpointHandler))
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        log.Println("There was an error listening on port :8080", err)
    }
}
```

现在以每秒一个请求的速率限制了端点。你的新main函数使用`tollbooth.NewLimiter`创建`tollbooth.Limiter`，指定自定义JSON拒绝消息，然后为`/ping`端点注册限制器和处理程序。

## 实现其他算法的库

- [`ratelimit`](https://github.com/uber-go/ratelimit): 漏水桶
- [`rerate`](https://github.com/abo/rerate): 固定窗口 (Redis-based)
- [`slidingwindow`](https://github.com/RussellLuo/slidingwindow): 滑动窗口
