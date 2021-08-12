# 02 | demo示例

## 一、示例业务场景

```sql
insert overwrite table view_udp_gmv
select
	a.category_name_level1 category_name_level1,
	a.category_name_level2 category_name_level2,
	a.category_name_level3 category_name_level3,
	count(distinct aa.real_product_id) r_cnt,
	count(distinct b.real_product_id) visit_r_cnt,
	count(distinct c.real_product_id) dx_r_cnt_1000,
	count(distinct c.real_product_id)/count(distinct aa.real_product_id) dx_rate,
	count(distinct d.real_product_id) dx_r_cnt_0,
	sum(order_cnt) order_cnt,
	sum(account_cnt) account_cnt,
	sum(GMV) GMV,
	sum(amount_order_cnt) amount_order_cnt,
	sum(amount_account_cnt) amount_account_cnt,
	sum(amount) amount
from (
	select 
  	category_name_level1, 
  	category_name_level2, 
  	category_name_level3, 
  	real_product_id, 
  	product_id
	from dwd_homedo.dwd_sub_product_info
	where category_name_level1 is not null
	group by 
  	category_name_level1,
  	category_name_level2,
  	category_name_level3,
  	real_product_id ,product_id
) a
left join (
	select distinct real_product_id
	from (
		select
  			distinct real_product_id
		from dwd_homedo.dwd_sub_product_info
		where first_sale_time< '$end_date_var$' and sales_status_id=1
		union all
		select
  			distinct b.real_product_id
		from dwd_homedo.dwd_sub_product_log a
		left join dwd_homedo.dwd_sub_product_info b
		on a.product_id=b.product_id
		where a.sales_status=2 and a.insert_time >= '$start_date_var$' and a.insert_time < '$end_date_var$'
	) p
) aa on a.real_product_id=aa.real_product_id
left join (
	select
 		distinct real_product_id
	from dws_homedo.dws_log_product_click_inner  aa
	where date_d >= '$start_date_var$' and date_d < '$end_date_var$'
) b on a.real_product_id=b.real_product_id
left join (
	select
 		real_product_id
	from dws_homedo.dws_core_orderitem_detail
	where is_return='否' and amount_date >= '$start_date_var$' and amount_date < '$end_date_var$'
	group by real_product_id
	having sum(amount) >= 1000
) c on a.real_product_id=c.real_product_id
left join (
	select real_product_id
	from dws_homedo.dws_core_orderitem_detail
	where is_return='否' and amount_date >= '$start_date_var$' and amount_date < '$end_date_var$'
	group by real_product_id
	having sum(amount)>0
) d on a.real_product_id=d.real_product_id
left join (
	select
		product_id,
		count(distinct order_id) order_cnt,
		count(distinct account_id) account_cnt,
		sum(order_item_share_amount_net) GMV
	from dws_homedo.dws_core_orderitem_detail
	where is_return='否' and order_date >= '$start_date_var$' and order_date < '$end_date_var$'
	group by product_id
) e on a.product_id=e.product_id
left join (
	select
 		product_id,
		count(distinct order_id) amount_order_cnt,
		count(distinct account_id) amount_account_cnt,
		sum(amount) amount
	from dws_homedo.dws_core_orderitem_detail
	where is_return='否' and amount_date >= '$start_date_var$' and amount_date < '$end_date_var$'
	group by product_id
) f on a.product_id=f.product_id
group by a.category_name_level1, a.category_name_level2, a.category_name_level3;
```

解析该 SQL 的测试代码:

```scala
val tableLineage: NebulaTableLineage = lineageParser
			.parseNebulaTableLineage(Seq(view_udp_gmv), SQLEngine.Doris)
JsonUtil.printJsonString(tableLineage)
TimeUnit.SECONDS.sleep(3)
```

测试结果 Json:

