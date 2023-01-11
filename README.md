# INGOS

При выполнении задания подразумевается наличие следующих прав у пользователя
 - создание и редактирование таблиц
 - создание и редактирование типов
 - создание и редактирование функций
 
Если его нет его можно создать таким

```sql

begin execute immediate ('drop user IGS_TEST cascade'); exception when others then null; end;
/

create user IGS_TEST identified by "1234" quota 0m on SYSTEM;
grant CONNECT to IGS_TEST;
grant RESOURCE to IGS_TEST;
grant CREATE ANY TABLE to IGS_TEST;
grant ALTER ANY TABLE to IGS_TEST;
grant EXECUTE ANY PROCEDURE to IGS_TEST;

connect IGS_TEST/1234@localhost:1521/XE;

```



##№ Вопрос 1
Вывести 1000 случайных чисел от 1 до 1000, таких что
не повторяются в этой последовательности, больше чем 3 раза.

Ответ 

```diff
! Ответ
```

<font color=darkorange size=10>Ответ</font>

```sql
select trunc(dbms_random.value(level, level+3))
from dual connect by level <= 1000
order by 1;
```



## Вопрос 2.
Имеется таблица без первичного ключа. Известно, что
в таблице имеется задвоение данных. Необходимо удалить дубликаты из таблицы.

``` create table t (a number, b number); ```

Пример данных:
| a | b |
| --- | --- |
| 1 | 1 |
| 2 | 2 |
| 2 | 2 |
| 3 | 3 |
| 3 | 3 |
| 3 | 3 |

Требуемый результат:
| a | b |
| --- | --- |
| 1 | 1 |
| 2 | 2 |
| 3 | 3 |

```sql 
-- небольшой дамп
begin execute immediate ('drop table t'); exception when others then null; end;
/
create table t (a number, b number);
insert into t
select 1, 1 from dual
union all select 2, 2 from dual
union all select 2, 2 from dual
union all select 3, 3 from dual
union all select 3, 3 from dual
union all select 3, 3 from dual
commit;

select * from t;
-- убираем дубликаты
delete from t
 where rowid in (select rowid from t
 minus
 select min(rowid) from t group by a);
commit ;
-- результат
select * from t;
```




Вопрос 3. XML

Имеется
```
xmltype('
<root>
<row>
<col>v11</col>
<col>v12</col>
<col>v13</col>
<col>v14</col>
</row>
<row>
<col>v21</col>
<col>v22</col>
<col>v23</col>
<col>v24</col>
</row>
</root>')
```

Необходимо:

а) Получить выборку
| C1 | C2 | C3 | C4 |
| --- | --- | --- | --- |
| v11 | v12 | v13 | v14 |
| v21 | v22 | v23 | v24 |

Условия: количество узлов row может варьироваться,
col всегда статично = 4 шт в пределах row

Ответ

```sql 
select
  tseq.extract('/row/col[position()=1]/text()').getStringVal()  c1,
  tseq.extract('/row/col[position()=2]/text()').getStringVal()  c2,
  tseq.extract('/row/col[position()=3]/text()').getStringVal()  c3,
  tseq.extract('/row/col[position()=4]/text()').getStringVal()  c4
from
  (select XmlType('
    <root>
      <row>
        <col>v11</col>
        <col>v12</col>
        <col>v13</col>
        <col>v14</col>
      </row>
      <row>
        <col>v21</col>
        <col>v22</col>
        <col>v23</col>
        <col>v24</col>
      </row>
    </root>') as val from dual
  ) txml,
  table(XmlSequence(txml.val.extract('/root/row'))) tseq;
```


b) Получить в виде результата колонку с типом xmltype SQL запроса со следующей структурой:
```XML
<root>
<data row="1" col="1">v11</data>
<data row="1" col="2">v12</data>
<data row="1" col="3">v13</data>
<data row="1" col="4">v14</data>
<data row="2" col="1">v21</data>
<data row="2" col="2">v22</data>
<data row="2" col="3">v23</data>
<data row="2" col="4">v24</data>
</root>
```

Реализовать данный запрос не используя XSLT трансформацию.

