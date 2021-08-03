题目: 基于电商交易场景（用户、商品、订单），设计一套简单的表结构，提交 DDL 的 SQL 文件到 Github（后面 2 周的作业依然要是用到这个表结构）。

作答: 

1). `t_user` 用户信息

```SQL
CREATE TABLE `t_user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_nick` varchar(20) DEFAULT NULL COMMENT '用户名',
  `mobile` varchar(16) DEFAULT NULL COMMENT '联系方式',
  `icon` varchar(256) DEFAULT NULL COMMENT '用户头像',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

2). `t_user_address` 用户收货地址

```sql
CREATE TABLE `t_address` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) DEFAULT NULL COMMENT '用户id',
  `address_id` varchar(256) DEFAULT NULL COMMENT '省市区对应的区域ID',
  `address_detail` varchar(256) DEFAULT NULL COMMENT '街道相关收货地址详细信息',
  `user_name` varchar(200) DEFAULT NULL COMMENT '收货人',
  `mobile` varchar(16) DEFAULT NULL COMMENT '收货人联系方式',
  `status` TINYINT(4) DEFAULT NULL COMMENT '是否生效 0有效 1废弃',
  `idx` TINYINT(4) DEFAULT NULL COMMENT '排序方式，值越大越靠前',
  `default_address` TINYINT(4) DEFAULT NULL COMMENT '默认地址 0否 1是',  
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

3).`t_member`等级信息表

```sql
CREATE TABLE `t_member` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) DEFAULT NULL COMMENT '用户id',
  `level` TINYINT(4) DEFAULT NULL COMMENT '用户等级',
  `start_time`  datetime DEFAULT NULL COMMENT '开始时间',
  `end_time`  datetime DEFAULT NULL COMMENT '截止时间', 
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

4). `t_login` 登录相关表

```sql
CREATE TABLE `t_login` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) DEFAULT NULL COMMENT '用户id',
  `mobile` bigint(20) DEFAULT NULL COMMENT '用户手机号，用户手机号登录',
  `user_name` bigint(20) DEFAULT NULL COMMENT '用户名称',
  `pwd` bigint(20) DEFAULT NULL COMMENT '用户密码',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  UNIQUE KEY `idx_user_name` (`user_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

5). `t_order`订单表

```sql
CREATE TABLE `t_order` (
  `order_no` varchar(128) NOT NULL COMMENT '订单号',
  `user_id` bigint(20) DEFAULT NULL COMMENT '用户id',
  `order_create_time` datetime DEFAULT NULL COMMENT '订单创建时间',
  `pay_time` datetime DEFAULT NULL COMMENT '支付时间',
  `real_pay` int(8)  DEFAULT NULL COMMENT '实付总金额，扩大100倍',
  `status`  TINYINT(4) DEFAULT NULL COMMENT '状态：-1取消订单 0提交订单 1已经支付 2等待确认收货 3已收货 4退货',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`)
  UNIQUE KEY `idx_order_no` (`order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

6). `t_item` 商品表

```SQL
CREATE TABLE `t_item` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `title` varchar(128) NOT NULL COMMENT '商品标题',
  `shop_id` bigint(20) NOT NULL COMMENT '所属店铺ID',
  `price` int(8)  DEFAULT NULL COMMENT '价格，扩大100倍',
  `start_time`  datetime DEFAULT NULL COMMENT '开始时间',
  `end_time`  datetime DEFAULT NULL COMMENT '截止时间',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

7). `t_coupon` 券信息

```SQL
CREATE TABLE `t_item` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `coupon_name` varchar(128) NOT NULL COMMENT '券名称',
  `item_id` bigint(20) NOT NULL COMMENT '商品ID',
  `amount` int(8)  DEFAULT NULL COMMENT '券面额',
  `start_time`  datetime DEFAULT NULL COMMENT '开始时间',
  `end_time`  datetime DEFAULT NULL COMMENT '截止时间',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_item_id` (`item_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

8). `t_order_goods` 订单商品信息

```SQL
CREATE TABLE `t_order_goods` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `order_no` varchar(128) NOT NULL COMMENT '订单号',
  `item_id` bigint(20) NOT NULL COMMENT '商品ID',
  `real_pay` int(8)  DEFAULT NULL COMMENT '实付金额，扩大100倍',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `idx_order_no` (`order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

9). `t_address` 地址信息

```SQL
CREATE TABLE `t_address` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `provice` varchar(12) NOT NULL COMMENT '省',
  `city` varchar(12) NOT NULL COMMENT '市',
  `area` varchar(12)  DEFAULT NULL COMMENT '区',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

10). `t_shop` 店铺信息

```SQL
CREATE TABLE `t_shop` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `address_id` varchar(12) NOT NULL COMMENT '位置id',
  `address_detail` varchar(256) DEFAULT NULL COMMENT '街道相关地址详细信息',
  `name` varchar(12) NOT NULL COMMENT '店铺名称',
  `star` int(4)  DEFAULT NULL COMMENT '星级',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```


