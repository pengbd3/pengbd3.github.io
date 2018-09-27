# 维他命接口说明

> 备注：sql语句中凡是包含{$}表示变量,替换为对应参数值及可

## 获取商城商品图片

```sql
select g.bn,gcl.color,image.image_url,g.last_modify,g.intro from sdb_b2c_goods g left join (select goods_id,GROUP_CONCAT(color) color from (select DISTINCT p.goods_id,SUBSTRING_INDEX(SUBSTRING(p.spec_info,4),'、',1) color from sdb_b2c_products p where p.marketable = 'true' and p.disabled = 'false' ) gc group by gc.goods_id) gcl on g.goods_id = gcl.goods_id
                left join (select iia.target_id,GROUP_CONCAT(concat('http://mall.yingerfashion.com/',ii.l_url)) image_url from sdb_image_image_attach iia
		        left join sdb_image_image ii on iia.image_id = ii.image_id
		        where iia.target_type = 'goods' group by iia.target_id) image on g.goods_id = image.target_id
```

## 获取商品条数

参数说明：

|参数名称|类型|非必须|备注|
|:--:|--:|--|--:|
| lastmoddate | string(255) | N | 最后修改时间,默认值2015-01-01 00:00:00 |

```sql
select count(*) total from goodsbarcode  where goodsid in(
                  select goodsid from goods  where status=1 and lastmoddate > TO_DATE('{$lastmoddate}', 'yyyy-MM-dd HH24:mi:ss'))
```

## 获取库存信息

参数说明：

|参数名称|类型|非必须|备注|
|:--|--:|--|--:|
| lastmoddate | string(255) | N | 最后修改时间,默认值2015-01-01 00:00:00。只传入当前参数执行SQL1 |
| start | int(11) | N | 取数起始值,当该参有值则end参数为必填项。包含当前参数执行SQL2|
| end | int(11) | N | 取数截止值,当该参有值则start参数为必填项。包含当前参数执行SQL2|
| barcodes | string(255) | N | 条码，格式实例:'808631967000240','808631967000241'。包含当前参数执行SQL3 |

```sql
-- SQL1
select count(*) nums from (
                    select  a.barcode,a.code,a.channelid,a.name channelname,a.goodsno,a.goodsname,a.colorcode,a.colordesc,a.filedname,a.sizedesc,a.qty 总库存,nvl(c.qty,0) 占用库存,nvl(b.qty,0) 可用库存,nvl(d.qty,0) 在途库存 from (
                    select  gb.barcode, sg.sizedesc,c.code ,c.name ,a.channelid,a.colorid ,a.filedname ,a.goodsid ,a.qty,a.colordesc,a.colorcode,a.goodsname ,a.goodsno from viewavaistockdetail  
                    UNPIVOT(qty for filedname in  (S1,S2,S3,S4,S5,S6,S7,S8,S9,S10)) a
                    left join channel c on a.channelid =c.channelid 
                    left join goods g on g.goodsid =a.goodsid
                    left join sizecategory sg on  sg.sizecategoryid=g.sizecategoryid and sg.filedname=a.filedname
                    left join goodsbarcode gb on gb.goodsid=a.goodsid and gb.colorid=a.colorid and gb.sizeid=sg.id
                    where 
                    a.qty<>0 and a.qty<>-9999) a
                    left join 
                    (select * from viewavaistock   
                    UNPIVOT(qty for filedname in  (S1,S2,S3,S4,S5,S6,S7,S8,S9,S10)) a  
                    ) b on a.goodsid=b.goodsid and a.filedname=b.filedname and a.channelid=b.channelid and a.colorid=b.colorid
                    left join 
                    (select * from vstocklock  
                    UNPIVOT(qty for filedname in  (S1,S2,S3,S4,S5,S6,S7,S8,S9,S10)) a  
                    ) c on c.goodsid=b.goodsid and c.filedname=b.filedname and c.channelid=b.channelid and c.colorid=b.colorid 
                     left join 
                    (select * from vstocktransit   
                    UNPIVOT(qty for filedname in  (S1,S2,S3,S4,S5,S6,S7,S8,S9,S10)) a  
                    ) d on c.goodsid=d.goodsid and c.filedname=d.filedname and c.channelid=d.channelid and c.colorid=d.colorid   
                    ) stock where stock.barcode in (
                   select DISTINCT goodscontent.barcode from (
                    select gb.barcode,to_char(gb.lastmoddate,'YYYY/MM/DD HH24:MI:SS') lastmoddate,sd.channelid,rownum rm,c.code
                    from GOODSBARCODE gb
                    left join goods g on gb.goodsid = g.goodsid
                    left join stockdetail sd on g.GOODSID = sd.goodsid
                    left join CHANNEL c on sd.channelid = c.channelid 
                    where g.lastmoddate > to_date('{$lastmoddate}','yyyy-mm-dd HH24:MI:SS') order by g.lastmoddate desc ) goodscontent )
```