```json
{
	"edges":{-652237516:2074352856,1136754541:2074352856,-362477784:2074352856,-975206992:2074352856
	},
	"nodes":[
		{
			"db":{
				"id":-700883495,
				"name":"dws_homedo",
				"type":"Doris"
			},
			"metadata":null,
			"name":"dws_log_product_click_inner",
			"schema":null,
			"tid":1136754541
		},
		{
			"db":{
				"id":-1073234744,
				"name":"dwd_homedo",
				"type":"Doris"
			},
			"metadata":null,
			"name":"dwd_sub_product_info",
			"schema":null,
			"tid":-652237516
		},
		{
			"db":{
				"id":-700883495,
				"name":"dws_homedo",
				"type":"Doris"
			},
			"metadata":null,
			"name":"dws_core_orderitem_detail",
			"schema":null,
			"tid":-362477784
		},
		{
			"db":{
				"id":-1073234744,
				"name":"dwd_homedo",
				"type":"Doris"
			},
			"metadata":null,
			"name":"dwd_sub_product_log",
			"schema":null,
			"tid":-975206992
		},
		{
			"db":{
				"id":-697368687,
				"name":"default",
				"type":"Doris"
			},
			"metadata":null,
			"name":"view_udp_gmv",
			"schema":null,
			"tid":2074352856
		}
	]
}
```

该结果添加到格式化 csv 文件中

### 1.1 添加 node

```csv
1136754541,dws_homedo,dws_log_product_click_inner,Doris
-652237516,dwd_homedo,dwd_sub_product_info,Doris
-362477784,dws_homedo,dws_core_orderitem_detail,Doris
-975206992,dwd_homedo,dwd_sub_product_log,Doris
2074352856,default,view_udp_gmv,Doris
```

### 1.2 添加 edge

```csv
-652237516,2074352856
1136754541,2074352856
-362477784,2074352856
-975206992,2074352856
```



## 二、第一层 Hive 表

```
dws_homedo.dws_log_product_click_inner
dwd_homedo.dwd_sub_product_info
dws_homedo.dws_core_orderitem_detail
dwd_homedo.dwd_sub_product_log
```

### 2.1 dws_log_product_click_inner

```sql
insert into table dws_homedo.dws_log_product_click_inner
select
 	date_format(tt.inserttime, 'yyyy-MM-dd')date_d,
 	tt.productid, aa.product_name,
	tt.goodsid,
	aa.real_product_id,
	case when LOWER(tt.platform) like '%B2B%' or LOWER(tt.platform)=1 then 'PC'
		when LOWER(tt.platform) like '%msite%' or LOWER(tt.platform)=2  then 'M站'
	else 'APP' end channel,
	tt.promotionid,
 	cc.name as promotion_name,
 	tt.accountid, dd.account_name,
	count(*) cnt,
	from_unixtime(unix_timestamp(),'yyyy-MM-dd HH:mm:ss')
from
	(select * from bak_homedo_stat.t_product_browse
		where InsertTime <  current_date()  and mark >0
		and inserttime >= date_format(date_sub(current_date(),1),'yyyy-MM-dd')
		and d = date_format(date_sub(current_date(),1),'yyyyMMdd')
	)tt
left join dwd_homedo.dwd_sub_product_info aa on tt.productid = aa.product_id
left join (
		select
			id, name
		from bak_homedo.t_product_promotion
		where mark >0 and d =  date_format(date_sub(current_date(),1),'yyyyMMdd')
) cc on tt.promotionid = cc.id
left join dwd_homedo.dwd_sub_account_info dd on tt.accountid = dd.account_id
group by
	tt.productid,
 	aa.product_name,
  tt.goodsid,
  aa.real_product_id ,
	date_format(tt.inserttime, 'yyyy-MM-dd'),
 	case when LOWER(tt.platform) like '%B2B%' or LOWER(tt.platform)=1 then 'PC'
		when LOWER(tt.platform) like '%msite%' or LOWER(tt.platform)=2  then 'M站'
	else 'APP' end,
  tt.promotionid,
  cc.name,
	tt.accountid,
 	dd.account_name;
```

解析该 SQL 的测试代码:

```scala
val tableLineage: NebulaTableLineage = lineageParser
			.parseNebulaTableLineage(Seq(log_product_click_inner), SQLEngine.Hive)
JsonUtil.printJsonString(tableLineage)
TimeUnit.SECONDS.sleep(3)
```

测试结果 Json:

