### 1，此项目为练习SQL和简单的数据分析所作的较为粗糙的分析项目，请浏览者批评指正，数据可视化内容也在完善当中，包含数据可视化图表以及分析的内容在飞书当中存储 
### 2，用python处理的版本还在完善当中 
### 3，数据来源：https://tianchi.aliyun.com/dataset/649 
### 4，项目背景:使用淘宝双十二前夕1亿+条用户行为数据，对用户行为习惯、用户行为转化、类目偏好以及用户价值进行分析。 
### 5，项目内容:使用SQL结合tablcau进行数据处理，分析和数据可视化呈现。利用SQL首先进行数据的清洗(去空、去重、去异常);分析用户时间维度的行为习惯;刻画用户行为路径，利用漏斗分析进行用户转化率分析;结合RFM模型进行用户分层和用户的价值分析;并对商品类目进行偏好分析。 
### 6，附加tableau图表的markdown版本在飞书云文档当中，连接：https://ycnsy2lcbs3o.feishu.cn/wiki/NmsRwWGKHivctMkQZK3c3JNsnWf?from=from_copylink

# 1，数据预处理

## 1.1检查空值

```sql
select * from userbehavior where user_id is null;
select * from userbehavior where item_id is null;
select * from userbehavior where category_id is null;
select * from userbehavior where behavior_type is null;
select * from userbehavior where timestamps is null;
```
##### 在后面分析过程中，发现存在时间值-timestamps小于0的情况，前面处理脏数据没有清理干净，在此补充脏数据的清理
```sql
delete userbehavior 
from userbehavior
where timestamps<0
```

## 1.2去重

### 1.2.1检查重复值：
分组后每组多于1项，就表示该项是重复值，以user_id,item_id,timestamps三列识别重复值

```sql
select user_id,item_id,timestamps from userbehavior
group by user_id, item_id, timestamps
having count(*)>1;
```

### 1.2.2第一列建立从1开始的主键，并命名为id

```sql
alter table userbehavior add id int first;
select * from userbehavior limit 5;
alter table userbehavior modify id int primary key auto_increment;
```
### 1.2.3去重：
保留userbehavior表内重复值id更小的该行，利用userbehavior.id>u2.id仅去除重复重复值（保证相同两行有一行被保留）

#### 法1：
```sql
with u2 as 
(
        select user_id,item_id,timestamps,min(id) as id from userbehavior
        group by user_id,item_id,timestamps
        having count(*)>1
        )

delete userbehavior from userbehavior
inner join u2 
        on  userbehavior.user_id=u2.user_id
        and userbehavior.item_id=u2.item_id
        and userbehavior.timestamps=u2.timestamps
where userbehavior.id>u2.id;
```

#### 法二
```sql
delete userbehavior from
userbehavior
(
        select user_id,item_id,timestamps,min(id) id from userbehavior
        group by user_id,item_id,timestamps
        having count(*)>1
        )u2
        where userbehavior.user_id=u2.user_id
        and userbehavior.item_id=u2.item_id
        and userbehavior.timestamps=u2.timestamps
        and userbehavior.id>u2.id;
```

## 1.3增加日期描述列
##### 更改buffer值；查看缓冲值现在大小，更改缓冲值的大小
```sql
show VARIABLES like '%_buffer%';
set global innodb_buffer_pool_size=10700000000;
```

##### datetimes修改：
```sql
alter table userbehavior add datetimes timestamp(0);
update userbehavior set datetimes=FROM_UNIXTIME(timestamps);
```

##### date与times：
```sql
alter table userbehavior add dates char(10);
alter table userbehavior add times char(8);
alter table userbehavior add hours char(2);
update userbehavior set dates=substring(datetimes,1,10);
update userbehavior set times=substring(datetimes,12,8);
update userbehavior set hours=substring(datetimes,12,2);
select * from userbehavior limit 5;
```

## 1.4去异常

