redis热点key问题分两个：如何找到热点key，如何处理热点key

一、如何发现热点key
1. redis slot的qps监控，对redis集群的每个slot做流量监控，粒度太粗
2. redis proxy的监控，在redis proxy中对key的访问做一个计数(redis6.0推出了集群proxy)
3. redis4.0支持每个节点上基于lfu进行key热点机制，客户端命令rediscli -hotkeys可以获得key访问次数，定时查询这个信息就可以获得热点key
4. 对客户端的改造，在客户端做key访问计数，这个就需要公司基建
5. 业务埋点，服务端对key访问计数

二、处理热点key
1. 缓存预热：实际上部分热点key是可以通过业务可预测的，提前缓存好这部分数据
2. 二级缓存（本地缓存）：每次获得热点key时在本地内存中缓存一份，当缓存过期再重新从redis获取，查询数据先查本地缓存，查不到再查redis，再查不到查数据库
2.1 本地缓存最大的问题就是缓存一致性，需要watch数据的变更（redis watcher）
3. 热点key拆分到redis集群不同节点，redis集群是分布式的，可以降低压力
4. 限流：对于超过单秒qps压力的部分请求，通过限流算法进行限流

三、限流算法
1. 常用的限流算法有四种：计数器、滑动窗口、漏桶、令牌桶
2. 计数器：在固定窗口时间内，对请求进行计数，超过计数就限流，当到达最大窗口时间对计数器清空并重置窗口开始时间
`
package main

import (
	"fmt"
	"sync"
	"time"
)

const (
	MAXREQUEST = 10              // 限制最大请求数
	WINDOWTIME = 3 * time.Second // 最大窗口时间
)

type limit struct {
	beginTime time.Time
	counter   int
	mu        sync.Mutex
}

func (limit *limit) apiLimit() bool {
	limit.mu.Lock()
	defer limit.mu.Unlock()

	nowTime := time.Now()

	if nowTime.Sub(limit.beginTime) >= WINDOWTIME {
		limit.beginTime = nowTime
		limit.counter = 0
	}

	if limit.counter > MAXREQUEST {
		return false
	}

	limit.counter++
	fmt.Println("counter: ", limit.counter)
	return true
}

func main() {
	var wg sync.WaitGroup
	var limit limit

	for i := 0; i < 15; i++ {
		wg.Add(1)

		fmt.Println("req start:", i, time.Now())

		go func(i int) {
			if limit.apiLimit() {
				fmt.Println("req counter: ", i, time.Now())
			}
			wg.Done()
		}(i)
		time.Sleep(150 * time.Millisecond)
	}
	wg.Wait()
}
`
2.1 计数器算法有一个临界时间问题，对于限制1分钟内30个请求，如果当前分钟最后1秒30个请求，下一分钟第1秒30个请求，符合计数器算法，但是对服务每秒30的请求是不能承受的；计数器算法的缺点就是无法处理临近时间的突发请求，用滑动窗口法可以解决部分，思想就是将大的时间分为多个小时间片，随时间变动每次清零小时间片


3. 滑动窗口：在计数器的基础上，将最大窗口时间划分为多个小窗口，每个小窗口有计数器;每次请求将遍历所有小窗口，从当前时间往前数最大窗口时间，超过的小窗口清空，然后判断当前最大窗口内次数，当前请求是否限流
`
package main

import (
	"fmt"
	"sync"
	"time"
)

const (
	MAXREQUEST = 300              // 限制最大请求数
	WINDOWTIME = 30 * time.Second // 最大窗口时间
)

type SlidingWindow struct {
	smallWindowTime int64         // 小窗口时间大小
	smallWindowNum  int64         // 小窗口总数
	smallWindowCap  int           // 小窗口请求容量
	counters        map[int64]int // 小窗口计数器
	mu              sync.Mutex    // 锁
}

func NewSlidingWindow(smallWindowTime time.Duration) (*SlidingWindow, error) {
	num := int64(WINDOWTIME / smallWindowTime) // 小窗口总数
	return &SlidingWindow{
		smallWindowTime: int64(smallWindowTime),
		smallWindowNum:  num,
		smallWindowCap:  MAXREQUEST / int(num),
		counters:        make(map[int64]int),
	}, nil
}

func (sw *SlidingWindow) ReqLimit() bool {
	sw.mu.Lock()
	sw.mu.Unlock()

	// 获取当前小格子窗口所在的时间值
	curSmallWindowTime := time.Now().Unix()
	// 计算当前小格子窗口起始时间
	beginTime := curSmallWindowTime - sw.smallWindowTime*(sw.smallWindowNum-1) 
    
	// 计算当前小格子窗口请求总数
	var count int
	for sWindowTime, counter := range sw.counters { // 遍历计数器
		if sWindowTime < beginTime { // 判断不是当前小格子
			delete(sw.counters, sWindowTime) 
		} else {
			count += counter // 当前小格子窗口计数器累加
		}
	}

	// 当前小格子请求到达请求限制，请求失败，返回 false
	if count >= sw.smallWindowCap {
		return false
	}

	// 没有到达请求上限，当前小格子窗口计数器+1，请求成功
	sw.counters[curSmallWindowTime]++
	return true
}

func main() {
	var wg sync.WaitGroup
	sw, _ := NewSlidingWindow(1 * time.Second)
	fmt.Println("num:", sw.smallWindowNum, "cap:", sw.smallWindowCap)

	for i := 0; i < 15; i++ {
		wg.Add(1)

		fmt.Println("req start:", i, time.Now())

		go func(i int) {
			if sw.ReqLimit() {
				fmt.Println("req counter: ", time.Now())
			}
			wg.Done()
		}(i)
		time.Sleep(200 * time.Millisecond)
	}
	wg.Wait()
}
`
3.1 只能解决部分临界时间问题，上面算法解决的是秒内的临近时间问题，如果是毫秒又无法解决，所以解决时间临界问题看小窗口时间分片的大小

