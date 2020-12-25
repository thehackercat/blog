# Golang 实现装饰器

## Go 实现装饰器

类似于 Java 中的装饰器模式, 在 go 中也会有需要 patch 一个函数的需求.

一个常见的场景就是比如计算某个函数耗时的 decorator, 那么利用 functional programming 思想. 在 go 中的实现如下:

```
func timeSpent(inner func(n int) int) func(op int) int {
	return func(n int){
		start := time.Now()
		ret := inner(n)
		fmt.Println("time spent: ", time.Since(start).Seconds())
		return ret
	}
}
```

当然实际场景中可能更多需要的是一种 pipeline 装饰器的用法, 比如一个 http handler 需要进行 Auth 认证, 认证通过后需要进行耗时计算, 计算完耗时需要进行 statsd 打点. 那么对应的实现如下:

```go
func StatsdTiming() func(http.handler) http.Handler {
	return func(next http.Handler) http.Hanlder {
		fn := func(w http.ResponseWriter, r *http.Request) {
			lbSpanTime := time.Duration(0)
			if parsed, err := strconv.ParseFloat(r.Header.Get("X-Request-Start"), 64); err == nil {
				sec := int64(parsed)
				nsec := int64((parsed - float64(sec)) * 1000000000)
				lbStartTime := time.Unix(sec, nsec)
				lbSpanTime = time.Now().Sub(lbStartTime)
				statsdClient.TimingDuration(fmt.Sprintf("reqtime", lbSpanTime))
			}
		}
	}
}

func Auth() func(http.handler) http.Handler {
    return func(next http.Handler) http.Hanlder {
        function_name_str := runtime.FuncForPC(reflect.ValueOf(input).Pointer()).Name()

        function_name_array := strings.Split(function_name_str, "/")
        module_method := strings.Split(function_name_array[len(function_name_array)-1], ".")
        module := module_method[0]
        method := module_method[1]

        if module != "Login" {
            abort("Not Auth")
        }
    }
}

http.HandleFunc("/hello", Handler(hello,
                Auth, StatsdTiming))
```

