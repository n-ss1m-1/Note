好的！下面我系统地总结 MySQL 的基础语法和操作，基于你提供的详细笔记进行整理和补充。

## 📚 一、SQL 通用语法

### 📌 基本规则

sql

```
-- 1. 语句可以单行或多行书写，以分号结尾
SELECT * FROM users;

-- 2. 不区分大小写，建议关键字大写
SELECT name, age FROM users WHERE age > 18;

-- 3. 注释
-- 单行注释（双横线）
# 单行注释（井号）
/*
   多行注释
   可以跨越多行
*/
```

## 📚 二、DDL（数据定义语言）

### 📌 2.1 数据库操作

#### 查询数据库

sql

```
-- 查询所有数据库
SHOW DATABASES;

-- 查询当前所在数据库
SELECT DATABASE();

-- 查询数据库创建语句
SHOW CREATE DATABASE database_name;
```

#### 创建数据库

sql

```
-- 基本语法
CREATE DATABASE database_name;

-- 如果不存在则创建
CREATE DATABASE IF NOT EXISTS database_name;

-- 指定字符集和排序规则
CREATE DATABASE database_name
    DEFAULT CHARSET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

#### 删除数据库

sql

```
-- 删除数据库
DROP DATABASE database_name;

-- 如果存在则删除
DROP DATABASE IF EXISTS database_name;
```

#### 使用数据库

sql

```
USE database_name;
```

### 📌 2.2 表操作

#### 查询表

sql

```
-- 查询当前数据库的所有表
SHOW TABLES;

-- 查询指定表的结构
DESC table_name;
DESCRIBE table_name;

-- 查询指定表的建表语句
SHOW CREATE TABLE table_name;
```

#### 创建表

sql

```
CREATE TABLE table_name (
    field1 datatype [COMMENT '字段注释'],
    field2 datatype [COMMENT '字段注释'],
    ...
    fieldn datatype [COMMENT '字段注释']
) [COMMENT '表注释'];
```

示例：

sql

```
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT COMMENT '用户ID',
    username VARCHAR(50) NOT NULL UNIQUE COMMENT '用户名',
    password VARCHAR(100) NOT NULL COMMENT '密码',
    email VARCHAR(100) COMMENT '邮箱',
    age TINYINT UNSIGNED COMMENT '年龄',
    gender CHAR(1) DEFAULT 'M' COMMENT '性别：M-男，F-女',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间'
) COMMENT '用户表';
```

#### 修改表

sql

```
-- 添加字段
ALTER TABLE table_name
    ADD COLUMN field_name datatype [COMMENT '注释'] [约束];

-- 修改数据类型
ALTER TABLE table_name
    MODIFY COLUMN field_name new_datatype(length);

-- 修改字段名和类型
ALTER TABLE table_name
    CHANGE COLUMN old_field_name new_field_name datatype(length) [COMMENT '注释'];

-- 删除字段
ALTER TABLE table_name
    DROP COLUMN field_name;

-- 修改表名
ALTER TABLE old_table_name
    RENAME TO new_table_name;

-- 添加约束
ALTER TABLE table_name
    ADD CONSTRAINT constraint_name constraint_definition;

-- 删除约束
ALTER TABLE table_name
    DROP CONSTRAINT constraint_name;
```

#### 删除表

sql

```
-- 删除表（删除表结构和数据）
DROP TABLE [IF EXISTS] table_name;

