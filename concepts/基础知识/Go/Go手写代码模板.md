# Go手写代码模板

See also: [[设计模式]]

## 连接池
```go
type Pool struct {
    mu    sync.Mutex
    conns chan *Conn
    factory func() (*Conn, error)
}

func (p *Pool) Get() (*Conn, error) {
    p.mu.Lock()
    defer p.mu.Unlock()
    select {
    case conn := <-p.conns:
        if conn.IsClosed() {
            return p.factory()
        }
        return conn, nil
    default:
        return p.factory()
    }
}

func (p *Pool) Put(conn *Conn) {
    p.mu.Lock()
    defer p.mu.Unlock()
    if conn.IsClosed() {
        return
    }
    select {
    case p.conns <- conn:
    default:
        conn.Close()
    }
}
```

## 限流器
```go
type RateLimiter struct {
    rate  int
    burst int
    tokens int
    last time.Time
    mu sync.Mutex
}

func (rl *RateLimiter) Allow() bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    now := time.Now()
    elapsed := now.Sub(rl.last).Seconds()
    rl.tokens += int(elapsed * float64(rl.rate))
    if rl.tokens > rl.burst {
        rl.tokens = rl.burst
    }
    if rl.tokens > 0 {
        rl.tokens--
        return true
    }
    return false
}
```

## 分布式锁
```go
type DistributedLock struct {
    redis *redis.Client
    key   string
    ttl   time.Duration
}

func (l *DistributedLock) Lock(ctx context.Context) (bool, error) {
    result, err := l.redis.SetNX(ctx, l.key, "1", l.ttl).Result()
    return result, err
}

func (l *DistributedLock) Unlock(ctx context.Context) error {
    return l.redis.Del(ctx, l.key).Err()
}
```

[src: raw/ingested/2技术/go/Go语言知识.md]

## Related Pages
- [[设计模式]]
- [[Go接口]]
- [[Go语言基础]]
