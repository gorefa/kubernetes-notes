

### **Counter**

只增不减的计数器
计数器可以用于记录只会增加不会减少的指标类型,比如记录应用请求的总量，cpu使用时间等。

对于Counter类型的指标，只包含一个inc()方法，用于计数器+1。

一般而言，Counter类型的metrics指标在命名中我们使用_total结束，如http_requests_total。

### **Gauge** 

可增可减的仪表盘
对于这类可增可减的指标，可以用于反应应用的当前状态。

例如在监控主机时，主机当前空闲的内存大小，可用内存大小。或者容器当前的cpu使用率,内存使用率。

对于Gauge指标的对象则包含两个主要的方法inc()以及dec(),用户添加或者减少计数。

### **Histogram**

自带buckets区间用于统计分布统计图
主要用于在指定分布范围内(Buckets)记录大小或者事件发生的次数。

### **Summary** 

客户端定义的数据分布统计图
Summary和Histogram非常类型相似，都可以统计事件发生的次数或者大小，以及其分布情况。

Summary和Histogram都提供了对于事件的计数_count以及值的汇总_sum。 因此使用_count,和_sum时间序列可以计算出相同的内容，例如http每秒的平均响应时间：rate(basename_sum[5m]) / rate(basename_count[5m])。

同时Summary和Histogram都可以计算和统计样本的分布情况，比如中位数，9分位数等等。其中 0.0<= 分位数Quantiles <= 1.0。

不同在于Histogram可以通过histogram_quantile函数在服务器端计算分位数。 而Sumamry的分位数则是直接在客户端进行定义。因此对于分位数的计算。 Summary在通过PromQL进行查询时有更好的性能表现，而Histogram则会消耗更多的资源。相对的对于客户端而言Histogram消耗的资源更少。



假设样本的 9 分位数（quantile=0.9）的值为 x，即表示小于 x 的采样值的数量占总体采样值的 90%



![image-20200511113909450](/Users/lisai/go/src/llussy.github.io/images/index.png)

https://fuckcloudnative.io/prometheus/2-concepts/metric_types.html



### 参考

[prometheus metrics](https://www.cnblogs.com/duanxz/p/10168490.html)

[histogram_quantile 相关的若干问题](http://disksing.com/histogram-quantile/)