-- 清空表数据（保留表结构）
TRUNCATE TABLE table_name;
```

### 📌 2.3 数据类型

#### 数值类型

表格

|类型|范围|说明|
|---|---|---|
|TINYINT|-128 ~ 127|1 字节|
|SMALLINT|-32768 ~ 32767|2 字节|
|MEDIUMINT|-8388608 ~ 8388607|3 字节|
|INT|-2147483648 ~ 2147483647|4 字节|
|BIGINT|-9223372036854775808 ~ 9223372036854775807|8 字节|
|FLOAT|单精度浮点数|4 字节|
|DOUBLE|双精度浮点数|8 字节|
|DECIMAL(M,D)|精确小数|可指定精度|

示例：

sql

```
age TINYINT UNSIGNED,              -- 无符号，0~255
score DOUBLE(5,2),                 -- 总共5位，小数2位
price DECIMAL(10,2)                -- 精确小数，适合金额
```

#### 字符串类型

表格

|类型|说明|最大长度|
|---|---|---|
|CHAR(n)|定长字符串|255|
|VARCHAR(n)|变长字符串|65535|
|TEXT|长文本|65535|
|MEDIUMTEXT|中等文本|16777215|
|LONGTEXT|长文本|4294967295|
|BLOB|二进制大对象|65535|

示例：

sql

```
name VARCHAR(50),                  -- 变长，节省空间
gender CHAR(1),                    -- 定长，固定长度
content TEXT,                      -- 长文本
image_data LONGBLOB                -- 二进制数据
```

#### 日期时间类型

表格

|类型|格式|说明|
|---|---|---|
|DATE|YYYY-MM-DD|日期|
|TIME|HH:MM:SS|时间|
|DATETIME|YYYY-MM-DD HH:MM:SS|日期时间|
|TIMESTAMP|YYYY-MM-DD HH:MM:SS|时间戳|
|YEAR|YYYY|年份|

示例：

sql

```
birth_date DATE,                   -- 生日
login_time DATETIME,               -- 登录时间
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP  -- 创建时间
```

## 📚 三、DML（数据操作语言）

### 📌 插入数据

sql

```
-- 插入指定字段
INSERT INTO table_name (field1, field2, field3)
VALUES (value1, value2, value3);

-- 插入所有字段
INSERT INTO table_name
VALUES (value1, value2, value3, ...);

-- 批量插入
INSERT INTO table_name (field1, field2)
VALUES
    (value1_1, value2_1),
    (value1_2, value2_2),
    (value1_3, value2_3);

-- 插入查询结果
INSERT INTO table_name (field1, field2)
SELECT field1, field2 FROM another_table WHERE condition;
```

示例：

sql

```
-- 插入单条记录
INSERT INTO users (username, password, email, age)
VALUES ('zhangsan', '123456', 'zhangsan@example.com', 25);

-- 批量插入
INSERT INTO users (username, password, email)
VALUES
    ('lisi', '123456', 'lisi@example.com'),
    ('wangwu', '123456', 'wangwu@example.com'),
    ('zhaoliu', '123456', 'zhaoliu@example.com');
```

### 📌 更新数据

sql

```
-- 基本语法
UPDATE table_name
SET field1 = value1, field2 = value2, ...
WHERE condition;

-- 更新多个字段
UPDATE users
SET age = 30, email = 'newemail@example.com'
WHERE id = 1;

-- 注意：没有WHERE条件会更新所有记录！
UPDATE users SET status = 0;  -- ❌ 危险！
```

### 📌 删除数据

sql

```
-- 删除指定记录
DELETE FROM table_name WHERE condition;

-- 删除所有记录（保留表结构）
DELETE FROM table_name;

-- 清空表（更快，但不能回滚）
TRUNCATE TABLE table_name;

-- 注意：没有WHERE条件会删除所有记录！
DELETE FROM users;  -- ❌ 危险！
```

## 📚 四、DQL（数据查询语言）

### 📌 4.1 基本查询

sql

```
-- 查询所有字段
SELECT * FROM table_name;

-- 查询指定字段
SELECT field1, field2, field3 FROM table_name;

-- 设置别名
SELECT field1 AS alias1, field2 alias2 FROM table_name;

-- 去除重复记录
SELECT DISTINCT field1, field2 FROM table_name;
```

### 📌 4.2 条件查询（WHERE）

sql

```
-- 比较运算符
SELECT * FROM users WHERE age > 18;
SELECT * FROM users WHERE age >= 18 AND age <= 30;
SELECT * FROM users WHERE age != 20;
SELECT * FROM users WHERE age <> 20;

