---
layout:     post
title:      "PgSql 数据类型"
subtitle:   " \"PGSQL 独有\""
date:       2017-12-01 12:00:00
author:     "BaiDong"
header-img: "img/post-bg-alibaba.jpg"
catalog: true
tags:
    - 数据库
---

名字 存储尺寸 描述 范围

smallint 2字节 小范围整数 -32768 to +32767

integer 4字节 整数的典型选择 -2147483648 to +2147483647

bigint 8字节 大范围整数 -9223372036854775808 to +9223372036854775807

decimal 可变 用户指定精度，精确 最高小数点前131072位，以及小数点后16383位

numeric 可变 用户指定精度，精确 最高小数点前131072位，以及小数点后16383位

real 4字节 可变精度，不精确 6位十进制精度

double precision 8字节 可变精度，不精确 15位十进制精度

smallserial 2字节 自动增加的小整数 1到32767

serial 4字节 自动增加的整数 1到2147483647

bigserial 8字节 自动增长的大整数 1到9223372036854775807


表 8-4. 字符类型

名字 描述

character varying(n), varchar(n) 有限制的变长

character(n), char(n) 定长，空格填充

text 无限变长

表 8-9. 日期/时间类型

名字 存储尺寸 描述 最小值 最大值 解析度

timestamp [ (p) ] [ without time zone ] 8字节 包括日期和时间（无时区） 4713 BC 294276 AD 1微秒 / 14位

timestamp [ (p) ] with time zone 8字节 包括日期和时间，有时区 4713 BC 294276 AD 1微秒 / 14位

date 4字节 日期（没有一天中的时间） 4713 BC 5874897 AD 1日

time [ (p) ] [ without time zone ] 8字节 一天中的时间（无日期） 00:00:00 24:00:00 1微秒 / 14位

time [ (p) ] with time zone 12字节 仅仅是一天中的时间，带有时区 00:00:00+1459 24:00:00-1459 1微秒 / 14位

interval [ fields ] [ (p) ] 16字节 时间间隔 -178000000年 178000000年 1微秒 / 14位

注意: SQL要求只写timestamp等效于timestamp without time zone，并且Postgr

表 8-19. 布尔数据类型

名字 存储字节 描述

boolean 1字节 状态为真或假

"真"状态的有效文字值是：

TRUE

't'

'true'

'y'

'yes'

'on'

'1'

而对于"假"状态，你可以使用下面这些值：

FALSE

'f'

'false'

'n'

'no'

'off'

'0'

表 8-20. 几何类型

名字 存储尺寸 表示 描述

point 16字节 平面上的点 (x,y)

line 32字节 无限长的线 {A,B,C}

lseg 32字节 有限线段 ((x1,y1),(x2,y2))

box 32字节 矩形框 ((x1,y1),(x2,y2))

path 16+16n字节 封闭路径（类似于多边形） ((x1,y1),...)

path 16+16n字节 开放路径 [(x1,y1),...]

polygon 40+16n字节 多边形（类似于封闭路径） ((x1,y1),...)

circle 24字节 圆 <(x,y),r> (center point and radius)

表 8-21. 网络地址类型

名字 存储尺寸 描述

cidr 7或19字节 IPv4和IPv6网络

inet 7或19字节 IPv4和IPv6主机以及网络

macaddr 6字节 MAC地址

组合类型！

这里有两个定义组合类型的简单例子：

CREATE TYPE complex AS (
    r       double precision,
    i       double precision
);

CREATE TYPE inventory_item AS (
    name            text,
    supplier_id     integer,
    price           numeric
);

定义了类型之后，我们可以用它们来创建表：

CREATE TABLE on_hand (
    item      inventory_item,
    count     integer
);

INSERT INTO on_hand VALUES (ROW('fuzzy dice', 42, 1.99), 1000);

查询语言（SQL）函数

这个函数从emp表中移除具有负值薪水的行：

CREATE FUNCTION clean_emp() RETURNS void AS '
    DELETE FROM emp
        WHERE salary < 0;
' LANGUAGE SQL;

SELECT clean_emp();

 clean_emp

调整余额并且返回新的余额！

CREATE FUNCTION tf1 (accountno integer, debit numeric) RETURNS integer AS $$
    UPDATE bank
        SET balance = balance - debit
        WHERE accountno = tf1.accountno
    RETURNING balance;