```sql
-- SQL2
select DISTINCT goodscontent.barcode,goodscontent.lastmoddate,goodscontent.channelid,goodscontent.code from (
                    select gb.barcode,to_char(gb.lastmoddate,'YYYY/MM/DD HH24:MI:SS') lastmoddate,sd.channelid,rownum rm,c.code
                    from GOODSBARCODE gb
                    left join goods g on gb.goodsid = g.goodsid
                    left join stockdetail sd on g.GOODSID = sd.goodsid
                    left join CHANNEL c on sd.channelid = c.channelid 
                    where g.lastmoddate > to_date('{$lastmoddate}','yyyy-mm-dd HH24:MI:SS') order by g.lastmoddate desc ) goodscontent where goodscontent.rm BETWEEN {$start} and {$end}
```

``` sql
-- SQL3
select DISTINCT gb.barcode,to_char(gb.lastmoddate,'YYYY/MM/DD HH24:MI:SS') lastmoddate,sd.channelid,c.code 
                    from GOODSBARCODE gb
                    left join goods g on gb.goodsid = g.goodsid
                    left join stockdetail sd on g.GOODSID = sd.goodsid
                    left join CHANNEL c on sd.channelid = c.channelid 
                    where gb.barcode in ({$barcodes})
```

## 获取丽晶库存信息

参数说明：

|参数名称|类型|非必须|备注|
|:--|--:|--|--:|
| barcode | string(255) | Y | 编码ID，格式实例:'808631967000240','808631967000241' |

```sql
select * from (
                select  a.barcode,a.code,a.channelid,a.name channelname,a.goodsno,a.goodsname,a.colorcode,a.colordesc,a.filedname,a.sizedesc,a.qty 总库存,nvl(c.qty,0) 占用库存,nvl(b.qty,0) 可用库存,nvl(d.qty,0) 在途库存 from (
                select  gb.barcode, sg.sizedesc,c.code ,c.name ,a.channelid,a.colorid ,a.filedname ,a.goodsid ,a.qty,a.colordesc,a.colorcode,a.goodsname ,a.goodsno from viewavaistockdetail  
                UNPIVOT(qty for filedname in  (S1,S2,S3,S4,S5,S6,S7,S8,S9,S10)) a
                left join channel c on a.channelid =c.channelid 
                left join goods g on g.goodsid =a.goodsid
                left join sizecategory sg on  sg.sizecategoryid=g.sizecategoryid and sg.filedname=a.filedname
                left join goodsbarcode gb on gb.goodsid=a.goodsid and gb.colorid=a.colorid and gb.sizeid=sg.id
                where 
                a.qty<>0 and a.qty<>-9999) a
                left join 
                (select * from viewavaistock   
                UNPIVOT(qty for filedname in  (S1,S2,S3,S4,S5,S6,S7,S8,S9,S10)) a  
                ) b on a.goodsid=b.goodsid and a.filedname=b.filedname and a.channelid=b.channelid and a.colorid=b.colorid
                left join 
                (select * from vstocklock  
                UNPIVOT(qty for filedname in  (S1,S2,S3,S4,S5,S6,S7,S8,S9,S10)) a  
                ) c on c.goodsid=b.goodsid and c.filedname=b.filedname and c.channelid=b.channelid and c.colorid=b.colorid 
                 left join 
                (select * from vstocktransit   
                UNPIVOT(qty for filedname in  (S1,S2,S3,S4,S5,S6,S7,S8,S9,S10)) a  
                ) d on c.goodsid=d.goodsid and c.filedname=d.filedname and c.channelid=d.channelid and c.colorid=d.colorid   
                ) stock where stock.barcode in ({$goodsbarcode})
```
