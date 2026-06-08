CREATE TABLE employees(
id integer PRIMARY KEY,
name text NOT NULL,
age integer,
gender text,
department_id integer
);


CREATE TABLE department(
id integer PRIMARY KEY,
name text NOT NULL
);


SELECT*FROM employees;
----------------------------------------演習１

INSERT INTO employees(id,name,age,gender,department_id)
VALUES(1,'伊賀将之',38,'男性',3),
      (2,'山田花子',52,'女性',1),
      (3,'鈴木一郎',23,'男性',2),
      (4,'鈴木一子',44,'女性',1),
      (5,'佐藤次郎',18,'男性',3);


INSERT INTO department(id,name)
VALUES(1,'開発部'),
      (2,'営業部'),
      (3,'管理部');



SELECT*FROM employees;
------------------------------------演習２

SELECT * FROM employees;

SELECT name,age FROM employees WHERE age>25;

SELECT * FROM employees ORDER BY age;

SELECT * FROM employees WHERE age>25 AND gender='男性';

-------------------------------------演習３


UPDATE employees SET name = '坂本龍馬', age = 31 WHERE id = 5;

UPDATE employees SET age = age + 1;

DELETE FROM employees WHERE department_id = 1;

下記で確認する↓
SELECT * FROM employees;
-------------------------------------演習４