$$ LANGUAGE SQL;

通过执行这个函数从账号17中借100.00：

SELECT tf1(17, 100.0);

组合类型上的SQL函数

CREATE TABLE emp (
    name        text,
    salary      numeric,
    age         integer,
    cubicle     point
);

INSERT INTO emp VALUES ('Bill', 4200, 45, '(2,1)');

CREATE FUNCTION double_salary(emp) RETURNS numeric AS $$
    SELECT $1.salary * 2 AS salary;
$$ LANGUAGE SQL;

SELECT name, double_salary(emp.*) AS dream
    FROM emp
    WHERE emp.cubicle ~= point '(2,1)';

 创建一个触发器，表中的行在任何时候被插入或更新时，当前用户名和时间也会被标记在该行中。并且它会检查雇员的姓名以及薪水。

--创建测试表

CREATE TABLE emp (
    empname text,
    salary integer,
    last_date timestamp,
    last_user text
);

--创建触发器函数

CREATE FUNCTION emp_stamp() RETURNS trigger AS $emp_stamp$
    BEGIN
        -- 检查 empname 以及 salary
        IF NEW.empname IS NULL THEN
            RAISE EXCEPTION 'empname cannot be null';
        END IF;
        IF NEW.salary IS NULL THEN
            RAISE EXCEPTION '% cannot have null salary', NEW.empname;
        END IF;

        -- 谁会倒贴钱为我们工作？
        IF NEW.salary < 0 THEN
            RAISE EXCEPTION '% cannot have a negative salary', NEW.empname;
        END IF;

        -- 记住谁在什么时候改变了工资单
        NEW.last_date := current_timestamp;
        NEW.last_user := current_user;
        RETURN NEW;
    END;
$emp_stamp$ LANGUAGE plpgsql;

--创建触发器

CREATE TRIGGER emp_stamp BEFORE INSERT OR UPDATE ON emp
    FOR EACH ROW EXECUTE PROCEDURE emp_stamp();

--测试触发器

test=# insert into emp values ('John');  --salary为空，触发器报错

ERROR:  John cannot have null salary

CONTEXT:  PL/pgSQL function emp_stamp() line 7 at RAISE

test=# insert into emp values (null,1200);   --empname为空，触发器报错

ERROR:  empname cannot be null

CONTEXT:  PL/pgSQL function emp_stamp() line 4 at RAISE

test=# insert into emp values ('John',-200); --salary为负数，触发器报错

ERROR:  John cannot have a negative salary

CONTEXT:  PL/pgSQL function emp_stamp() line 10 at RAISE

test=# insert into emp values ('Bob',1200);  --成功插入正常数据，并记录了最后操作时间和操作用户

INSERT 0 1

test=# select * from emp;

 empname | salary |         last_date          | last_user
---------+--------+----------------------------+-----------
 Bob     |   1200 | 2017-08-09 17:39:23.671957 | postgres
(1 row)

.用于审计的触发器过程

这个例子触发器保证了在emp表上的任何插入、更新或删除一行的动作都被记录（即审计）在emp_audit表中。当前时间和用户名以及在其上执行的操作类型都会被记录到行中。

--创建测试表

create table emp (
empname text not null,
salary integer
);

--创建审计表

create table emp_audit(
operation       char(1)   not null,
stamp           timestamp not null,
userid          text      not null,
empname         text      not null,
salary          integer
);

--创建触发器函数

create or replace function process_emp_audit() returns trigger as $emp_audit$

begin
   if (TG_OP = 'DELETE') then
     insert into emp_audit select 'D',now(),user,old.*;
     return old;
   elsif (TG_OP = 'UPDATE') then
     insert into emp_audit select 'U',now(),user,new.*;
     return new;
   elsif (TG_OP = 'INSERT') then
     insert into emp_audit select 'I',now(),user,new.*;
     return new;
   end if;
   return null;
end;
$emp_audit$ language plpgsql;

--创建触发器

create trigger emp_audit
after insert or update or delete on emp
for each row execute procedure process_emp_audit();

--测试触发器

test=# insert into emp values ('John',1200);
INSERT 0 1
test=# select * from emp_audit;
 operation |           stamp            |  userid  | empname | salary
-----------+----------------------------+----------+---------+--------
 I         | 2017-08-09 18:18:10.189772 | postgres | John    |   1200
(1 row)
---


