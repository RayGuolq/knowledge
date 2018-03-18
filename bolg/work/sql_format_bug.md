# sql语句中format()函数的坑

今天发下了bug，很深很深，这里就不刻意渲染故事情节了，直奔主题吧。

今天在调试一个接口发现http接口返回总是 404的状态码。（整个项目是用spring+mybatis架构成的）不可思议，request的path路径毫无疑问是正确的。怎么会发生这种事情呢？？？

经过我的种种调试，最终定位到了出错点，发现这个错误是 mybatis文件中的 sql产生的。下面我只说明是如何发生错误的。

- mybatis 的xml文件中的原始代码（某位同事写的），如下：

```
<select id="getOrderInfo" resultType="com.aishuikeji.ishuidi.mybatis.vo.OrderInfoVol">
SELECT
	so.OrderPrice price,
	so.UserId userId,
	wcl.coupon_money couponMoney,
	format(wcl.order_discount_amount+wcl.pay_discount_amount+wcl.coupon_money,2) discountAmount,
	wcl.original_price originalPrice,
	wcl.pay_discount_amount payDiscountAmount,
	wcl.wallet_amount walletAmount,
	format(wcl.original_price - wcl.order_discount_amount-wcl.coupon_money - wcl.pay_discount_amount,2) orderPrice,
	format(wcl.order_discount_amount+wcl.coupon_money+wcl.pay_discount_amount,2) orderDiscountAmount,
	so.OrderStatus orderStatus
FROM
	shopping_order so
LEFT JOIN waterfee_couponuse_log wcl ON so.OrderId = wcl.order_id
where so.OrderId = #{orderId}
and so.UserId = #{userId}
</select>
```

- 查询结果返回值resultType指向一个结果类

```
public class OrderInfoVol implements Serializable {
    private float price;
    private long userId;
    private float couponMoney;
    private float discountAmount;
    private float originalPrice;
    private float payDiscountAmount;
    private float walletAmount;
    private int orderStatus;
    private float orderPrice;
    private float orderDiscountAmount;
    ......//getter setter方法我就都省略了
}
```
- OK，代码展示完毕，这里边可是有个很深的bug哦（奸笑表情），眼尖的同学看到了里边有个 format函数了，大眼一看并没啥问题，就是将加减运算后的结果保留两位小数的意思嘛，好，我们看下面我的实验：

```
mysql> select format(12.411, 2) price;
+-------+
| price |
+-------+
| 12.41 |
+-------+
1 row in set

mysql> select format(1212.411, 2) price;
+----------+
| price    |
+----------+
| 1,212.41 |
+----------+
1 row in set
```
- 注意到了吧，price整数位超过4位的话，中间就会多了一个 **逗号**，这完全是 转成string格式了嘛，好了这下总该明白了：

> （1）price 在数额比较小的时候可以成功转成float格式，一旦有上千的数额出现就直接错误了；
> （2）另外发生404是因为没办法直接转成float格式时，直接抛出了错误，阻断了当前的线程，所以此次请求没有返回数据就产生了404
> （3）而mybatis又没有将这个错误打印出来是最糟糕的

## 将数据转成2位小数，正常有两种方式：

1. 使用 round() 函数，如 round(@num,2)  参数 2 表示 保留两位有效数字。
2. 更好的方法是使用 convert(@num,decimal(18,2)) 实现转换，decimal(18,2) 指定要保留的有效数字。

> 这两个方法有一点不同：使用 round() 函数，如果 @num 是常数，如 round(2.3344,2) 则会在把有效数字后面的 变为0 ，成 2.3300。但 convert() 函数就不会。