4. 漏桶算法：桶以固定的流出速率，流入桶的速率是不确定的，把流入桶的请求计数，当超出桶容量，丢弃溢出的请求
`
package main

import (
	"fmt"
	"sync"
	"time"
)

const (
	MAXREQUEST = 5
)

type LeakyBucket struct {
	capacity      int        // 桶的容量 - 最高水位
	currentReqNum int        // 桶中当前请求数量 - 当前水位
	lastTime      time.Time  // 桶中上次请求时间 - 上次放水时间
	rate          int        // 桶中流出请求的速率，每秒流出多少请求,水流速度/秒
	mu            sync.Mutex // 锁
}

func NewLeakyBucket(rate int) *LeakyBucket {
	return &LeakyBucket{
		capacity: MAXREQUEST, //容量
		lastTime: time.Now(),
		rate:     rate,
	}
}

func (lb *LeakyBucket) ReqLimit() bool {
	lb.mu.Lock()
	lb.mu.Unlock()

	now := time.Now()
	// 计算距离上次放水时间间隔
	gap := now.Sub(lb.lastTime)
	fmt.Println("gap：", gap)
	if gap >= time.Second {
		// gap 这段时间流出的请求数=gap时间 * 每秒流出速率
		out := int(gap/time.Second) * lb.rate

		// 计算当前桶中请求数
		lb.currentReqNum = maxInt(0, lb.currentReqNum-out)
		lb.lastTime = now
	}

	// 桶中的当前请求数大于桶容量，请求失败
	if lb.currentReqNum >= lb.capacity {
		return false
	}

	// 若没超过桶容量，桶中请求量+1，返回true
	lb.currentReqNum++
	fmt.Println("curReqNum:", lb.currentReqNum)
	return true
}

func maxInt(a, b int) int {
	if a > b {
		return a
	}
	return b
}

// 测试
func main() {
	var wg sync.WaitGroup

	lb := NewLeakyBucket(1)
	fmt.Println("cap:", lb.capacity)

	for i := 0; i < 15; i++ {
		wg.Add(1)

		fmt.Println("req start:", i, time.Now())

		go func(i int) {
			if lb.ReqLimit() {
				fmt.Println("req counter: ", time.Now())
			}
			wg.Done()
		}(i)
		time.Sleep(10 * time.Millisecond)
	}
	wg.Wait()
}
`
4.1 漏桶算法虽然能保护系统，但当请求数量过多时会造成大部分请求失败

5. 令牌桶算法：相比漏桶算法，存储在桶中的是令牌，请求必须得到令牌才能请求，否则就丢掉请求
`
package main

import (
	"fmt"
	"sync"
	"time"
)

const (
	MAXREQUEST = 5
)

type TokenBucket struct {
	capacity        int       // 桶的容量
	currentTokenNum int       // 当前桶中tokent总量
	lastTime        time.Time // 上次发放 tokent 时间
	rate            int       // 放入tokent 速率， token/s - 每秒放入多少个token
	mu              sync.Mutex
}

func NewTokenBucket(rate int) *TokenBucket {
	return &TokenBucket{
		capacity: MAXREQUEST,
		lastTime: time.Now(),
		rate:     rate,
	}
}

func (tb *TokenBucket) ReqLimit() bool {
	tb.mu.Lock()
	tb.mu.Unlock()

	now := time.Now()
	// 距离上次发放令牌间隔时间
	gap := now.Sub(tb.lastTime)
	fmt.Println("gap: ", gap)
	if gap >= time.Second {
		// 这段时间发放的token数 = gap时间(秒为单位) * 每秒发放token速率
		in := int(gap/time.Second) * tb.rate
		// 这段时间桶中的token总量
		// tb.currentTokenNum+in : 当前已有的token总量+发放的token树
		// 当前桶中的token数量当然不能超过桶的最大容量
		tb.currentTokenNum = minInt(tb.capacity, tb.currentTokenNum+in)
		tb.lastTime = now
	}

	// 没有token，请求失败
	if tb.currentTokenNum <= 0 {
		return false
	}

	// 如果有token，当前 token 数量-1，也就是发放token，请求成功
	tb.currentTokenNum--
	fmt.Println("curTokenNum:", tb.currentTokenNum)
	return true
}

func minInt(a, b int) int {
	if a < b {
		return a
	}
	return b
}

// 测试
func main() {
	var wg sync.WaitGroup

	tb := NewTokenBucket(3)
	fmt.Println("cap:", tb.capacity)

	for i := 0; i < 15; i++ {
		wg.Add(1)

		fmt.Println("req start:", i, time.Now())

		go func(i int) {
			if tb.ReqLimit() {
				fmt.Println("req counter: ", time.Now())
			}
			wg.Done()
		}(i)
		time.Sleep(1 * time.Second)
	}
	wg.Wait()
}
`
5.1 因为令牌桶可以存储令牌，可以支持一定程度的突发请求
5.2 golang中已实现令牌桶golang.org/x/time/rate

6. 四种限流算法
6.1 计数器：时间空间复杂度低，实现简单，但是限制突发流量和平滑限流都差
6.2 滑动窗口：空间复杂度高，时间复杂度中等，实现简单，限制突发流量和平滑限流都依靠滑动的小窗口大小
6.3 漏桶算法：空间复杂度低，时间复杂度高，实现难度高，平滑限流通过固定的流出，限制突发流量差；如果需要严格保证请求速率，漏桶算法则比令牌桶更适合
6.4 令牌桶：空间复杂度低，时间复杂度高，实现难度高，平滑限流和限制突发流量都很强