```json
{
	"edges":{-575373642:1136754541,-652237516:1136754541,1373807724:1136754541,550444400:1136754541
	},
	"nodes":[
		{
			"db":{
				"id":1238676905,
				"name":"dwd_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"dwd_sub_product_info",
			"schema":null,
			"tid":-652237516
		},
		{
			"db":{
				"id":2103754311,
				"name":"bak_homedo_stat",
				"type":"Hive"
			},
			"metadata":null,
			"name":"t_product_browse",
			"schema":null,
			"tid":550444400
		},
		{
			"db":{
				"id":2055766894,
				"name":"bak_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"t_product_promotion",
			"schema":null,
			"tid":1373807724
		},
		{
			"db":{
				"id":1238676905,
				"name":"dwd_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"dwd_sub_account_info",
			"schema":null,
			"tid":-575373642
		},
		{
			"db":{
				"id":1611028154,
				"name":"dws_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"dws_log_product_click_inner",
			"schema":null,
			"tid":1136754541
		}
	]
}
```

#### 2.1.1 添加 node

```csv
550444400,bak_homedo_stat,t_product_browse,Hive
1373807724,bak_homedo,t_product_promotion,Hive
-575373642,dwd_homedo,dwd_sub_account_info,Hive
```

#### 2.1.2 添加 edge

```csv
-575373642,1136754541
-652237516,1136754541
1373807724,1136754541
550444400,1136754541
```

### 2.2 dwd_sub_product_info

```sql
insert overwrite table dwd_homedo.dwd_sub_product_info
select
      tpp.product_id,-- 商品id
      tpp.real_product_id, -- 真实商品id
      tpp.spu_id,  -- 商品spu
      tpp.u8code,  -- u8商品编码
      tpp.product_name, -- 商品名称
      pc.category_id_level1, -- 一级商品分类
      pc.category_id_level2, -- 二级商品分类
      pc.category_id_level3, -- 三级商品分类
      pc.category_name_level1,  -- 一级商品分类
      pc.category_name_level2,  -- 二级商品分类
      pc.category_name_level3,  -- 三级商品分类
      tpp.user_group_id,  -- 用户组Id
      bu.usergroup_name user_group_name, -- 用户组
      tpp.price, -- 普通会员价(市场价)
      tpp.cost,  -- 认证会员价(批发价)
      tpp.price_mobile, -- 会员价移动端
      tpp.cost_mobile,   -- 批发价移动端
      purchase_price, -- 进价
      IsOEM is_oem,
      tpp.hebi_present_ratio,  -- 河币赠送比例默认为0 不赠送河币 根据vip等级赠送
      tpp.brand_id, -- 品牌
      tpp.brand_name, -- 品牌
      tpp.model,  -- 型号
      tpp.product_type_id, -- 商品类型(例如：渠道类、项目类、服务类)
      tpp.product_type_name, -- 商品类型(例如：渠道类、项目类、服务类)
      tpp.pack_unit, --  打包单位
      tpp.series_tag,  -- 商品所属系列
      tpp.sale_properties,  -- 销售属性
      tpp.product_property, -- ('普通货品') 货品属性，决定仓库存放(普通货品，贵重货品，虚拟货品)
      tpp.warehouse_id, -- 仓库Id
      tpp.is_sales, -- 是否销售
      tpp.sales_status_id, --  销售状态 0或空 未提交审核1 上架2 下架4 一级审核8 二级审核16 被驳回
      tpp.sales_status_name,
      tpp.is_custom,  -- 是否定制
      tpp.is_return_tips, -- 退货时是否有提示
      tpp.is_step, -- 是否启用阶梯价
      tpp.is_temp,--是否临时下单商品
      tpp.is_lianying,  -- 是否联营(0 自营(河姆渡有多家店铺) 1 联营(河姆渡以外的店铺))
      tpp.project_type, -- 项目产品
      pc.is_neicai,   -- 是否内采管专用类别（所有子类遵从一级分类设定规则）
      pc.is_special_project, -- 是否报备项目分类标识
      pc.is_channel,    -- 是否渠道馆专用类别（所有子类遵从一级分类设定规则）
      pc.is_crowd,      -- 是否众筹专用类别（所有子类遵从一级分类设定规则）
      tpp.first_sale_time,  -- 首次上架时间
      tpp.insert_time,
      current_timestamp() etl_time
from (
  select
    t.id product_id,-- 商品id
    t.realproductid real_product_id,
    t.goodsid spu_id,
    t.name product_name,
    t.price,
    t.cost,
    t.count virtual_inventory,
    t.salescount sales_count,
    t.ordermincount order_min_count,
    t.orderstepcount order_step_count,
    t.sku sale_properties,
    t.issales is_sales,
    t.dataid outside_code,
    t.model,
    t.currentcost purchase_price,
    IsOEM,
    t.averagecost avg_price,
    t.packcount pack_count,
    t.packunit pack_unit,
    t.salesstatus sales_status_id,
  	case t.salesstatus when 1 then '上架' when 2 then '下架' when 4 then '一级审核'
  	when 8 then '二级审核' when 16 then '被驳回' else '未提交审核' end sales_status_name,
    t.brandid brand_id,
    t.brand brand_name,
    t.supplierid supplier_id,
    t.isdirectdelivery is_direct_delivery,
    t.isinventory is_inventory,
    t.depositrate deposit_rate,
    t.iscustom is_custom,
    t.isreturn is_return_tips,
    t.inserttime insert_time,
    t.updatetime update_time,
    t.code u8code,
    t.u8name,
    t.keyword key_word,
    t.subname sub_name,
    t.categoryid1 category_id_level1,
    t.categoryid2 category_id_level2,
    t.categoryid3 category_id_level3,
    t.pricemobile price_mobile,
    t.costmobile cost_mobile,
    t.deposit,
    t.seriestag series_tag,
    t.goodsattr product_property,
    t.barcode bar_code,
    t.taxrate tax_rate,
    t.wmscode wms_code,
    t.usergroupid user_group_id,
    t.firstsaletime first_sale_time,
    t.integralpresentscale hebi_present_ratio,
    t.isstep is_step,
    t.istemp is_temp,
    t.producttype product_type_id,
  	case t.ProductType when 1 then '渠道类' when 2 then '项目类' when 3 then '服务类' else '' end product_type_name,
    t.warehouseid warehouse_id,
    t.islianying is_lianying,
    t.projecttype project_type
	from bak_homedo.t_product_product t
	left join bak_homedo.t_product_product_fortest tt on tt.id=t.id and tt.d = date_format(date_sub(current_date(), 1), 'yyyyMMdd')
	where t.d=date_format(date_sub(current_date(),1),'yyyyMMdd')
	and t.mark>0
	and tt.id is null
	and t.categoryid1 <> 12531 -- 测试分类
) tpp
left join (
   select UserGroup_Code,
      UserGroup_Name
   from bak_homedo.Base_UserGroup
  where d=date_format(date_sub(current_date(),1),'yyyyMMdd')
    and mark>0
) bu on bu.UserGroup_Code=tpp.user_group_id
left join dwd_homedo.dwd_sub_product_category pc on tpp.category_id_level3 = pc.category_id_level3;
```