Условия: количество узлов row и col может варьироваться (прим. это более сложный пример,

можно вернуть условие что количество col всегда статично = 4 шт в пределах row)

```sql 
-- вариант c различное количествово row и col
with xml as (
  select xmltype('
    <root>
      <row>
        <col>v11</col>
        <col>v12</col>
        <col>v13</col>
      </row>
      <row>
        <col>v21</col>
        <col>v22</col>
        <col>v23</col>
        <col>v24</col>
        <col>v25</col>
      </row>
      <row>
        <col>v31</col>
        <col>v32</col>
      </row>
    </root>
  ') val from dual
)
, v_rows as (
  select
    rownum row_count,
    value(t) t_rows
  from
    xml,
    table(xmlsequence(extract(xml.val, 'root/row'))) t
)
, res as (select
  v_rows.row_count p_row,
  extractvalue(value(w), 'col') p_col
from
  v_rows,
  table(xmlsequence(extract(v_rows.t_rows, 'row/col'))) w
)
select
  xmlelement("root",
    xmlagg(
      xmlelement("data",
        xmlattributes(p_row as "row", p_col as "col")
      ))
    ) as res from res;
```

c) Получить в виде результата колонку с типом xmltype SQL запроса со следующей структурой:
```xml
<root>
<data row="1" col="1">v11</data>
<data row="1" col="2">v12</data>
<data row="1" col="3">v13</data>
<data row="1" col="4">v14</data>
<data row="2" col="1">v21</data>
<data row="2" col="2">v22</data>
<data row="2" col="3">v23</data>
<data row="2" col="4">v24</data>
</root>
```

Реализовать данный запрос используя XSLT трансформацию.

Условия: количество узлов row и col может варьироваться

Ответ
```sql
with xlst as (
  select
    xmltype('
    <root>
      <row>
        <col>v11</col>
        <col>v12</col>
        <col>v13</col>
      </row>
      <row>
        <col>v21</col>
        <col>v22</col>
        <col>v23</col>
        <col>v24</col>
        <col>v25</col>
      </row>
      <row>
        <col>v31</col>
        <col>v32</col>
      </row>
    </root>
  ') xml,
    xmltype('
    <xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
    <xsl:output method="xml" version="1.0" encoding="UTF-8" indent="yes"/>

    <xsl:template match="/root">
      <root>
        <xsl:for-each select="/root/row">
        <xsl:variable name="i" select="position()" />

        <xsl:for-each select="./col">
        <xsl:variable name="j" select="position()" />
         
          <data row="{$i}" col="{$j}">
            <xsl:value-of select="."></xsl:value-of>
          </data>
          
          </xsl:for-each>
        
        </xsl:for-each>
      </root>
    </xsl:template>
    </xsl:stylesheet>
  ') xsl
  from dual
)
select xmltransform(xml, xsl) as res from xlst;
```