```sql
select max(datetimes),min(datetimes) from userbehavior;
delete from userbehavior
where datetimes<'2017-11-25 00:00:00'
or datetimes>'2017-12-03 23:59:59';
```

## 1.5数据概览
```sql
desc userbehavior;
select * from userbehavior limit 5;
select count(1) from userbehavior;###目前仍有超1亿条数据--100095495
```
<img width="1011" height="147" alt="image" src="https://github.com/user-attachments/assets/e1f7eb1b-104a-4f58-a85d-8241b48d6050" />

# 2，查看获客情况

## 2.1创建临时表
接下来的每一个环节都将经过临时表的预测试，确保结果没有问题再进行后续处理，以提高处理效率，降低处理错误等问题（文档很少包含预测试过程）
```sql
CREATE TABLE temp_behavior LIKE userbehavior;
INSERT INTO temp_behavior
SELECT * FROM userbehavior LIMIT 100000;
### 查看临时表的导入情况
SELECT * FROM temp_behavior;
```

## 2.2获客情况查询

### 2.2.1创建新表评估浏览量，访客数量和浏览深度
计算页面浏览量 page view;独立访客数量 unique visitor;浏览深度 pv/uv

```sql
create table pv_uv_puv (
dates char(10)
, pv int(9)
, uv int(9)
,puv decimal(10,1)
);

INSERT INTO pv_uv_puv
SELECT dates
,COUNT(*) 'pv'
,COUNT(DISTINCT(user_id)) 'uv'
,ROUND(COUNT(*)/COUNT(DISTINCT(user_id)),2) 'puv'
FROM userbehavior
WHERE behavior_type='pv'
GROUP BY dates;
```

# 3、行为分析
## 3.1时间序列分析
### 3.1.1建立按小时分类的用户行为表

```sql
create table date_hour_behavior(
dates char(10)
,hours char(2)
,pv int 
,cart int 
,fav int
,buy int 
);

insert into date_hour_behavior
select dates
,hours
,count(if(behavior_type='pv',behavior_type,null)) as '浏览'
,count(if(behavior_type='cart',behavior_type,null)) as '加购物车'
,count(if(behavior_type='fav',behavior_type,null)) as '收藏'
,count(if(behavior_type='buy',behavior_type,null)) as '购买'
from userbehavior -- 更改为原表数据
group by dates,hours
order by dates,hours
```

### 按日统计用户行为
<img width="1008" height="807" alt="image" src="https://github.com/user-attachments/assets/04f19d70-a9d3-4d01-98d3-9772b7924b91" />
#### 图表基本情况说明：
柱状图：用户浏览量（pv）；折线1：加入购物车（cart）；折线2：收藏（fav）；折线2：购买（buy）
#### 2017/11/25-2017/12/03期间淘宝用户日行为：
11/26-12/01为工作日，用户行为整体呈现向上缓慢攀升趋势。12/02-12/03用户活跃度大幅上升，即周末用户活跃度显著高于工作日，且12月这两个周六日显著高于上一周周六日用户活跃度情况。
#### 分析
12月用户行为均显著增加，可能是由于在为双十二做预热，外部投放/大促/站内推送/首页资源位变更或渠道促销（付费流量或活动流量）导致。

### 3.1.3按小时统计用户行为
<img width="1005" height="798" alt="image" src="https://github.com/user-attachments/assets/eb8601bf-75fa-4cc5-8970-60d9a176b599" />
#### 图表基本情况说明：
横坐标更改为小时，分小时统计用户行为
#### 2017/11/25-2017/12/03期间淘宝用户时点行为：
淘宝为国民级APP，其使用情况可以反应民众生活习惯。用户活跃度显著增加时间点为20-22点，其中21点达到活跃高峰，用户购买量，浏览量等行为均显著增加。
#### 分析
根据此图可以确定商家应该在合适开启活动，才能达到最佳浏览量和用户加购量。以及在一天当中应当如何分配店铺投流情况。