解析该 SQL 的测试代码:

```scala
val tableLineage: NebulaTableLineage = lineageParser
			.parseNebulaTableLineage(Seq(log_product_click_inner), SQLEngine.Hive)
JsonUtil.printJsonString(tableLineage)
TimeUnit.SECONDS.sleep(3)
```

测试结果 Json:

```json
{
	"edges":{1916016388:-652237516,382245368:-652237516,-1117294188:-652237516,1428416426:-652237516
	},
	"nodes":[
		{
			"db":{
				"id":1238676905,
				"name":"dwd_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"dwd_sub_product_category",
			"schema":null,
			"tid":1916016388
		},
		{
			"db":{
				"id":2055766894,
				"name":"bak_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"t_product_product",
			"schema":null,
			"tid":382245368
		},
		{
			"db":{
				"id":2055766894,
				"name":"bak_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"t_product_product_fortest",
			"schema":null,
			"tid":-1117294188
		},
		{
			"db":{
				"id":2055766894,
				"name":"bak_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"Base_UserGroup",
			"schema":null,
			"tid":1428416426
		},
		{
			"db":{
				"id":1238676905,
				"name":"dwd_homedo",
				"type":"Hive"
			},
			"metadata":null,
			"name":"dwd_sub_product_info",
			"schema":null,
			"tid":-652237516
		}
	]
}
```