/*
Вопрос 4. Коллекции

Имеется декларация типа:

```sql 
CREATE OR REPLACE TYPE TNUM as table of number; 
```

Необходимо написать реализацию функции, возвращающая в качестве
.
результата заполненный массив имеющий тип TNUM с значениями от 1..1000

Ответ

```sql

create or replace type tnum as table of number;
/
create or replace function func_4 return tnum
is
v_num tnum := tnum();
begin
  v_num.extend;        
  select
    cast(
      multiset(
        select level from dual
        where level >= 1 connect by level <= 1000) as tnum
        )
  into v_num from dual;
  return v_num;
end;
/
select * from table(func_4());
```


Вопрос 5. Регулярные выражения

Задана строка '1,2,3,4', необходимо, используя регулярные выражения

в запросе получить результат:

| C1 | C2 | C3 | C4 |
| --- | --- | --- | --- |
| 1 | 2 | 3 | 4 |

Ответ 

```sql 

select
  regexp_substr(str, '([[:digit:]]{1})+', 1, 1) C1,
  regexp_substr(str, '([[:digit:]]{1})+', 1, 2) C2,
  regexp_substr(str, '([[:digit:]]{1})+', 1, 3) C3,
  regexp_substr(str, '([[:digit:]]{1})+', 1, 4) C4
FROM (select '1,2,3,4' as str from dual);
```


Вопрос 6. Использование pipelined функции

Имеется таблица dept со следующей структурой:

| Name | Type | Nullable | Default | Comments |
| --- | --- | --- | --- | --- |
| DEPTNO | NUMBER |   |   |   |
| DNAME | VARCHAR2(14) | Y |   |   |
| LOC | VARCHAR2(13) | Y |   |   |

Необходимо реализовать функцию PL/SQL которая будет возвращать выборку

из таблицы dept заданную минимальным и максимальным значением поля DEPTNO.

Реализуемая функция должна использовать метод pipelined.

Ответ 
```sql 
begin execute immediate ('drop table DEPT'); exception when others then null; end;
/
begin execute immediate ('drop type t_tab'); exception when others then null; end;
/
begin execute immediate ('drop type t_rec'); exception when others then null; end;
/

create table DEPT
(
  deptno  number not null,
  name  varchar2(100),
  loc  varchar2(30)
);
comment on table DEPT is 'Департамент';
comment on column DEPT.deptno is 'Код';
comment on column DEPT.name is 'Наименование';
comment on column DEPT.loc is 'Расположение';

insert into DEPT
select 10, 'Администрация', 'Корпус 1, 2 этаж' from dual
union all select 20, 'Отдел продаж', 'Корпус 3, 4 этаж' from dual
union all select 25, 'Отдел логистики', 'Корпус 2, 4 этаж' from dual
union all select 30, 'Отдел архитектуры и разработки приложений', 'Корпус 4, 1 этаж' from dual
union all select 35, 'Отдел Тестирования, сопробождения и внедрения', 'Корпус 1, 6 этаж' from dual
union all select 40, 'Бухгалтерия', 'Корпус 1, 1 этаж' from dual
union all select 50, 'Юридический отдел', 'Корпус 4, 1 этаж' from dual
union all select 60, 'Отдел кадров', 'Корпус 2, 4 этаж' from dual
union all select 70, 'Отдел систем внутренней безопасности', 'Корпус 2, 1 этаж' from dual;

commit;

create type t_rec as object (
  deptno number,
  name varchar2(100),
  loc varchar2(30)
);
/

create type t_tab is table of t_rec;
/

create or replace function func_6 return t_tab pipelined
as
 v_tab t_tab := t_tab();
begin
  v_tab.extend;
  for i in (
    select deptno, name, loc
    from dept
    where
      deptno = (select max(deptno) from dept)
      or
      deptno =  (select min(deptno) from dept)
     order by deptno
    )
  loop
  pipe row(t_rec(i.deptno, i.name, i.loc));
  end loop;
  return;
end;
/

select * from table(func_6);
```

/*
7. Отчёт для отдела маркетинга
Отделу маркетинга требуется сводная выгрузка по клиентам,
с гранулярностью до клиента, при этом для каждого клиента в выборке
должны быть «лучшие» адрес, телефон и адрес электронной почты. То есть,
в результирующей выборке по каждому клиенту есть только одна строка.
При этом:
1) Лучший адрес отбирается по приоритету фактический > регистрации > домашний,
  при наличии нескольких адресов одного приоритета выбирается наиболее полный
  (заполнено больше из перечня атрибутов city-street-house-flat,
  при равенстве по заполненности выбирается последний по дате внесения в базу.
2) Лучший телефон это последний по дате внесения в базу
3) Лучший email это первый по дате внесения в базу
4) Данные по контактам и адресам – не архивные
*/

begin execute immediate ('drop table ADDRESSES'); exception when others then null; end;
/
begin execute immediate ('drop table CONTACTS'); exception when others then null; end;
/
begin execute immediate ('drop table CLIENTS'); exception when others then null; end;
/

create table CLIENTS
(
  id      NUMBER NOT NULL,
  name     VARCHAR2(300)
);
alter table CLIENTS add constraint PK_CLIENTS primary key (id);
comment on table CLIENTS is 'Клиенты';
comment on column CLIENTS.id is 'Уникальный код';
comment on column CLIENTS.name is 'Наименование';