-- BETWEEN 范围查询
SELECT * FROM users WHERE age BETWEEN 18 AND 30;

-- IN 列表匹配
SELECT * FROM users WHERE age IN (18, 20, 25, 30);
SELECT * FROM users WHERE age NOT IN (18, 20);

-- LIKE 模糊查询
SELECT * FROM users WHERE username LIKE '张%';    -- 以张开头
SELECT * FROM users WHERE username LIKE '%三';    -- 以三结尾
SELECT * FROM users WHERE username LIKE '%小%';   -- 包含小
SELECT * FROM users WHERE username LIKE '张_';     -- 张+一个字符

-- NULL 判断
SELECT * FROM users WHERE email IS NULL;
SELECT * FROM users WHERE email IS NOT NULL;

-- 逻辑运算符
SELECT * FROM users WHERE age > 18 AND gender = 'M';
SELECT * FROM users WHERE age > 18 OR age < 10;
SELECT * FROM users WHERE NOT (age > 18);
```

### 📌 4.3 聚合函数

sql

```
-- COUNT 统计数量
SELECT COUNT(*) FROM users;                    -- 统计所有行
SELECT COUNT(id) FROM users;                   -- 统计非NULL的id
SELECT COUNT(DISTINCT age) FROM users;         -- 统计不重复的年龄

-- MAX 最大值
SELECT MAX(age) FROM users;

-- MIN 最小值
SELECT MIN(age) FROM users;

-- AVG 平均值
SELECT AVG(age) FROM users;

-- SUM 求和
SELECT SUM(salary) FROM employees;

-- 组合使用
SELECT
    COUNT(*) AS total_users,
    MAX(age) AS max_age,
    MIN(age) AS min_age,
    AVG(age) AS avg_age,
    SUM(salary) AS total_salary
FROM users;
```

### 📌 4.4 分组查询（GROUP BY）

sql

```
-- 基本分组
SELECT gender, COUNT(*) FROM users GROUP BY gender;

-- 分组 + 聚合
SELECT
    department,
    COUNT(*) AS employee_count,
    AVG(salary) AS avg_salary,
    MAX(salary) AS max_salary
FROM employees
GROUP BY department;

-- HAVING 过滤分组结果
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 5000;

-- WHERE 和 HAVING 的区别
SELECT department, AVG(salary) AS avg_salary
FROM employees
WHERE age > 25                    -- WHERE：分组前过滤
GROUP BY department
HAVING AVG(salary) > 5000;        -- HAVING：分组后过滤
```

### 📌 4.5 排序查询（ORDER BY）

sql

```
-- 单字段排序
SELECT * FROM users ORDER BY age ASC;    -- 升序（默认）
SELECT * FROM users ORDER BY age DESC;   -- 降序

-- 多字段排序
SELECT * FROM users
ORDER BY age DESC, username ASC;

-- 排序 + 限制
SELECT * FROM users
ORDER BY age DESC
LIMIT 10;
```

### 📌 4.6 分页查询（LIMIT）

sql

```
-- 基本分页
SELECT * FROM users LIMIT 10;           -- 前10条
SELECT * FROM users LIMIT 0, 10;        -- 从第0条开始，取10条
SELECT * FROM users LIMIT 10, 10;       -- 从第10条开始，取10条

-- 计算公式
-- 起始索引 = (页码 - 1) * 每页数量
-- 第1页：LIMIT 0, 10
-- 第2页：LIMIT 10, 10
-- 第3页：LIMIT 20, 10

-- 完整示例
SELECT * FROM users
WHERE age > 18
ORDER BY created_at DESC
LIMIT 20, 10;  -- 第3页，每页10条
```

### 📌 4.7 查询执行顺序

sql

```
-- 编写顺序
SELECT 字段列表
FROM 表名
WHERE 条件
GROUP BY 分组字段
HAVING 分组后条件
ORDER BY 排序字段
LIMIT 分页参数;

