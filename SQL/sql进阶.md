## 神奇的sql

### case表达式

##### case表达式概述

* 简单case表达式

	case sex 

	​	when '1' then '男' 

	​	when '2' then '女'

	else '其他' end

* 搜索case表达式

	case when sex = '1' then '男'

	​		when sex = '2' then '女'

	else '其他' end

* 注意事项
	* 统一各分支返回的数据类型
	* 不要忘了写end
	* 养成写else子句的习惯

##### 将已有编号方式转换为新的方式并统计

* select case pref_name

	​	when '德岛' then '四国'

	​	when '香川' then '四国'

	​	when '爱媛' then '四国'

	​	when '高知' then '四国'

	 	when '福冈' then '九州'

	​	when '佐贺' then '九州'

	​	when '长崎' then '九州'

	else '其他' end as district,

	sum(population) 

	from PopTbl

	group by case pref_name

	​	when '德岛' then '四国'

	​	when '香川' then '四国'

	​	when '爱媛' then '四国'

	​	when '高知' then '四国'

	 	when '福冈' then '九州'

	​	when '佐贺' then '九州'

	​	when '长崎' then '九州'

	else '其他' end;

* select case when population < 100 then '01' 

	​					when population >=100 and population < 200 then '02'

	​					when population >= 200 and population <300 then '03'

	​					when population >= 300 then '04'

	​			else null end as pop_class,

	​			count(*) as cnt

	​	from PopTbl

	group by case when population < 100 then '01' 

	​					when population >=100 and population < 200 then '02'

	​					when population >= 200 and population <300 then '03'

	​					when population >= 300 then '04'

	​			else null end ;

##### 用一条sql语句进行不同条件的统计

* select pre_name

	​			--男性人口

	​			sum(case when sex = '1' then population else 0 end) as cnt_m,

	​			--女性人口

	​			sum(case when sex='2' then population else 0 end) as cnt_f

	​	from PopTbl2

	group by pref_name;

##### 用check约束定义多个列的条件关系(蕴含式)

* constraint check_salary check 

	​					(case when sex='2' 

	​								then case when salary <= 20000 then 1 else 0 end 

	​								else 1 end = 1)

##### 在update语句里进行条件分支

* update Salaries 

	​		set salary = case when salary >= 30000

	​										then salary * 0.9

	​										when salary >= 25000 and salary < 28000

	​										then salary  * 1.2

	​										else salary end;

* update SomeTable

	​		set p_key = case when p_key = 'a'

	​									then 'b'

	​									when p_key = 'b'

	​									then 'a'

	​									else p_key end

	​		where p_key in ('a','b')

##### 表之间的数据匹配(exists进行的子查询容易用刀主键索引)

* select course_name,

	​			case when course_id in

	​								 (select course_id from OpenCourses

	​									where month = 200706) then 'O'

	​					else 'x' end as '6月',

	​			case when course_id in

	​								 (select course_id from OpenCourses

	​									where month = 200707) then 'O'

	​					else 'x' end as '7月',

	​			case when course_id in

	​								 (select course_id from OpenCourses

	​									where month = 200708) then 'O'

	​					else 'x' end as '8月'

	​	from CourseMaster;

* select CM.course_name,

	​			case when exists 

	​										(select course_id from OpenCourses OC

	​											where month = '200706'

	​											and OC.course_id = CM.course_id) Then 'O'

	​			else 'x' end as '6月',

	​			case when exists 

	​										(select course_id from OpenCourses OC

	​											where month = '200707'

	​											and OC.course_id = CM.course_id) Then 'O'

	​			else 'x' end as '7月',

	​			case when exists 

	​										(select course_id from OpenCourses OC

	​											where month = '200708'

	​											and OC.course_id = CM.course_id) Then 'O'

	​			else 'x' end as '8月'

	​	from CourseMaster CM;

##### 在case表达式中使用聚合函数

* select std_id,

	​			case when count(*) =1 --只加入了一个社团的学生

	​						then max(club_id)

	​						else max(case when main_club_flg = 'Y'

	​													then club_i 

	​													else null end)

	​			end as main_club

	​	from StudentClub

	group by std_id;

##### 本章要点