create table CONTACTS
(
  id  NUMBER NOT NULL,
  client_id  NUMBER,
  c_type  NUMBER,
  c_info  VARCHAR2(50),
  created  DATE,
  active  CHAR(1)
);
alter table CONTACTS add constraint PK_CONTACTS primary key (id);
alter table CONTACTS add constraint FK_CONTACTS$CLIENTS foreign key (client_id) references CLIENTS(id);
comment on table CONTACTS is 'Контакты';
comment on column CONTACTS.id is 'Уникальный код';
comment on column CONTACTS.client_id is 'код клиента';
comment on column CONTACTS.c_type is 'Тип контакта 1-телефон 2-email';
comment on column CONTACTS.c_info is 'Контакт – телефон либо адрес email';
comment on column CONTACTS.created is 'Дата внесения в базу';
comment on column CONTACTS.active is 'Y/N активный или архив';

create table ADDRESSES
(
  id  NUMBER NOT NULL,
  client_id  NUMBER,
  a_type  NUMBER,
  city  VARCHAR2(100),
  street  VARCHAR2(500),
  house  VARCHAR2(10),
  flat  VARCHAR2(5),
  created  DATE,
  active  CHAR(1)
);
alter table ADDRESSES add constraint PK_ADDRESSES primary key (id);
alter table ADDRESSES add constraint FK_ADDRESSES$CLIENTS foreign key (client_id) references CLIENTS(id);
comment on table ADDRESSES is 'Адреса';
comment on column ADDRESSES.id is 'Уникальный код';
comment on column ADDRESSES.client_id is 'код клиента';
comment on column ADDRESSES.a_type is 'Тип адреса 1-домашний 2-регистрации 3- фактический';
comment on column ADDRESSES.city is 'Город';
comment on column ADDRESSES.street is 'Улица';
comment on column ADDRESSES.house is 'Дом';
comment on column ADDRESSES.flat is 'Квартира';
comment on column ADDRESSES.created is 'Дата внесения в базу';
comment on column ADDRESSES.active is 'Y/N активный или архив';

insert into CLIENTS
select rownum, str from (
  select 'Николаев Демид Святославович' str from  dual
  union select 'Елисеев Геральд Назарович' from  dual
  union select 'Чиркаш Велор Яковлевич' from  dual
  union select 'Зубов Степан Адамович' from  dual
  union select 'Ешевский Панкрат Эдуардович' from  dual
  union select 'Туров Тимофей Львович' from  dual
  union select 'Князев Фома Вадимович' from  dual
  union select 'Сафиюлин Артур Ефимович' from  dual
  union select 'Пичугин Томас Рубенович' from  dual
  union select 'Снаткин Клим Тимурович' from  dual
);
commit;