-- 实际执行顺序
FROM 表名                    -- 1. 确定数据来源
WHERE 条件                   -- 2. 过滤数据
GROUP BY 分组字段            -- 3. 分组
HAVING 分组后条件            -- 4. 过滤分组
SELECT 字段列表              -- 5. 选择字段
ORDER BY 排序字段            -- 6. 排序
LIMIT 分页参数;              -- 7. 分页
```

## 📚 五、DCL（数据控制语言）

### 📌 5.1 用户管理

sql

```
-- 查询用户
USE mysql;
SELECT user, host FROM user;

-- 创建用户
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
CREATE USER 'username'@'%' IDENTIFIED BY 'password';  -- 允许远程访问

-- 修改密码
ALTER USER 'username'@'localhost'
IDENTIFIED WITH mysql_native_password BY 'new_password';

-- 删除用户
DROP USER 'username'@'localhost';
```

### 📌 5.2 权限控制

sql

```
-- 查询权限
SHOW GRANTS FOR 'username'@'localhost';

-- 授予权限
-- 授予所有权限
GRANT ALL PRIVILEGES ON database_name.* TO 'username'@'localhost';

-- 授予特定权限
GRANT SELECT, INSERT, UPDATE, DELETE
ON database_name.table_name
TO 'username'@'localhost';

-- 授予所有数据库所有表的权限
GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost';

-- 刷新权限
FLUSH PRIVILEGES;

-- 撤销权限
REVOKE ALL PRIVILEGES ON database_name.* FROM 'username'@'localhost';
REVOKE SELECT, INSERT ON database_name.table_name FROM 'username'@'localhost';
```

## 📚 六、函数

### 📌 6.1 字符串函数

sql

```
-- CONCAT 拼接字符串
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM users;

-- LOWER/UPPER 大小写转换
SELECT LOWER(username) FROM users;
SELECT UPPER(username) FROM users;

-- LPAD/RPAD 填充
SELECT LPAD('123', 5, '0');  -- 结果：00123
SELECT RPAD('123', 5, '*');  -- 结果：123**

-- TRIM 去除空格
SELECT TRIM('  hello  ');    -- 结果：hello
SELECT LTRIM('  hello');     -- 结果：hello
SELECT RTRIM('hello  ');     -- 结果：hello

-- SUBSTRING 截取子串
SELECT SUBSTRING('hello world', 1, 5);  -- 结果：hello
SELECT SUBSTRING('hello world', 7);     -- 结果：world

-- LENGTH 字符串长度
SELECT LENGTH('hello');  -- 结果：5

-- REPLACE 替换
SELECT REPLACE('hello world', 'world', 'MySQL');  -- 结果：hello MySQL
```

### 📌 6.2 数值函数

sql

```
-- CEIL 向上取整
SELECT CEIL(3.14);  -- 结果：4

-- FLOOR 向下取整
SELECT FLOOR(3.99);  -- 结果：3

-- ROUND 四舍五入
SELECT ROUND(3.14159, 2);  -- 结果：3.14
SELECT ROUND(3.14159);     -- 结果：3

-- MOD 取模
SELECT MOD(10, 3);  -- 结果：1

-- RAND 随机数
SELECT RAND();  -- 返回0~1之间的随机数
SELECT FLOOR(RAND() * 100);  -- 返回0~99的随机整数

-- ABS 绝对值
SELECT ABS(-10);  -- 结果：10

-- POWER 幂运算
SELECT POWER(2, 3);  -- 结果：8
```

### 📌 6.3 日期函数

sql

```
-- CURDATE 当前日期
SELECT CURDATE();  -- 2026-04-20

-- CURTIME 当前时间
SELECT CURTIME();  -- 15:26:00

-- NOW 当前日期时间
SELECT NOW();  -- 2026-04-20 15:26:00

