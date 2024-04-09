# Circuit breaking

[Circuit Breaker Pattern](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/dn589784(v=pandp.10)) 
该`microsoft`文章指出处理连接到远程服务或资源时可能需要不同时间来纠正的故障。`Circuit Breaker Pattern`可以提高应用程序的稳定性和弹性。

`Circuit breaking`

- 避免级联故障：在微服务架构中，服务之间通过网络调用互相通讯，一个服务的不可用可能导致对它的依赖服务出现雪崩效应。

- 保护系统：当外部服务响应缓慢或失败时，断路器可以阻止内部服务不断重试，从而保护系统不受进一步影响。

- 快速失败：用户不需要等待长时间的响应，当一定条件触发时，立即返回失败，这提高了用户体验。

- 资源节省：减少了对故障服务的调用，防止了不必要的资源消耗和等待时间。

![](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/images/dn589784.ae4c3e59526d69403f5bacc7840b1fb5(en-us,pandp.10).png)

省略`stateToString` `printLog` `printChange` 等打印函数
```go
var logChan = make(chan string, 1) // 创建日志通道
var changeChan = make(chan string, 1)
settings := gobreaker.Settings{
		Name:        "Mock Breaker",
    //- Name：熔断器的名字，用于区分不同的熔断器实例。
		ReadyToTrip: getReadyToTripFunc(logChan),
    //- ReadyToTrip：一个函数，它定义了熔断器从关闭状态转换到开启状态的条件。gobreaker.Counts 包含失败、成功等请求的计数。
		OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
			changeChan <- fmt.Sprintf("Circuit breaker '%s' state changed from %s to %sn", name, stateToString(from), stateToString(to))
		},
    //- OnStateChange：状态变化时的回调函数，当熔断器的状态发生变化（例如从关闭到开启）时会被调用。这可以用来记录日志或者进行其他操作。
	}
/*
- MaxRequests：在半开状态时允许通过的最大请求数。熔断器从打开状态转换到半开状态时，会允许有限数量的请求通过以检测系统的健康状况。如果这些请求都成功了，熔断器会关闭，系统恢复正常状态。
- Interval：在打开状态时，熔断器尝试恢复的时间间隔。一旦过了这个时间，熔断器会转到半开状态，允许部分请求尝试执行。
- Timeout：熔断器开启状态的持续时间。在此期间所有尝试通过的请求都会立即被拒绝。一旦超时，熔断器会转换到半开状态。
- IsSuccessful：一个函数，用于判定一个操作是否成功。通常根据错误值来判断，如果没有错误，则认为操作成功。这个函数被用来更新成功和失败的计数，它们是决定是否熔断的关键因素。
*/
cb := gobreaker.NewCircuitBreaker(settings)

// getReadyToTripFunc 返回一个闭包，用于确定断路器何时触发
func getReadyToTripFunc(logChan chan string) func(gobreaker.Counts) bool {
	return func(counts gobreaker.Counts) bool {
		if counts.Requests < 3 {
			return false
		}
		failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
		if failureRatio >= 0.6 {
			logChan <- "failure ratio too high"
			return true
		}
		return false
	}
}
```

模拟请求

```go
for i := 0; i < 20; i++ {
		// 执行并通过断路器保护的操作
		_, err := cb.Execute(func() (interface{}, error) {
			// 随机返回错误以模拟失败请求
			if i%2 == 0 {
				return nil, errors.New("error")
			}
			return "success", nil
		})

		if err != nil {
			fmt.Println("Operation failed:", err.Error())
		} else {
			fmt.Println("Operation succeeded.")
		}

		printLog(logChan) // 检查并打印日志消息
		printChange(changeChan)

		time.Sleep(500 * time.Millisecond) // 等待一段时间再尝试下一个请求
	}
```