* 在group by 子句里使用case表达式，可以灵活地选择作为聚合的单位的编号或等级，这一点在进行非定制化统计时能发挥巨大的威力
* 在聚合函数中使用case表达式，可以轻松的将行结构的数据转换成列结构的数据
* 相反，聚合函数也可以嵌套进case表达式里使用
* 相比依赖于具体数据库的函数，case表达式有更强大的表达能力和更好的可移植性
* 正因为case表达式是一种表达式而不是语句，才有了这诸多优点

##### 练习题答案(未测试)

* select key,case when x > y

	​							then case when x > z 

	​												then x

	​												else z

	​										end

	​							else case when y > z

	​												then  y

	​												else z

	​									end

	​					end as greatest

	​	from Greatests;

* 略(略略略- -)

* selecct key from Greatests 

	​		order by case key 

	​								when 'A' then 1

	​								when 'B' then 0

	​								when 'D' then 2

	​								when 'C' then 3

	​								else null

	​						end;		

### 自连接的用法

##### 可重排列、排列、组合

*  select P1.name as name_1,P2.name as name_2

	​		from Products P1,Products P2

	​	where P1.name <> P2.name;

* select P1.name as name_1,P2.name as name_2

	​		from Products P1,Products P2

	​	where P1.name > P2.name;

* select P1.name as name_1,P2.name as name_2,P3.name as name_3

	​		from Products P1,Products P2,Products P3

	​	where P1.name > P2.name

	​		and P2.name > P3.name;

##### 删除重复行

* --用于删除重复行的SQL语句(1)：使用极值函数

	delete from Products P1

	​	where rowid < (select max(P2.rowid) 

	​									from Products P2 

	​									where P1.name = P2.name

	​									and P1.price = P2.price );

* --用于删除重复行的SQL语句(2)：使用非等值连接

	delete from Products P1

	​	where exists (select * 

	​								from Products P2

	​								where P1.name = P2.name 

	​									and P1.price = P2.price

	​									and P1.rowid < P2.rowid)

##### 查找局部不一致的列

* --用于查找是同一家人但住址却不同的记录的SQL语句

	select distinct A1.name,A1.address

	​	from Address A1,Address A2

	​	where A1.family_id = A2.family_id

	​		and A1.address <> A2.address;

* --用于查找价格相等但商品名称不同的记录的SQL语句

	select distinct P1.name,P1.price

	​	from Products P1,Products P2

	​	where P1.price = P2.price

	​		and P1.name <> P2.name;

##### 排序

* --排序：使用窗口函数/OLAP函数/分析函数(Oracle、DB2)

	select name,price,

	​	 rank() over(order by price desc) as rank_1,

	​	dense_rank() over(order by price desc) as rank_2

	from Products;

* --排序从1开始。如果已出现相同位次，则跳过之后的位次  (非等值自连接)

	select P1.name,

	​			P1.price,

	​			(select count(P2.price) 

	​				from Products P2

	​				where P2.price > P1.price ) + 1 as rank_1

	​		from Products P1

	​		order by rank_1;

* --排序：使用自连接

	select P1.name,

	​			max(P1.price) as price

	​			count(P2.name) + 1 as rank_1

	​	from Products P1 left outer join Products P2

	​		on P1.price < P2.price 

	​	group by P1.name

	​	order by rank_1;

* --不聚合，查看集合的包含关系

	select P1.name,P2.name

	​	from Product P1 left outer join Products P2

	​		on P1.price = P2.price;

##### 本章小结

* 自连接经常和非等值连接结合起来使用
* 自连接和group by 结合使用可以生成递归集合
* 将自连接看作不同表之间的连接更容易理解
* 应把表看作行的集合，用面向集合的方法来思考
* 自连接的性能开销更大，应尽量给用于连接的列建立索引

##### 练习题答案(未测试)

* select P1.name name_1,P2.name name_2

	​		from Products P1,Products P2

	​	where P1.name >= P2.name;

* select D1.district,D1.name,D1.price,

	​			(select count(D2.price) 

	​				from DistrictProducts D2

	​					where D2.price > D1.price 

	​						and D2.district = D1.district) + 1 as rank_1

	​	from DistrictProducts D1

	​	order by D1.district,D1.price desc;

	--剩余两种解法(略略略- -)

* update DistrictProducts D1

	​			set ranking = (select count(D2.price) + 1

	​												from DistrictProducts D2

	​												where D2.price > D1.price

	​													and D2.district = D1.district)