-- YEAR/MONTH/DAY 提取年月日
SELECT YEAR(NOW());   -- 2026
SELECT MONTH(NOW());  -- 4
SELECT DAY(NOW());    -- 20

-- DATE_ADD 日期加减
SELECT DATE_ADD(NOW(), INTERVAL 1 DAY);      -- 加1天
SELECT DATE_ADD(NOW(), INTERVAL 7 DAY);      -- 加7天
SELECT DATE_ADD(NOW(), INTERVAL 1 MONTH);    -- 加1个月
SELECT DATE_ADD(NOW(), INTERVAL -1 DAY);     -- 减1天

-- DATEDIFF 日期差
SELECT DATEDIFF('2026-12-31', '2026-01-01');  -- 结果：364

-- DATE_FORMAT 日期格式化
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');  -- 2026-04-20 15:26:00
SELECT DATE_FORMAT(NOW(), '%Y年%m月%d日');       -- 2026年04月20日

-- TIMESTAMPDIFF 时间差
SELECT TIMESTAMPDIFF(YEAR, '2000-01-01', NOW());   -- 年差
SELECT TIMESTAMPDIFF(MONTH, '2026-01-01', NOW());  -- 月差
SELECT TIMESTAMPDIFF(DAY, '2026-04-01', NOW());    -- 天差
```

### 📌 6.4 流程控制函数

sql

```
-- IF 条件判断
SELECT IF(age >= 18, '成年人', '未成年人') AS age_group FROM users;

-- IFNULL 空值处理
SELECT IFNULL(email, '未填写') AS email FROM users;

-- CASE WHEN 多条件判断
-- 方式1：类似if-else
SELECT
    name,
    CASE
        WHEN age < 18 THEN '未成年'
        WHEN age >= 18 AND age < 60 THEN '成年'
        ELSE '老年'
    END AS age_group
FROM users;

-- 方式2：类似switch
SELECT
    name,
    CASE gender
        WHEN 'M' THEN '男'
        WHEN 'F' THEN '女'
        ELSE '未知'
    END AS gender_text
FROM users;
```

## 📚 七、约束

### 📌 约束类型

表格

|约束|关键字|说明|
|---|---|---|
|非空约束|NOT NULL|字段不能为空|
|唯一约束|UNIQUE|字段值唯一|
|主键约束|PRIMARY KEY|非空且唯一，每表只能一个|
|默认约束|DEFAULT|默认值|
|检查约束|CHECK|检查条件（MySQL 8.0.16+）|
|外键约束|FOREIGN KEY|关联其他表|

### 📌 创建约束

sql

```
-- 创建表时添加约束
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,           -- 主键 + 自增
    username VARCHAR(50) NOT NULL UNIQUE,        -- 非空 + 唯一
    password VARCHAR(100) NOT NULL,              -- 非空
    email VARCHAR(100) DEFAULT 'noemail@example.com',  -- 默认值
    age TINYINT UNSIGNED CHECK (age >= 0 AND age <= 150),  -- 检查约束
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP       -- 默认值
);

-- 修改表添加约束
ALTER TABLE users
ADD CONSTRAINT uk_username UNIQUE (username);

ALTER TABLE users
MODIFY COLUMN email VARCHAR(100) NOT NULL;
```

### 📌 外键约束

sql

```
-- 创建外键
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    order_date DATETIME,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- 指定外键名称和删除/更新行为
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    order_date DATETIME,
    CONSTRAINT fk_user_id
        FOREIGN KEY (user_id)
        REFERENCES users(id)
        ON UPDATE CASCADE      -- 更新级联
        ON DELETE SET NULL     -- 删除设为NULL
);

-- 添加外键
ALTER TABLE orders
ADD CONSTRAINT fk_user_id
FOREIGN KEY (user_id) REFERENCES users(id);

