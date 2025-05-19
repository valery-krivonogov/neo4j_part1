# Домашнее задание
## Сравнение с неграфовыми БД

### Варианты использования

+ Генеалогическое дерево<br/>
Построение и визуализация генеалогического дерева семьи.

+ Система построения маршрута поездки (аналог tutu.ru)  
На основании расписания движения различных видов транспорта формировать маршрут поездки из города А в город Б.

+ Система выявления дропперов<br/>
По данным операций (ребра) по счетам (вершины) реализовать поиск клиентов, выполняющих функцию дробления сумм в целях ухода от налогооблажения 

+ "Сделать из мухи слона"<br/>
Определение маршрута для преобразования исходного слова из четырех букв (МУХА) в конечное слово (СЛОН) путем последовательной замены любой буквы исходного слова, при которой новый набор символов является словом русского языка.
Перечень слов русского языка из четырех букв будут вершинами, между словами, имеющими различие на один символ, будет сформирована связь (ребро).
Список слов: https://russkiyslovar.ru/slova-4-bukvy
Решение задачи - поиск пути из вершины исходного слова к вершине конечного слова.

### Сравнение реализации хранения и манипуляции данными проекта "Генеалогическое дерево" в neo4j и sql базах

Модель БД PostgreSQL

![Data Model](/img/schema_db.jpg)

Создание таблицы

```sql
CREATE SEQUENCE public.pers_id	INCREMENT BY 1 	START 1;

CREATE TABLE public.person (
	id int4 DEFAULT nextval('pers_id'::regclass) NOT NULL,
	last_name varchar(64) NULL,
	first_name varchar(64) NULL,
	middle_name varchar(32) NULL,
	birth_date date NULL,
	father int4 NULL,
	mother int4 NULL,
	CONSTRAINT pk_person PRIMARY KEY (id),
	CONSTRAINT fk_person_reference_person_f FOREIGN KEY (father) REFERENCES public.person(id) ON DELETE RESTRICT ON UPDATE RESTRICT,
	CONSTRAINT fk_person_reference_person_m FOREIGN KEY (mother) REFERENCES public.person(id) ON DELETE RESTRICT ON UPDATE RESTRICT
);
```
Наполнение данными
```sql
INSERT INTO public.person (id,last_name,first_name,middle_name,birth_date,father,mother) VALUES
	 (1,'Иванов','Иван','Иванович','1969-09-22',NULL,NULL),
	 (2,'Иванова','Ирина','Евгеньевна','1970-04-25',NULL,NULL),
	 (3,'Иванова','Ольга','Ивановна','1989-10-11',1,2),
	 (4,'Иванов','Сергей','Иванович','1991-06-17',1,2),
	 (5,'Иванов','Артем','Иванович','2005-02-16',1,2),
	 (6,'Иванов','Александр','Иванович','2015-03-31',1,2);
```

Запрос данных (по мужской линии)
```sql
with recursive ts ( id, father,last_name, first_name, middle_name, birth_date) AS 
( select  p.id, p.father, p.last_name, p.first_name, p.middle_name, p.birth_date  from person p   where p.father IS null  
  union 
  select c.id, c.father, c.last_name, c.first_name, c.middle_name, c.birth_date   from person c, ts s  where s.id= c.father ) 
select * from ts 

id|father|last_name|first_name|middle_name|birth_date|
--+------+---------+----------+-----------+----------+
 1|      |Иванов   |Иван      |Иванович   |1969-09-22|
 2|      |Иванова  |Ирина     |Евгеньевна |1970-04-25|
 3|     1|Иванова  |Ольга     |Ивановна   |1989-10-11|
 4|     1|Иванов   |Сергей    |Иванович   |1991-06-17|
 5|     1|Иванов   |Артем     |Иванович   |2005-02-16|
 6|     1|Иванов   |Александр |Иванович   |2015-03-31|

```

Реализация в Neo4j

Схема данных отсутствует