### 3.1.4分层结构（date&hour）
<img width="1077" height="699" alt="image" src="https://github.com/user-attachments/assets/06979556-c8d0-48f6-adca-8cfeb33ceab3" />
#### 图表基本情况说明：
每天每小时的用户行为数据
#### 2017/11/25-2017/12/03期间淘宝用户每天每小时行为：
用户行为分布呈现周期性，趋势变化情况相似，12/02-12/03 峰值明显放大
### 3.1.5用户购买率与加购率
<img width="1260" height="879" alt="image" src="https://github.com/user-attachments/assets/17950955-4525-4c76-a999-17cf171e2a77" />
<img width="1293" height="792" alt="image" src="https://github.com/user-attachments/assets/b68ae343-afda-4b4f-be3d-bbf595c81dd9" />
#### 图表基本情况说明：
横坐标：24小时；图一：用户购买率（buy/pv）；图二：用户购买率与用户加购率（（cart+fav）/pv）
#### 2017/11/25-2017/12/03期间淘宝日购买率变化：
购买率最高峰时间非上面显示的用户活跃度最高峰时期20-22点，而是中午10点；用户加购率峰值出现在20-22点，与购买率呈现明显的变化趋势。
#### 分析：
购买率与加购率峰值与变化趋势呈现明显的差异的原因可能是由于用户存在决策延迟，即转化滞后。用户在夜间发生高频率的加购行为，在第二天经过比较后及逆行购买决策，产生行为时间差异。

## 3.2用户转化率分析

### 3.2.1各类行为下有多少用户数量，并粗算购买率
##### 此时的购买率当中包含未浏览或加购就购买的用户，这些buy的产生可能源自于，统计日期之前就已经完成决策的用户，分析时应当予以剔除。所以此处的购买率可能会偏大。

#### 法一：横向分列显示数量：各种用户行为下有多少去重用户（更便于通过SQL语句实现后续调用需求）

```sql
create table behavior_user_num2
(pv_hot int 
,cart_hot int 
,fav_hot int 
,buy_hot int );

insert into behavior_user_num2
select 
count(pv_hot)
,count(cart_hot)
,count(fav_hot)
,count(buy_hot)
from (
    select distinct user_id
    ,behavior_type
    ,(case 
     when behavior_type='pv' then 1
      else null 
    end) as 'pv_hot'
    ,(case 
      when behavior_type='cart' then 1
      else null 
    end) as 'cart_hot'
    ,(case 
      when behavior_type='fav' then 1
      else null 
    end) as 'fav_hot'
    ,(case 
      when behavior_type='buy' then 1
      else null 
    end) as 'buy_hot'
    from userbehavior
) as hot 
```
<img width="918" height="180" alt="image" src="https://github.com/user-attachments/assets/93f8aeb3-eed4-4e2f-ac7e-df3f0db943ef" />


#### 法二：

```sql
create table behavior_user_num
(
  behavior_type varchar(5)
  ,user_num int 
  );
```
<img width="975" height="300" alt="image" src="https://github.com/user-attachments/assets/7c009a9e-305f-420e-a5ea-a208c6b3cbbb" />

```sql
insert into behavior_user_num
select behavior_type
,count(distinct user_id) as user_num
from userbehavior
group by behavior_type
order by behavior_type desc; -- 购买率=672404/984105
```

### 3.2.2以各类行为总量计算购买率
#### 法一：横向分列（每一行为为一列）显示数量