-- 删除外键
ALTER TABLE orders
DROP FOREIGN KEY fk_user_id;
```

### 📌 删除 / 更新行为

表格

|行为|说明|
|---|---|
|NO ACTION / RESTRICT|默认，拒绝删除 / 更新|
|CASCADE|级联，同时删除 / 更新子表数据|
|SET NULL|设置为 NULL|
|SET DEFAULT|设置为默认值（不常用）|

## 📚 八、多表关系和多表查询

### 📌 8.1 多表关系

表格

|关系类型|说明|外键设置|
|---|---|---|
|一对一|表结构拆分|任意一方设置外键 + UNIQUE|
|一对多|一个对多个|多的一方设置外键|
|多对多|多个对多个|建立中间表，两个外键|

示例：

sql

```
-- 一对一：用户和用户详情
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50)
);

CREATE TABLE user_profiles (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT UNIQUE,  -- 一对一：加UNIQUE
    bio TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- 一对多：部门和员工
CREATE TABLE departments (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50)
);

CREATE TABLE employees (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50),
    department_id INT,  -- 多的一方设置外键
    FOREIGN KEY (department_id) REFERENCES departments(id)
);

-- 多对多：学生和课程
CREATE TABLE students (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50)
);

CREATE TABLE courses (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50)
);

CREATE TABLE student_courses (
    student_id INT,
    course_id INT,
    PRIMARY KEY (student_id, course_id),  -- 联合主键
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id) REFERENCES courses(id)
);
```

### 📌 8.2 多表查询

#### 内连接（INNER JOIN）

sql

```
-- 隐式内连接
SELECT u.username, e.name, e.salary
FROM users u, employees e
WHERE u.id = e.user_id;

-- 显式内连接
SELECT u.username, e.name, e.salary
FROM users u
INNER JOIN employees e ON u.id = e.user_id;

-- 多表连接
SELECT u.username, d.name AS dept_name, e.salary
FROM users u
INNER JOIN employees e ON u.id = e.user_id
INNER JOIN departments d ON e.department_id = d.id;
```

#### 左外连接（LEFT JOIN）

sql

```
-- 左表全部保留，右表匹配不到的为NULL
SELECT u.username, e.name, e.salary
FROM users u
LEFT JOIN employees e ON u.id = e.user_id;

-- 查询所有用户，包括没有员工信息的
SELECT u.*, e.*
FROM users u
LEFT JOIN employees e ON u.id = e.user_id
WHERE e.id IS NULL;  -- 找出没有员工信息的用户
```

#### 右外连接（RIGHT JOIN）

sql

```
-- 右表全部保留，左表匹配不到的为NULL
SELECT u.username, e.name, e.salary
FROM users u
RIGHT JOIN employees e ON u.id = e.user_id;
```

#### 自连接

sql

```
-- 查询员工及其上级
SELECT e1.name AS employee, e2.name AS manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id;

-- 查询同一部门的员工
SELECT e1.name AS emp1, e2.name AS emp2, e1.department_id
FROM employees e1
JOIN employees e2 ON e1.department_id = e2.department_id
    AND e1.id < e2.id;  -- 避免重复和自连接
```

#### 联合查询（UNION）

sql

```
-- 合并查询结果（去重）
SELECT username FROM admins
UNION
SELECT username FROM users;

-- 合并查询结果（保留重复）
SELECT username FROM admins
UNION ALL
SELECT username FROM users;

-- 注意：字段数量和类型必须一致
```

### 📌 8.3 子查询（嵌套查询）

#### 标量子查询

sql

```
-- 子查询返回单个值
SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

SELECT * FROM employees
WHERE department_id = (SELECT id FROM departments WHERE name = '技术部');
```

#### 列子查询

sql

```
-- 子查询返回一列
SELECT * FROM employees
WHERE department_id IN (SELECT id FROM departments WHERE location = '北京');

SELECT * FROM employees
WHERE salary > ANY (SELECT salary FROM employees WHERE department_id = 1);