Формирование графа (заполнение БД)
``` sql
CREATE (p1:Person {	last_name : "Иванов",  first_name : "Иван",      middle_name : "Иванович",   birth_date : "1969-09-22"}) RETURN p1
CREATE (p2:Person {	last_name : "Иванова", first_name : "Ирина",	 middle_name : "Евгеньевна", birth_date : "1970-04-25"}) RETURN p2
CREATE (p3:Person {	last_name : "Иванова", first_name : "Ольга",	 middle_name : "Ивановна",   birth_date : "1989-10-11"}) RETURN p3
CREATE (p4:Person {	last_name : "Иванов",  first_name : "Сергей",    middle_name : "Иванович",   birth_date : "1991-06-17"}) RETURN p4
CREATE (p5:Person {	last_name : "Иванов",  first_name : "Артем",     middle_name : "Иванович",   birth_date : "2005-02-16"}) RETURN p5
CREATE (p6:Person {	last_name : "Иванов",  first_name : "Александр", middle_name : "Иванович",	 birth_date : "2015-03-31"}) RETURN p6

CREATE (p1)-[:father]->(p3)
CREATE (p1)-[:father]->(p4)
CREATE (p1)-[:father]->(p5)
CREATE (p1)-[:father]->(p6)

CREATE (p2)-[:mother]->(p3)
CREATE (p2)-[:mother]->(p4)
CREATE (p2)-[:mother]->(p5)
CREATE (p2)-[:mother]->(p6);
```

Запрос данных
```sql
MATCH p=()-[]->() RETURN p LIMIT 25
```

Результат выполнения
```
╒════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╕
│p                                                                                                                               │
╞════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
│(:Person {birth_date: "1969-09-22",last_name: "Иванов",middle_name: "Иванович",first_name: "Иван"})-[:father]->(:Person {birth_d│
│ate: "1991-06-17",last_name: "Иванов",middle_name: "Иванович",first_name: "Сергей"})                                            │
├────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│(:Person {birth_date: "1969-09-22",last_name: "Иванов",middle_name: "Иванович",first_name: "Иван"})-[:father]->(:Person {birth_d│
│ate: "2005-02-16",last_name: "Иванов",middle_name: "Иванович",first_name: "Артем"})                                             │
├────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│(:Person {birth_date: "1969-09-22",last_name: "Иванов",middle_name: "Иванович",first_name: "Иван"})-[:father]->(:Person {birth_d│
│ate: "2015-03-31",last_name: "Иванов",middle_name: "Иванович",first_name: "Александр"})                                         │
├────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│(:Person {birth_date: "1970-04-25",last_name: "Иванова",middle_name: "Евгеньевна",first_name: "Ирина"})-[:mother]->(:Person {bir│
│th_date: "1989-10-11",last_name: "Иванова",middle_name: "Ивановна",first_name: "Ольга"})                                        │
├────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│(:Person {birth_date: "1970-04-25",last_name: "Иванова",middle_name: "Евгеньевна",first_name: "Ирина"})-[:mother]->(:Person {bir│
│th_date: "1991-06-17",last_name: "Иванов",middle_name: "Иванович",first_name: "Сергей"})                                        │
├────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│(:Person {birth_date: "1970-04-25",last_name: "Иванова",middle_name: "Евгеньевна",first_name: "Ирина"})-[:mother]->(:Person {bir│
│th_date: "2005-02-16",last_name: "Иванов",middle_name: "Иванович",first_name: "Артем"})                                         │
├────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│(:Person {birth_date: "1970-04-25",last_name: "Иванова",middle_name: "Евгеньевна",first_name: "Ирина"})-[:mother]->(:Person {bir│
│th_date: "2015-03-31",last_name: "Иванов",middle_name: "Иванович",first_name: "Александр"})                                     │
├────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│(:Person {birth_date: "1969-09-22",last_name: "Иванов",middle_name: "Иванович",first_name: "Иван"})-[:father]->(:Person {birth_d│
│ate: "1989-10-11",last_name: "Иванова",middle_name: "Ивановна",first_name: "Ольга"})                                            │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```
В виде графа <br/>
![Data Model](/img/data.jpg)


## Выводы

Запрос данных с Neo4j более лаконичен.
Привычнее работать с реляционной БД.