```sql
create table behavior_num2
(pv int 
,cart int 
,fav int 
,buy int );

insert into behavior_num2
select 
count(pv_hot)
,count(cart_hot)
,count(fav_hot)
,count(buy_hot)
from (
    select user_id
    ,behavior_type
    ,(case 
     when behavior_type='pv' then 1
      else null 
    end) as 'pv_hot'
    ,(case 
      when behavior_type='cart' then 1
      else null 
    end) as 'cart_hot'
    ,(case 
      when behavior_type='fav' then 1
      else null 
    end) as 'fav_hot'
    ,(case 
      when behavior_type='buy' then 1
      else null 
    end) as 'buy_hot'
    from userbehavior 
        ) as hot 
```
<img width="1044" height="138" alt="image" src="https://github.com/user-attachments/assets/15b522b6-7cc8-4010-8f06-f6b7e945a5ef" />


```sql
select buy/pv 
from behavior_num2 -- 购买率=0.0225
```

#### 法二：纵向分行为类别显示行为总数

```sql
create table behavior_num 
(
	behavior varchar(5)
	,behavior_num int
	)
	
insert into behavior_num
select behavior_type
,count(*) as behavior_num
from userbehavior
group by behavior_type
order by behavior_type desc;
```

### 3.2.3 用户购买率&加购率总览
#### 购买率=购买行为总量/浏览总量--0.0225
#### 加购率=（加购物车+收藏）/浏览总量--0.0939
#### 存在问题：基于此数据无法进行漏斗分析，数据当中计算得到的购买user未必全部来自加购行为的user，尚存在直接购买而没有使用加购行为的用户
<img width="1986" height="1194" alt="image" src="https://github.com/user-attachments/assets/5e597d44-ec60-4137-b555-da00fee3ba12" />

## 3.3用户行为路径

### 3.3.1对用户行为按照商品进行分组

```sql
create view userbehavior_view as 
select 
  user_id
  ,item_id
  ,count(if(behavior_type='pv',behavior_type,null)) as 'pv'
  ,count(if(behavior_type='fav',behavior_type,null)) as 'fav'
  ,count(if(behavior_type='cart',behavior_type,null)) as 'cart'
  ,count(if(behavior_type='buy',behavior_type,null)) as 'buy'
from userbehavior
group by user_id,item_id
```
### 3.3.2刻画行为路径：
#### 用户行为标准化
```sql
create view userbehavior_standard as 
select  user_id,item_id
,(case when pv>0 then 1 else 0 end) as '浏览次数'
,(case when fav>0 then 1 else 0 end) as '已收藏'
,(case when cart>0 then 1 else 0 end) as '已加购'
,(case when buy>0 then 1 else 0 end) as '已购买'
from userbehavior_view
```
<img width="1893" height="1134" alt="image" src="https://github.com/user-attachments/assets/7cafb2d8-773d-4f1d-9c1c-e7db85ed9761" />



#### 形成路径类型：

```sql
create view userbehavior_path as 
select *,
concat(浏览次数,已收藏,已加购,已购买) as '购买路径类型'
from userbehavior_standard as a
where a.已购买>=1 

-- 统计各类购买行为数量
create view path_count as 
select 购买路径类型
,count(*) as '数量'
from userbehavior_path
group by 购买路径类型
order by 数量 desc
```
#### 提高可读性

```sql
create table renhua(
path_type char(4)
,description varchar(40));

insert into renhua 
values('0001','直接购买了')
,('1001','浏览后购买了')
,('0011','加购后购买了')
,('1011','浏览加购后购买了')
,('0101','收藏后购买了')
,('1101','浏览收藏后购买了')
,('0111','收藏加购后购买了')
,('1111','浏览收藏加购后购买了')
```
<img width="402" height="387" alt="image" src="https://github.com/user-attachments/assets/5fc71389-cce9-4a28-aae8-a94750f96dff" />


#### 保存为新表：

```sql
create table path_result(
path_type char(4)
,description varchar(40)
,num int);

insert into path_result
select renhua.path_type
,renhua.description
,path_count.数量
from path_count
join renhua on renhua.path_type=path_count.`购买路径类型`
order by 数量 desc
```
<img width="1521" height="612" alt="image" src="https://github.com/user-attachments/assets/c5508057-126f-449b-8fb3-01f4c25f6ae5" />


