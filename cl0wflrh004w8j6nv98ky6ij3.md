## Utilize CPU with Golang

Running jobs efficiently is an important part when developing large scale products. This blog will show 4 methods which are usually used: sequence, burst all, burst n, and worker pool. These approaches can be done via various programming languages, but we will use Golang for demonstration. Golang has wonderful features supporting for concurrency natively. So, let's go.

Github: https://github.com/liamhieuvu/ultilize-cpu-with-goroutine

# Level 1: Sequence

This method processes all jobs sequentially. It is very simple and perfectly fit for jobs having short processing time. However, it will waste a lot of CPU resource with IO bound jobs. The below program simulates job handling by delaying a specific time according to `Duration` data in `item`.

![Screen Shot 2022-03-18 at 7.25.08 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647606405530/_CNxhoJvs.png)

`sequence.go`

```go
package main

import (
	"log"
	"time"
)

type item struct {
	ID       string
	Duration time.Duration
}

func main() {
	items := []item{{"A", 1}, {"B", 2}, {"C", 1}, {"D", 3}, {"E", 2}}

	start := time.Now()
	for _, data := range items {
		process(data)
	}
	log.Print("processed time: ", time.Since(start))
}

func process(i item) {
	log.Print(i.ID, " start")
	time.Sleep(i.Duration * time.Second)
	log.Print(i.ID, " end")
}
```

```text
$ go run sequence.go
2022/03/18 18:37:47 A start
2022/03/18 18:37:48 A end
2022/03/18 18:37:48 B start
2022/03/18 18:37:50 B end
2022/03/18 18:37:50 C start
2022/03/18 18:37:51 C end
2022/03/18 18:37:51 D start
2022/03/18 18:37:54 D end
2022/03/18 18:37:54 E start
2022/03/18 18:37:56 E end
2022/03/18 18:37:56 processed time: 9.005201125s
```

# Level 2: Burst all jobs

This method will handle all jobs at the same time. It may cause a pike of CPU usage, but it guarantees the shortest processing time. It is perfectly fit for small size of items. However, it will face problems with IO rate limit due to issuing a huge number of requests to them.

![Screen Shot 2022-03-18 at 7.25.26 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647606429605/uHvZeiSWS.png)

`burst_all.go`

```go
package main

import (
	"log"
	"sync"
	"time"
)

type item struct {
	ID       string
	Duration time.Duration
}

func main() {
	items := []item{{"A", 1}, {"B", 2}, {"C", 1}, {"D", 3}, {"E", 2}}
	start := time.Now()

	var wg sync.WaitGroup
	wg.Add(len(items))
	for _, data := range items {
		go func(i item) {
			defer wg.Done()
			process(i)
		}(data)
	}
	wg.Wait()

	log.Print("processed time: ", time.Since(start))
}

func process(i item) {
	log.Print(i.ID, " start")
	time.Sleep(i.Duration * time.Second)
	log.Print(i.ID, " end")
}
```

```text
$ go run burst_all.go
2022/03/18 18:38:30 E start
2022/03/18 18:38:30 A start
2022/03/18 18:38:30 D start
2022/03/18 18:38:30 C start
2022/03/18 18:38:30 B start
2022/03/18 18:38:31 C end
2022/03/18 18:38:31 A end
2022/03/18 18:38:32 E end
2022/03/18 18:38:32 B end
2022/03/18 18:38:33 D end
2022/03/18 18:38:33 processed time: 3.001483041s
```

# Level 3: Burst n jobs sequentially

This method will handle n jobs at the same time, then processes next n jobs until finishing all jobs. It solves rate limit problems of burst all method. It suits the case when n jobs have equal processing time. However, it will waste resources if one of n jobs has much longer processing time than others.

![Screen Shot 2022-03-18 at 7.25.37 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647606443163/SNn85uWU1.png)

`burst_n.go`

```go
package main

import (
	"log"
	"sync"
	"time"
)

type item struct {
	ID       string
	Duration time.Duration
}

func main() {
	items := []item{{"A", 1}, {"B", 2}, {"C", 1}, {"D", 3}, {"E", 2}}
	n := 2
	start := time.Now()
	for s := 0; s < len(items); s += n {
		e := s + 2
		if s+n > len(items) {
			e = len(items)
		}
		processN(items[s:e])
	}
	log.Print("processed time: ", time.Since(start))
}

func processN(items []item) {
	var wg sync.WaitGroup
	wg.Add(len(items))
	for _, data := range items {
		go func(i item) {
			defer wg.Done()
			process(i)
		}(data)
	}
	wg.Wait()
}

func process(i item) {
	log.Print(i.ID, " start")
	time.Sleep(i.Duration * time.Second)
	log.Print(i.ID, " end")
}
```

```text
$ go run burst_n.go
2022/03/18 18:39:06 B start
2022/03/18 18:39:06 A start
2022/03/18 18:39:07 A end
2022/03/18 18:39:08 B end
2022/03/18 18:39:08 C start
2022/03/18 18:39:08 D start
2022/03/18 18:39:09 C end
2022/03/18 18:39:11 D end
2022/03/18 18:39:11 E start
2022/03/18 18:39:13 E end
2022/03/18 18:39:13 processed time: 7.003122542s
```

# Level 4: Worker pool

This method is like "burst n" method but solves unequal processing time cases. It has nice performance in most cases, but may make the code more complex. Like greedy algorithm, be careful about the order of tasks.

![Screen Shot 2022-03-18 at 7.25.49 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647606459502/J3kuO4WZ7.png)

`worker_n.go`

```go
package main

import (
	"log"
	"time"
)

type item struct {
	ID       string
	Duration time.Duration
}

func main() {
	items := []item{{"A", 1}, {"B", 2}, {"C", 1}, {"D", 3}, {"E", 2}}
	n := 2
	data := make(chan item, len(items))
	results := make(chan error, len(items))
	start := time.Now()

	for w := 1; w <= n; w++ {
		go worker(data, results)
	}

	for _, d := range items {
		data <- d
	}
	close(data)

	for i := 1; i <= len(items); i++ {
		<-results
	}

	log.Print("processed time: ", time.Since(start))
}

func worker(data <-chan item, results chan<- error) {
	for d := range data {
		process(d)
		results <- nil
	}
}

func process(i item) {
	log.Print(i.ID, " start")
	time.Sleep(i.Duration * time.Second)
	log.Print(i.ID, " end")
}
```

```text
$ go run worker_n.go
2022/03/18 18:39:33 A start
2022/03/18 18:39:33 B start
2022/03/18 18:39:34 A end
2022/03/18 18:39:34 C start
2022/03/18 18:39:35 B end
2022/03/18 18:39:35 D start
2022/03/18 18:39:35 C end
2022/03/18 18:39:35 E start
2022/03/18 18:39:37 E end
2022/03/18 18:39:38 D end
2022/03/18 18:39:38 processed time: 5.002048s
```

That's all. Use 4 approaches wisely - trade off between complexity and performance. Thank you for reading my blogs.


