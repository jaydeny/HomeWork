# HomeWork
## 分割表(Data-Migrate)
1、两个源表：==china_city==,==lagou_position_bk==。
2、整理数据，分成三个表：lagou_city,lagou_position,lagou_company。
### 一、lagou_city表字段**(cid,province,city,district)**数据从表china_city获取省市县数据。
*方法：*
```
create table lagou_city as
select d.id, p.cityName as province, c.cityName as city, d.cityName as district from
  (select * from china_city where depth=3) d
  join china_city c on d.parentId = c.id and c.depth=2
  join china_city p on c.parentId = p.id and p.depth=1
union
select c.id, p.cityName as province, c.cityName as city, null as district from (select * from china_city where depth=2) c
join china_city p on c.parentId = p.id and p.depth = 1;
```
代码解析：利用**join--union**查询
首先查询出==china_city==表里的县级数据作为表d,
再连接查询==china_city==表里的市级数据作为表c,
再连接查询==china_city==表里的省级数据作为表p,
联合查询出仅有省市数据。
*注意：*
1、depth为等级，1为省级，2为市级，3为县级。
2、```create table as select```复制另一张表数据时不会复制索引和主键。
3、**union**不重复两张表数据
**建议：**
```
create table lagou_city(
id int primary key,
province varchar(20),
city varchar(20),
district varchar(20));

insert into lagou_city
select d.id, p.cityName, c.cityName, d.cityName from
  (select * from china_city where depth=3) d
  join china_city c on d.parentId = c.id and c.depth=2
  join china_city p on c.parentId = p.id and p.depth=1
union
select c.id, p.cityName as province, c.cityName as city, null as district from (select * from china_city where depth=2) c
join china_city p on c.parentId = p.id and p.depth = 1;
```
### 二、lagou_company表字段**(cid,short_name,full_name,size,financestage)**数据从lagou_position_bk中获取公司id,公司简称，公司全称，公司大小，公司资金来源
*方法：*
```
drop table if exists lagou_company;
create table lagou_company as
select distinct t.company_id as cid,
        t.company_short_name as short_name,
        t.company_full_name  as full_name,
        t.company_size       as size,
        t.financestage
  from lagou_position_bk t;
```
代码解析：确认数据库是否已经存在lagou_company,如果存在则删除。
从表lagou_position_bk中查询出```distinct```不重复公司信息复制到新表当中。
### 三、从lagou_position_bk表中分离出城市、公司信息
*方法：*
```
create table lagou_position
as
select pid, cid as city, company_id as company, position, field, salary_min, salary_max, workyear, education, ptype, pnature, advantage, published_at, updated_at from
(
  select p.*, c.cid from (select * from lagou_position_bk where district is null) p
  join lagou_city c on c.city like concat(p.city, '%') and c.district is null

  union all

  select p.*, c.cid from (select * from lagou_position_bk where district is not null) p
  join lagou_city c on c.city like concat(p.city, '%') and c.district like concat(p.district, '%')

) as ppp;
```
代码解析：利用**join--union all**查询
先查询出lagou_position_bk中district为null的数据作为表p,
再连接查询表lagou_city条件lagou_city中城市名包含p表中城市名并且县级为Null，
联合查询出于上面相反有县级信息数据，
最后查询出需要字段，复制到新表lagou_position中。
*注意：*最后查询出数据必须要As成一张临时表，否则报错。
### 四、总结
经过本次对表数据进行拆分整理后，对sql语句有了新的认识，学会做之前先整理好思路，并且对连接查询有了新的认识。