#### 问题说明：
##### 1）0001型本来不应该存在，但是可能用户在此之前已经加购，或浏览时间不在统计范围之内，统计时间内仅涵盖了“购买”这一动作
##### 2）最后一段代码运行时间很长，而前面时间很短，原因在于之前create view仅存储逻辑，并未真正运行

## 3.4漏斗分析
##### 说明：用户有效购买路径 ①pv→buy；②pv→cart/fav→buy。但是漏斗分析路径一般为 pv→cart/fav→buy，但是不可以忽视直接购买（跳过fav/cart）的这部分用户，所以应剔除“直接购买”的这部分用户，构建更加精准的漏斗图，据此所得到的收藏加购比率、收藏加购后购买比率更加精准

### 3.4.1计算购买率：buy剔除未经历cart&fav的数量

```sql
select sum(buy)
from userbehavior_view_table
where buy>0 and fav=0 and cart=0

select buy-1528016 ## 487791
from behavior_num2

select 487791/(2888255+5530446)
```

##### 剔除未历经“加入购物车”和“收藏”的直接购买的buy数量
##### 说明：剔除的buy可能源自非统计期间的加购行为，即已经经历过决策阶段直接进行购买行为的情况 也可能是 单纯浏览后/未浏览 直接进行购买行为的情况

### 3.4.2计算加购率：（cart+fav）/pv=0.0939

# 4，用户价值分析

## 4.1RFM模型
##### 情况说明：数据缺少金额项，所以根据F和R将用户分为四类：价值用户（Ⅰ象限），保持用户（Ⅱ象限），挽留用户（Ⅲ象限），发展用户（Ⅳ象限）

### 4.1.1最近购买时间&次数

```sql
select user_id 
,max(dates) as 最近购买时间
,count(user_id) as 购买次数
from temp_behavior
where behavior_type='buy'
group by user_id
order by 最近购买时间 desc;

create table rfm_model(
user_id int 
,frequency int 
,recent char(10)
)

insert into rfm_model
select user_id 
,count(user_id) 购买次数
,max(dates) as 最近购买时间
from userbehavior
where behavior_type='buy'
group by user_id
order by 2 desc, 3 desc;
```

### 4.1.2用户分层

#### 打分

```sql
alter table rfm_model add column fscore int;

update rfm_model
set fscore=(
  case 
    when frequency>100 then 5
    when frequency between 50 and 99 then 4
    when frequency between 20 and 49 then 3
    when frequency between 5 and 19 then 2
    else 1
  end)

alter table rfm_model add column rscore int;

update rfm_model 
set rscore = (
	case 
		when recent='2017-12-03' then 5
		when recent in ('2017-12-01','2017-12-02') then 4
		when recent in ('2017-11-29','2017-11-30') then 3
		when recent in ('2017-11-27','2017-11-28') then 2
		else 1
	end ) 
```

#### 分层


```sql
set @f_avg=null;
set @r_avg=null;
select avg(fscore) into @f_avg from rfm_model;
select avg(rscore) into @r_avg from rfm_model;

alter table rfm_model add column class varchar(40);

update rfm_model 
set class=(
	case 
		when fscore>@f_avg and rscore>@r_avg then '价值用户'
		when fscore>@f_avg and rscore<@r_avg then '保持客户'
		when fscore<@f_avg and rscore>@r_avg then '发展客户'
		when fscore<@f_avg and rscore<@r_avg then '挽留用户'
	end )
```
<img width="1620" height="1056" alt="image" src="https://github.com/user-attachments/assets/903e3ae8-d60b-4d46-bd06-a924ff0fb724" />


#### 统计各分区总用户数

```sql
select class,count(user_id) from rfm_model 
group by class;
```
<img width="1269" height="246" alt="image" src="https://github.com/user-attachments/assets/e140f377-a8a2-42f0-9344-c8db9da904a6" />
<img width="1062" height="795" alt="image" src="https://github.com/user-attachments/assets/d76182d8-5112-4a4d-bb2d-74491bb86a58" />


