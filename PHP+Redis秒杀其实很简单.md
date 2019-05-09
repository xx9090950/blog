###前言： 
秒杀这个问题，一直以来都是经典的面试题。但是秒杀也分大小。如果一个产品的用户不超过5w，上来就问双十一级别的秒杀。那就没有意思了~，所以今天就简单聊下一般条件下的秒杀的思路。方法只有两个，一个是装载秒杀商品。一个就是模拟用户进场秒杀。
![图片发自简书App](http://upload-images.jianshu.io/upload_images/6016628-2a2b0cda41f1136a.jpg)


####工具介绍
首先环境就比较简单 
1. Apache
2. PHP 7.3
3. redis

框架我选择的ThinkPHP5.1 不过这次我主要还是选择贴近原生的写法


选择apache的原因很简单。自带压力测试工具ab。符合我们的需要。虽然我们知道nginx来做web服务器性能更好。
php7.* 这个不用多介绍了PHP 7 和 PHP 5的性能不是一个世界的
redis 虽然可以实现秒杀的方式有很多。redis算是非常常见的缓存和中间件工具了。在性能和上手难度上都是很不错的选择

####一.装载秒杀商品
我们先假设我们有300个人来抢30件商品。那么我们就在我们的商品库里面装载30件不同id的商品
秒杀商品一般都是定时添加的。所以我们需要一个定时任务控制器用cli模式执行
```php
class Crontab
{
      public function addGoods()
    {
        //设定商品数量
        $count=30;
        $listKey="2019_04_15_goods_list";
        //创建连接redis对象
        $redis = new \Redis();
        $redis->connect('127.0.0.1', 6379);
        for ($i=1;$i<=$count;$i++){
            //将商品id push到列表中
            $redis->rPush($listKey,$i);
        }
    }
}
```

然后当我们需要装载商品的时候我们使用php命令去执行下我们的方法
```linux
php /项目地址/public/index.php index/crontab/addgoods
```
用redis客户端查看下商品id是否放入成功
![查看商品id](https://upload-images.jianshu.io/upload_images/6016628-ed382be484364488.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



####二.秒杀商品
秒杀商品其实就是一个将集合中的商品id取出和用户id绑定的过程。只是这个过程进行的非常的快。那么我们将秒杀分为两步，如果秒杀成功，则记录下用户id和商品id 也就是所谓的秒杀订单。如果秒杀失败，我们则简单的记录一个秒杀失败的人数。来确定这次秒杀有多少有效用户参与。
```php
  public function kill()
    {
        //假装是用户的唯一标识
        $uuid=md5(uniqid('user').time());
        //创建连接redis对象
        $redis = new \Redis();
        $redis->connect('127.0.0.1', 6379);
        $listKey="2019_04_15_goods_list";
        $orderKey="2019_04_15_buy_order";
        $failUserNum="2019_04_15_fail_user_num";
        if ($goodsId=$redis->lPop($listKey)) {
            //秒杀成功
            //将幸运用户存在集合中
            $redis->hSet($orderKey,$goodsId,$uuid);
        }else{
            //秒杀失败
            //将失败用户计数
            $redis->incr($failUserNum);
        }
        echo "SUCCESS";
    }
```

####压力测试模拟秒杀
刚刚有提到会使用apache自带的ab做测试
小试牛刀 300并发 3000访问量
```linux
ab -c 300 -n 3000 http://shop.example.com/index.php/index/index/kill
```
**啥也不说就是干**
![运行结果](https://upload-images.jianshu.io/upload_images/6016628-dd3a7f0550be5ac9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


虽说还是比较慢，但是3000次请求，是全部命中没有死掉的用户。加上我本身docker性能没给到最大。加上只有单机节点。我对这个成绩还是比较满意的

下面来看看抢到商品的幸运用户
```linux
[root@2f7621a62356 bin]# redis-cli  
127.0.0.1:6379> HGETALL 2019_04_15_buy_order
```
![商品和 用户id的对应关系](https://upload-images.jianshu.io/upload_images/6016628-d1665b7634dbd30f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再看看秒杀失败的用户数量
![抢购失败次数](https://upload-images.jianshu.io/upload_images/6016628-52acc461a33bc8a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这时候的商品list已经空空如也了。


好了，今天简单做个秒杀，就介绍到这里。有时候思路比实现的方法更重要。今天我所介绍的主要是一个思路和redis的用法，现实中的秒杀肯定还有很多复杂的逻辑。我也是简单介绍下。如果有什么不对的地方欢迎大神指点。感谢


以上





