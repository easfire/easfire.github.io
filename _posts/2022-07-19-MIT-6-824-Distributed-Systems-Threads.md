![mit6.824](https://pic1.zhimg.com/v2-15016cef42422ccb60b8a842efe41fee_1440w.jpg?source=172ae18b)
## 6.824 分布式系统（Go Coordination） [MIT公开课]

MIT 6.824神作，就不用多余介绍，主讲老师 Robert Tappan Morris 的讲授，你值得拥有。

## MapReduce

一个简单MapReduce 实例流程图

![map-reduce](https://pic3.zhimg.com/80/v2-2932e6818a71967f1316b04c8366aae2_1440w.jpg)

Map function 把输入input （input maybe是文件or文件的一部分）处理下。上面的例子，计算所有输入的字符，并累加计算字符出现的总数。 Map function 输出一个map (k, v) k是word，比如 a，w，b等；v是 1，so reduce函数只需要取k，累加v，就可以等到结果。

从map func输出的行，到按k为列组合的转换过程，叫做 shuffle。

## Threads

### ① 一些概念

* I/O concurrency （IO的并发）
* Parallelism （并行的需要）
* Convenience （便利性）

这三条就是线程的存在的原因；Robert在回答学生的提问中，回答了一些很有意思的问题。