insert into CONTACTS(id,client_id,c_type,c_info,created,active)
  select '1','1','2','7kt1001@ya.ru',to_date('10.09.17','DD.MM.RR'),'N' from dual
  union all select '2','1','2','2wed002@rm.net',to_date('17.09.17','DD.MM.RR'),'N' from dual
  union all select '3','1','2','9ol0003@gmial.com',to_date('12.09.17','DD.MM.RR'),'N' from dual
  union all select '4','2','2','z0yw004@rambler.ru',to_date('28.08.17','DD.MM.RR'),'Y' from dual
  union all select '5','3','2','aavp005@gmial.com',to_date('26.08.17','DD.MM.RR'),'Y' from dual
  union all select '6','3','1','+79916941006',to_date('02.09.17','DD.MM.RR'),'Y' from dual
  union all select '7','3','2','psao007@rm.net',to_date('23.08.17','DD.MM.RR'),'Y' from dual
  union all select '8','3','2','mx1u008@mail.ru',to_date('16.09.17','DD.MM.RR'),'Y' from dual
  union all select '9','4','2','zg0d009@xxx.org',to_date('01.09.17','DD.MM.RR'),'Y' from dual
  union all select '10','4','2','9vho010@gmial.com',to_date('25.08.17','DD.MM.RR'),'Y' from dual
  union all select '11','4','2','u4ug011@rm.net',to_date('06.09.17','DD.MM.RR'),'N' from dual
  union all select '12','5','2','s1cb012@list.ru',to_date('09.09.17','DD.MM.RR'),'Y' from dual
  union all select '13','5','2','gbf9013@gmial.com',to_date('17.09.17','DD.MM.RR'),'Y' from dual
  union all select '14','5','1','89166821814',to_date('11.09.17','DD.MM.RR'),'N' from dual
  union all select '15','5','1','+79420025415',to_date('05.09.17','DD.MM.RR'),'Y' from dual
  union all select '16','5','1','89270911816',to_date('14.09.17','DD.MM.RR'),'N' from dual
  union all select '17','4','1','8976543231',to_date('28.08.17','DD.MM.RR'),'Y' from dual
  union all select '18','6','2','65kh018@rambler.ru',to_date('11.09.17','DD.MM.RR'),'N' from dual
  union all select '19','6','2','a44p019@xxx.org',to_date('28.08.17','DD.MM.RR'),'Y' from dual
  union all select '20','6','1','+79860036520',to_date('30.08.17','DD.MM.RR'),'N' from dual
  union all select '21','6','2','p1ul021@xxx.org',to_date('25.08.17','DD.MM.RR'),'N' from dual
  union all select '22','4','1','+76789787654',to_date('13.09.17','DD.MM.RR'),'Y' from dual
  union all select '23','7','2','n5rg023@xxx.org',to_date('03.09.17','DD.MM.RR'),'Y' from dual
  union all select '24','7','2','qi29024@gmial.com',to_date('02.09.17','DD.MM.RR'),'N' from dual
  union all select '25','7','1','89761438425',to_date('28.08.17','DD.MM.RR'),'Y' from dual
  union all select '26','7','1','89582481426',to_date('26.08.17','DD.MM.RR'),'N' from dual
  union all select '27','8','2','st9o027@ya.ru',to_date('17.09.17','DD.MM.RR'),'Y' from dual
  union all select '28','8','2','n7lp028@list.ru',to_date('05.09.17','DD.MM.RR'),'Y' from dual
  union all select '29','8','1','89916956829',to_date('25.08.17','DD.MM.RR'),'Y' from dual
  union all select '30','9','2','3ncp030@mail.ru',to_date('09.09.17','DD.MM.RR'),'Y' from dual
  union all select '31','9','2','i5p5031@yandex.ru',to_date('02.09.17','DD.MM.RR'),'N' from dual
  union all select '32','10','1','+79551790532',to_date('31.08.17','DD.MM.RR'),'Y' from dual
  union all select '33','10','2','d4lp033@mail.ru',to_date('14.09.17','DD.MM.RR'),'N' from dual;
commit;

