# Wasm自适应熔断算法

参考go-zero自适应熔断规则（也是google sre提供规则），可考虑支持：

熔断原理与实现 · go-zero document
zRPC中熔断器的实现参考了Google Sre过载保护算法，该算法的原理如下：
- 请求数量(requests)：调用方发起请求的数量总和
- 请求接受数量(accepts)：被调用方正常处理的请求数量

在正常情况下，这两个值是相等的，随着被调用方服务出现异常开始拒绝请求，请求接受数量(accepts)的值开始逐渐小于请求数量(requests)，这个时候调用方可以继续发送请求，直到requests = K * accepts，一旦超过这个限制，熔断器就回打开，新的请求会在本地以一定的概率被抛弃直接返回错误，概率的计算公式如下：

通过修改算法中的K(倍值)，可以调节熔断器的敏感度，当降低该倍值会使自适应熔断算法更敏感，当增加该倍值会使得自适应熔断算法降低敏感度，举例来说，假设将调用方的请求上限从 requests = 2 *acceptst 调整为 requests = 1.1 *accepts 那么就意味着调用方每十个请求之中就有一个请求会触发熔断。

## gozero中自适应熔断算法实现关键代码

```go
func (b *googleBreaker) accept() error {
    accepts, total := b.history()  // 请求接受数量和请求总量
    weightedAccepts := b.k * float64(accepts)
  // 计算丢弃请求概率,参考https://landing.google.com/sre/sre-book/chapters/handling-overload/#eq2101
    dropRatio := math.Max(0, (float64(total-protection)-weightedAccepts)/float64(total+1))
    if dropRatio <= 0 {
        return nil
    }
    // 动态判断是否触发熔断
    if b.proba.TrueOnProba(dropRatio) {
        return ErrServiceUnavailable
    }
    return nil
}

func (p *Proba) TrueOnProba(proba float64) (truth bool) {
    p.lock.Lock()
    truth = p.r.Float64() < proba  //random number in [0.0,1.0)
    p.lock.Unlock()
    return
}
```

protection 是指请求数小于等于protection则不会触发熔断，默认5；k是熔断敏感系数，在1.1到2之间，默认1.5，越小越容易熔断。

## 目前使用公式

total > protection 触发熔断检查：
protection 是指请求数小于等于protection则不会触发熔断，默认5，支持调整；
熔断概率百分比为 ((total）- (k * accepts)/ total+1 ) * 100
k是熔断敏感系数，在1.1到2之间，默认1.5，越小越容易熔断
k * accepts >= total则不会熔断，否则k * accepts < total -protection 则 概率性熔断。

## 谷歌公式计算拒绝率，滚动窗口记录请求数和接受数

原理：
谷歌公式计算拒绝率:total-protection - k *Accepts /(total+1)
用桶来实现的移动窗口，记录窗口时间内的请求数量、接受数量（窗口10s， 40个桶， 250ms一个）

基于历史记录的来计算拒绝率：
accepts, total := b.history() //移动窗口的请求数、接受数
weightedAccepts := b.k * float64(accepts) //接受数权重 k=1.5，也就是接受百分比超过60%就不拒绝，否则概率性拒绝，例如接受0则接近百分百，接受30%,则拒绝率 55%  ((total - total*30% * 1.5) /total = 55%)
dropRatio := math.Max(0, (float64(total-b.protection)-weightedAccepts)/float64(total+1)) //拒绝率

[src: raw/ingested/3项目/服务网格-即构/zg_wasm技巧.md]