SELECT * FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 1);
```

#### 行子查询

sql

```
-- 子查询返回一行
SELECT * FROM employees
WHERE (salary, department_id) = (SELECT MAX(salary), department_id FROM employees);

SELECT * FROM employees
WHERE (salary, manager_id) IN (
    SELECT salary, manager_id FROM employees WHERE department_id = 1
);
```

#### 表子查询

sql

```
-- 子查询返回多行多列
SELECT e.name, d.name AS dept_name
FROM employees e
JOIN (
    SELECT id, name FROM departments WHERE location = '北京'
) d ON e.department_id = d.id;

-- FROM后的子查询必须起别名
SELECT * FROM (
    SELECT name, salary FROM employees WHERE salary > 5000
) AS high_salary_employees;
```

## 📚 九、实用技巧和最佳实践

### 💡 1. 索引优化

sql

```
-- 创建索引
CREATE INDEX idx_username ON users(username);
CREATE INDEX idx_email ON users(email);

-- 复合索引
CREATE INDEX idx_name_age ON users(username, age);

-- 查看索引
SHOW INDEX FROM users;

-- 删除索引
DROP INDEX idx_username ON users;
```

### 💡 2. 事务控制

sql

```
-- 开启事务
START TRANSACTION;

-- 执行操作
INSERT INTO users (username, password) VALUES ('test', '123');
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;

-- 提交事务
COMMIT;

-- 回滚事务
ROLLBACK;
```

### 💡 3. 视图

sql

```
-- 创建视图
CREATE VIEW user_info AS
SELECT u.id, u.username, u.email, e.name AS employee_name, e.salary
FROM users u
LEFT JOIN employees e ON u.id = e.user_id;

-- 查询视图
SELECT * FROM user_info WHERE salary > 5000;

-- 删除视图
DROP VIEW user_info;
```

### 💡 4. 常用查询优化

sql

```
-- 使用EXPLAIN分析查询
EXPLAIN SELECT * FROM users WHERE username = 'zhangsan';

-- 避免SELECT *
SELECT id, username, email FROM users WHERE age > 18;

-- 使用LIMIT限制结果
SELECT * FROM users ORDER BY created_at DESC LIMIT 100;

-- 使用索引字段查询
SELECT * FROM users WHERE username = 'zhangsan';  -- username有索引
```

## 📚 十、完整示例

### 💡 示例：电商系统数据库设计

sql

```
-- 用户表
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    phone VARCHAR(20),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 商品分类表
CREATE TABLE categories (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    parent_id INT,
    FOREIGN KEY (parent_id) REFERENCES categories(id)
);

-- 商品表
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

-- 订单表
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT NOT NULL,
    order_number VARCHAR(50) NOT NULL UNIQUE,
    total_amount DECIMAL(10, 2) NOT NULL,
    status TINYINT DEFAULT 0 COMMENT '0-待支付,1-已支付,2-已发货,3-已完成',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- 订单详情表
CREATE TABLE order_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

### 💡 示例：常用查询

sql

```
-- 1. 查询用户及其订单数量
SELECT
    u.username,
    u.email,
    COUNT(o.id) AS order_count,
    SUM(o.total_amount) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id
HAVING order_count > 0
ORDER BY total_spent DESC;

-- 2. 查询热销商品
SELECT
    p.name,
    p.price,
    SUM(oi.quantity) AS total_sold,
    SUM(oi.quantity * oi.price) AS total_revenue
FROM products p
JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.id
ORDER BY total_sold DESC
LIMIT 10;

-- 3. 查询最近30天的订单
SELECT
    o.order_number,
    u.username,
    o.total_amount,
    o.status,
    o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY o.created_at DESC;

-- 4. 查询库存不足的商品
SELECT
    name,
    stock,
    price
FROM products
WHERE stock < 10
ORDER BY stock ASC;
```

希望这份 MySQL 基础语法和操作的总结对你有帮助！涵盖了从基础到高级的各个方面，包括实用示例和最佳实践。如有任何问题，随时告诉我。