insert into ADDRESSES(id,client_id,a_type,city,street,house,flat,created,active)
  select '1','1','3','Тольятти','ул. Степана Разина','11','10',to_date('06.09.17','DD.MM.RR'),'Y' from dual
  union all select '2','1','3','Туапсе','ул. Степана Разина','9','2',to_date('05.09.17','DD.MM.RR'),'N' from dual
  union all select '3','2','2','Казань','9-я Парковая улица','17','5',to_date('10.09.17','DD.MM.RR'),'N' from dual
  union all select '4','2','3','Казань','ул. Б. Хмельницкого',null,'15',to_date('15.09.17','DD.MM.RR'),'Y' from dual
  union all select '5','3','1','Люберцы','Звездный бульвар','9','12',to_date('17.09.17','DD.MM.RR'),'Y' from dual
  union all select '6','4','1','Лобня','ул. Ленина','19','13',to_date('16.09.17','DD.MM.RR'),'N' from dual
  union all select '7','4','2','Астрахань','ул. 50 лет Октября','10','21',to_date('25.08.17','DD.MM.RR'),'Y' from dual
  union all select '8','4','2','Видное','ул. Пушкинская','34','22',to_date('31.08.17','DD.MM.RR'),'Y' from dual
  union all select '9','4','3','Королёв','ул. М. Горького',null,'22',to_date('18.09.17','DD.MM.RR'),'N' from dual
  union all select '10','5','1','Орел','ул. Б. Хмельницкого','11','17',to_date('21.08.17','DD.MM.RR'),'Y' from dual
  union all select '11','5','2',null,'Звездный бульвар','13','15',to_date('19.09.17','DD.MM.RR'),'N' from dual
  union all select '12','5','3','Одинцово','ул. 50 лет Октября','9','11',to_date('27.08.17','DD.MM.RR'),'N' from dual
  union all select '13','6','2','Белгород','проспект Космонавтов','3','3',to_date('15.09.17','DD.MM.RR'),'Y' from dual
  union all select '14','7','1','Орел','ул. 40 лет Подбеды','12','14',to_date('23.08.17','DD.MM.RR'),'Y' from dual
  union all select '15','7','1','Ростов-на-Дону','бульвар Гая','20','18',to_date('20.08.17','DD.MM.RR'),'N' from dual
  union all select '16','7','1','Ростов-на-Дону','ул. Пушкинская','20','16',to_date('06.09.17','DD.MM.RR'),'N' from dual
  union all select '17','8','2','Воронеж','ул. 50 лет Октября','9','22',to_date('25.08.17','DD.MM.RR'),'Y' from dual
  union all select '18','9','1','Воронеж','ул. Говорова','19','15',to_date('22.08.17','DD.MM.RR'),'Y' from dual
  union all select '19','9','1','Краснодар','бульвар Гая','19','16',to_date('14.09.17','DD.MM.RR'),'Y' from dual
  union all select '20','9','2','Лобня','ул. Есенеа','14','12',to_date('24.08.17','DD.MM.RR'),'N' from dual
  union all select '21','9','3','Севастополь','Октябрьский проспект','16','9',to_date('05.09.17','DD.MM.RR'),'Y' from dual
  union all select '22','10','2',null,'ул. Красная Сосна','2','20',to_date('30.08.17','DD.MM.RR'),'N' from dual
  union all select '23','10','2','Воронеж','Октябрьский проспект','18','3',to_date('06.09.17','DD.MM.RR'),'Y' from dual
  union all select '24','10','2','Воронеж','ул. Лизы Чайкиной','2','9',to_date('03.09.17','DD.MM.RR'),'Y' from dual;
commit;

/*
результирущий запрос для отчёта
1) Лучший адрес отбирается по приоритету фактический(3) > регистрации(2) > домашний(1),
  при наличии нескольких адресов одного приоритета выбирается наиболее полный
  (заполнено больше из перечня атрибутов city-street-house-flat,
  при равенстве по заполненности выбирается последний по дате внесения в базу.
2) Лучший телефон(1) это последний по дате внесения в базу
3) Лучший email(2) это первый по дате внесения в базу
4) Данные по контактам и адресам – не архивные()
*/
select
  c.*,
  addr_body.city,
  addr_body.street,
  addr_body.house,
  addr_body.flat,
  c_phone.str phone,
  c_email.str email
from clients c
  left join (
    select distinct
      client_id,
      first_value(c_info) over(partition by client_id order by created desc) str
    from contacts
    where active = 'Y' and c_type = 1
  ) c_phone on c_phone.client_id = c.id
  left join (
    select distinct
      client_id,
      first_value(c_info) over(partition by client_id order by created) str
    from contacts
    where active = 'Y' and c_type = 2
  ) c_email on c_email.client_id = c.id
  left join (
    select distinct
      client_id,
      first_value(id)
        over(partition by client_id
        order by a_type desc, decode(nvl(city,'0'),'0',0,1)
          + decode(nvl(street,'0'),'0',0,1)
          + decode(nvl(house,'0'),'0',0,1)
          + decode(nvl(flat,'0'),'0',0,1) desc, created desc) id
    from addresses
    where active = 'Y'
  ) addr_head on addr_head.client_id = c.id
  left join (
    select
      a.id,
      a.client_id,
      a.city, a.street, a.house, a.flat
    from addresses a
    where a.active = 'Y'
  ) addr_body on addr_body.client_id = c.id and addr_body.id = addr_head.id
order by 1;