### 4.1.3RFM分析

#### 1）情况分析：价值用户14.5%；发展用户占比42.7%；保持用户占比3.6%；挽留用户占比38.8%。总体来看，发展用户和挽留用户占81.5%，说明用户整体活跃度偏低。保持用户仅占3.6%，说明高频用户活跃度维持较为困难。
#### 2）针对性策略：
##### ① 价值用户：高活跃高忠诚，可以提供专属权益和优先体验功能，鼓励其推荐信用户来进行维护，也建议通过定期的调研需求来提升满意度。
##### ②发展用户：近期有互动，但是频率较低，可能还没有养成高频习惯。通过个性推荐提高购买率，也可以设置连续购买奖励，培养用户习惯

# 5.商品分析

## 5.1商品分类-按照热度划分

### 5.1.1商品浏览热度
#### 品类维度-提取前十的热门品类

```sql
select category_id, count(behavior_type) as '品类浏览量'
from temp_behavior
where behavior_type='pv'
group by category_id
order by 2 desc 
limit 10 ## 提取出前十的热门品类

select category_id
,count(if(behavior_type='pv',behavior_type,null))'品类浏览量'
from temp_behavior
group by category_id
order by 品类浏览量 desc
limit 10 ## 提取前十的热门品类
```
<img width="1515" height="1050" alt="image" src="https://github.com/user-attachments/assets/f53e30be-1308-4aa7-8839-2421f5fb5ab7" />


#### 商品维度-提取前十的热门商品

```sql
select item_id, count(behavior_type) as '品类浏览量'
from userbehavior
where behavior_type='pv'
group by item_id
order by 2 desc 
limit 100 ## 提取前100热门商品

select item_id
,count(if(behavior_type='pv',behavior_type,null))'品类浏览量'
from userbehavior
group by item_id
order by 品类浏览量 desc
limit 10 ## 提取前100热门商品
```
<img width="2058" height="1068" alt="image" src="https://github.com/user-attachments/assets/5edf6fce-eed2-4e99-825f-81c4b1ee368c" />


#### 热门品类当中最热门商品

```sql
select category_id
, item_id
,count(if(behavior_type='pv',behavior_type,null))'商品浏览量'
from userbehavior
group by category_id, item_id
order by 3 desc
limit 100
```

##### 微调：求每个品类下浏览量最高的商品，并且视商品浏览量为热度情况进行排序

##### 法一：（相较于法二而言更节省算力）

```sql
select category_id
,item_id
, 商品浏览量
from 
(
    select category_id
    ,item_id
    ,count(behavior_type) as '商品浏览量'
    ,row_number() over (partition by category_id order by count(behavior_type) desc) as ranked 
    from userbehavior
    where behavior_type='pv'
    group by category_id,item_id
) as rank_category_item
where ranked=1
order by 3 desc 
limit 100
```
<img width="1578" height="969" alt="image" src="https://github.com/user-attachments/assets/0f8fbcd7-649b-47d7-9f6f-4dc43e4d8f84" />


##### 法二：

```sql
insert into popular_cateitem
select category_id,item_id,品类商品浏览量
from(
select category_id
, item_id
,count(if(behavior_type='pv',behavior_type,null)) as '品类商品浏览量'
,rank() over(partition by category_id order by count(if(behavior_type='pv',behavior_type,null)) desc) as ranked
from userbehavior
group by category_id, item_id
) as a
where ranked=1
order by 品类商品浏览量 desc
limit 10;
```

#### 存储

