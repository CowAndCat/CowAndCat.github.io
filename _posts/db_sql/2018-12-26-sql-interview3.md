---
layout: post
title: SQL笔试题4
category: sql
comments: true
---

## 一、选取出每个部门最高的薪酬
The Employee table holds all employees. Every employee has an Id, a salary, and there is also a column for the department Id.

    +----+-------+--------+--------------+
    | Id | Name  | Salary | DepartmentId |
    +----+-------+--------+--------------+
    | 1  | Joe   | 70000  | 1            |
    | 2  | Henry | 80000  | 2            |
    | 3  | Sam   | 60000  | 2            |
    | 4  | Max   | 90000  | 1            |
    +----+-------+--------+--------------+

The Department table holds all departments of the company.

    +----+----------+
    | Id | Name     |
    +----+----------+
    | 1  | IT       |
    | 2  | Sales    |
    +----+----------+

Write a SQL query to find employees who have the highest salary in each of the departments. For the above tables, Max has the highest salary in the IT department and Henry has the highest salary in the Sales department. Like:

    +------------+----------+--------+
    | Department | Employee | Salary |
    +------------+----------+--------+
    | IT         | Max      | 90000  |
    | Sales      | Henry    | 80000  |
    +------------+----------+--------+

### 1.1 解题思路

先按部门选择薪酬最高的人，然后将结果和部门表进行join得到部门名称。

### 1.2 按部门选择薪酬最高

这个部分有个tricky点：如果单纯地用group by和max，获取的Employee.name会不对，因为那列仍旧按原来的顺序返回。

第一种办法：

    select * from Employee AS tmp where Salary=
    (select max(Salary) from Employee where DepartmentId=tmp.DepartmentId) order by DepartmentId;

第二种办法(自己试验不通过，workbench里直接报错）：

    select Name, Salary, DepartmentId from 
    (select Name, Salary, DepartmentId from Employee order by Salary desc) AS p group by DepartmentId;

### 1.3 join获取部门名称

    select d.Name Department, e.Name Employee, Salary from 
        (select Name, Salary, DepartmentId from Employee tmp where Salary=
            (select max(inn.Salary) from Employee inn where inn.DepartmentId=tmp.DepartmentId) 
        order by Salary asc) e
    join Department d on e.DepartmentId=d.Id;

### 1.4 其他解法

1.先join部门获取部门名，选出每个部门最高薪酬，然后再获取员工的名称。

    SELECT d.Department, e.Name as Employee, e.Salary
    FROM (
        SELECT d.Name AS Department, d.Id AS dId, MAX(e.Salary) AS Salary
        FROM Employee e
        LEFT JOIN Department d
        ON e.DepartmentID = d.Id
        GROUP BY Department, dId
        ) d
    INNER JOIN Employee e
    ON d.Salary = e.Salary
    AND d.dId = e.DepartmentId

因为部门名和ID是对应的，所以这里相当于group by DepartmentId。

2.很直白的一个解法：

    SELECT D.Name AS Department ,E.Name AS Employee ,E.Salary 
    FROM
        Employee E,
        (SELECT DepartmentId,max(Salary) as max FROM Employee GROUP BY DepartmentId) T,
        Department D
    WHERE E.DepartmentId = T.DepartmentId 
      AND E.Salary = T.max
      AND E.DepartmentId = D.id

3.继续不用join：

    SELECT D.Name,A.Name,A.Salary 
    FROM 
        Employee A,
        Department D   
    WHERE A.DepartmentId = D.Id 
      AND NOT EXISTS 
      (SELECT 1 FROM Employee B WHERE B.Salary > A.Salary AND A.DepartmentId = B.DepartmentId)

### 建表语句

    CREATE TABLE `Employee` (
      `ID` int(11) NOT NULL,
      `Name` varchar(45) NOT NULL,
      `Salary` int(11) DEFAULT NULL,
      `DepartmentId` int(11) NOT NULL,
      PRIMARY KEY (`ID`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8

    CREATE TABLE `Department` (
      `ID` int(11) NOT NULL,
      `Name` varchar(45) NOT NULL,
      PRIMARY KEY (`ID`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8
