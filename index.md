## 商城购物车结算

结算第一步将购物车的物品进行结算加入订单表(shop_order)，将商品详情加入订单详情表(shop_order_detail),并清空购物车，如果没有地址则添加地址，等待下一步支付.

### Cart购物车表结构

```
CREATE TABLE `shop_cart` (
  `cartid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `productid` bigint(20) unsigned NOT NULL,
  `price` decimal(10,2) unsigned NOT NULL,
  `productnum` bigint(20) unsigned NOT NULL,
  `userid` bigint(20) unsigned NOT NULL,
  `createtime` int(10) unsigned NOT NULL,
  PRIMARY KEY (`cartid`),
  KEY `productid` (`productid`) USING BTREE,
  KEY `userid` (`userid`) USING BTREE
) ENGINE=MyISAM AUTO_INCREMENT=8 DEFAULT CHARSET=utf8;

```

### Order订单表结构
```
CREATE TABLE `shop_order` (
  `orderid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `userid` bigint(20) unsigned NOT NULL DEFAULT '0',
  `addressid` bigint(20) unsigned NOT NULL DEFAULT '0',
  `amount` decimal(10,2) unsigned NOT NULL DEFAULT '0.00',
  `status` int(10) unsigned NOT NULL,
  `expressid` int(10) unsigned NOT NULL DEFAULT '0',
  `expressno` varchar(50) NOT NULL DEFAULT '',
  `createtime` int(10) unsigned NOT NULL DEFAULT '0',
  `updatetime` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`orderid`),
  KEY `userid` (`userid`) USING BTREE,
  KEY `expressid` (`expressid`) USING BTREE,
  KEY `expressno` (`expressno`) USING BTREE
) ENGINE=MyISAM AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

```
### OrderDetail订单详情表
```
CREATE TABLE `shop_order_detail` (
  `detailid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `orderid` bigint(20) unsigned NOT NULL DEFAULT '0',
  `productid` bigint(20) NOT NULL DEFAULT '0',
  `productnum` bigint(20) unsigned NOT NULL DEFAULT '0',
  `price` decimal(10,2) NOT NULL DEFAULT '0.00',
  `createtime` int(10) unsigned DEFAULT '0',
  PRIMARY KEY (`detailid`),
  KEY `orderid` (`orderid`) USING BTREE,
  KEY `productid` (`productid`) USING BTREE
) ENGINE=MyISAM AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

```
### Address用户地址表结构
```
CREATE TABLE `shop_address` (
  `addressid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `firstname` varchar(32) NOT NULL DEFAULT '',
  `lastname` varchar(32) NOT NULL,
  `address` text NOT NULL,
  `company` varchar(100) NOT NULL DEFAULT '',
  `postcode` char(6) NOT NULL DEFAULT '',
  `telephone` varchar(20) NOT NULL,
  `email` varchar(100) NOT NULL DEFAULT '',
  `userid` bigint(20) unsigned NOT NULL,
  `careatetime` int(10) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`addressid`),
  KEY `userid` (`userid`)
) ENGINE=MyISAM AUTO_INCREMENT=3 DEFAULT CHARSET=utf8;

```

### 去计算逻辑代码:

结算就是将购物车的记录添加到order表,采用事物进行操作，避免数据出现不同步的问题，下单成功的同时减少库存并清空购物车。

```
//结算第一步
    public function actionAdd(){
        // 判断用户是否登录
        if(Yii::$app->session['user']['isLogin'] != 1) {
            return $this->redirect(['member/auth']);
        }

        // 开启事物
        $trasaction = Yii::$app->db->beginTransaction();
        try {
            if(Yii::$app->request->isPost) {
                // 接收数据
                $post = Yii::$app->request->post();
                // 根据登录名称查询userid
                $loginName = Yii::$app->session['user']['loginName'];
                $usermodel = User::find()->where('username=:name or useremail=:email', [
                    ':name' => $loginName,
                    ':email' => $loginName
                ])->one();
                if (!$usermodel) {
                    // 抛出异常
                    throw new Exception();
                }
                $userid = $usermodel->userid;
                // 订单入库
                $order = new Order();
                $order->userid = $userid;
                $order->status = Order::CREATEORDER;
                $order->createtime = time();
                $order->save();
                if (!$order->save()) {
                    throw new Exception();
                }

                // 订单详情入库
                $orderid = $order->getPrimaryKey();
                foreach ($post['OrderDetail']  as $product) {
                    $orderdetail = new OrderDetail();
                    $product['orderid'] = $orderid;
                    $product['createtime'] = time();
                    $data['OrderDetail'] = $product;
                    if($orderdetail->load($data) && $orderdetail->save()){
                    }

                    // 清空购物车
                    Cart::deleteAll('productid=:pid',[':pid'=>$product['productid']]);
                    // 减少库存
                    Product::updateAllCounters(['num'=> -$product['productnum']],'productid=:pid',[':pid'=>$product['productid']]);
                }
            }
            $trasaction->commit();

        }catch (Exception $e){
            $trasaction->rollBack();
        }
        // 跳转到确认页
        return $this->redirect(['order/check','orderid'=>$orderid]);
    }