```sql
create table popular_category(
category_id int 
,pv int);
create table popular_item(
item_id int 
,pv int);
create table popular_cateitem(
category_id int 
,item_id int
,pv int);

insert into popular_category
select category_id, count(behavior_type) as '品类浏览量'
from userbehavior
where behavior_type='pv'
group by category_id
order by 2 desc 
limit 10;

insert into popular_item
select item_id, count(behavior_type) as '商品浏览量'
from userbehavior
where behavior_type='pv'
group by item_id
order by 2 desc 
limit 100 ;

insert into popular_cateitem2
select category_id
,item_id
, 品类商品浏览量
from 
(
select category_id
,item_id
,count(behavior_type) as '品类商品浏览量'
,row_number() over (partition by category_id order by count(behavior_type) desc) as ranked 
from userbehavior
where behavior_type='pv'
group by category_id,item_id
) as a
where ranked=1
order by 3 desc 
limit 100;
```

### 5.1.2商品转化率分析

#### 商品转化率
##### 说明：该商品吸引到用户占总用户的比例→购买用户/总的该商品浏览用户

```sql
create table item_detail(
item_id int,
pv int,
fav int,
cart int,
buy int,
user_buy_rate float 
)

insert into item_detail
select item_id
  ,count(if(behavior_type='pv',behavior_type,null)) as 'pv'
  ,count(if(behavior_type='fav',behavior_type,null)) as 'fav'
  ,count(if(behavior_type='cart',behavior_type,null)) as 'cart'
  ,count(if(behavior_type='buy',behavior_type,null)) as 'buy'
  ,count(distinct if(behavior_type='buy',user_id,null))/count(distinct user_id) as '商品转化率'
from userbehavior
group by item_id
order by 商品转化率 desc 
```
<img width="1599" height="885" alt="image" src="https://github.com/user-attachments/assets/8fe34373-c6d0-4a31-a95c-4a1b251a50d9" />


#### 品类转化率

```sql
create table category_detail(
category_id int,
pv int,
fav int,
cart int,
buy int,
user_buy_rate float 
)

insert into category_detail
select category_id
  ,count(if(behavior_type='pv',behavior_type,null)) as 'pv'
  ,count(if(behavior_type='fav',behavior_type,null)) as 'fav'
  ,count(if(behavior_type='cart',behavior_type,null)) as 'cart'
  ,count(if(behavior_type='buy',behavior_type,null)) as 'buy'
  ,count(distinct if(behavior_type='buy',user_id,null))/count(distinct user_id) as '品类转化率'
from userbehavior
group by category_id
order by 品类转化率 desc 
```
<img width="1206" height="1128" alt="image" src="https://github.com/user-attachments/assets/c699843f-e249-4050-b080-5b82f537d916" />

### 5.1.3 散点矩阵分析/四象分析
基于购买量和浏览量用于评估产品性能
#### 说明1：以buy和pv两个维度设置均值回归线，将产品分四个象限。每个象限代表不同蟾皮特征的用户行为模式
#### 说明2：同时引入去实现，分析购买量和点击量之间是否有正/负相关关系
<img width="1236" height="915" alt="image" src="https://github.com/user-attachments/assets/e7cad585-175f-4094-876c-24838989181f" />
#### ①第一象限-点击数高，购买数高：
产品与用户行为特征：此类产品应为刚需产品，且品类多，种类丰富，用户在较高需求下也有非常多的选择
策略：加大推广和库存，满足市场需要
#### ②第二象限-小技术低，购买数高：
产品与用户行为特征：用户购买果断，且购买量大，可能用户选择性小，或者几个品牌存在垄断现象，或者产品差异性小，用户不愿意花太多时间进行挑选
策略：提高曝光，分析垄断问题，考虑扩大品类
#### ③第三象限-点击数少，购买数少：
产品与用户行为特征：产品可能存在替代品，用户很难集中在某个子类大量购买，而是跳跃式购买
策略：考虑淘汰一定产品或者捆绑销售
#### ④第四象限-点击数高，购买数低：
产品与用户行为特征：此类产品需求弹性较大，用户购买存在随机性
策略：优化转化路径，降低用户决策成本
