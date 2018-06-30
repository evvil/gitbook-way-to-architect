# 常用查询场景

#### 1、按某字段排序后，增加排名字段

比如，用户表中，按照年龄排序后，增加rank字段：

```sql
# 建表
CREATE TABLE user (id int, name varchar(20), age int, gender char(1));
# 测试数据
INSERT INTO user VALUES (1, 'Bob', 25, 'M');
INSERT INTO user VALUES (2, 'Jane', 20, 'F');
INSERT INTO user VALUES (3, 'Jack', 30, 'M');
INSERT INTO user VALUES (4, 'Bill', 32, 'M');
INSERT INTO user VALUES (5, 'Nick', 22, 'M');
INSERT INTO user VALUES (6, 'Kathy', 18, 'F');
INSERT INTO user VALUES (7, 'Steve', 36, 'M');
INSERT INTO user VALUES (8, 'Anne', 25, 'F');
```

查询语句：

```sql
SELECT    name,
          age,
          gender,
          @curRank := @curRank + 1 AS rank
FROM      user, (SELECT @curRank := 0) r
ORDER BY  age;
```

查询结果：

```sql
+-------+------+--------+------+
| name  | age  | gender | rank |
+-------+------+--------+------+
| Kathy | 18   | F      | 1    |
| Jane  | 20   | F      | 2    |
| Nick  | 22   | M      | 3    |
| Bob   | 25   | M      | 4    |
| Anne  | 25   | F      | 5    |
| Jack  | 30   | M      | 6    |
| Bill  | 32   | M      | 7    |
| Steve | 36   | M      | 8    |
+-------+------+--------+------+
8 rows in set (0.02 sec)
```

2、分组后取组内Top N的记录

假如，有成绩表，现在，需要按照课程进行分组，然后取出每门课程成绩最高的学生和分数：

```sql
# 建表
CREATE TABLE `score` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL COMMENT '用户id',
  `course_id` int(11) NOT NULL COMMENT '课程id',
  `score` int(11) NOT NULL COMMENT '分数',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=24 DEFAULT CHARSET=utf8

# 测试数据
INSERT INTO score (user_id, course_id, score) VALUES(1,1,60);
INSERT INTO score (user_id, course_id, score) VALUES(1,2,80);
INSERT INTO score (user_id, course_id, score) VALUES(1,3,90);
INSERT INTO score (user_id, course_id, score) VALUES(1,4,88);
INSERT INTO score (user_id, course_id, score) VALUES(1,5,61);

INSERT INTO score (user_id, course_id, score) VALUES(2,1,45);
INSERT INTO score (user_id, course_id, score) VALUES(2,2,78);
INSERT INTO score (user_id, course_id, score) VALUES(2,3,35);
INSERT INTO score (user_id, course_id, score) VALUES(2,4,89);
INSERT INTO score (user_id, course_id, score) VALUES(2,5,71);

INSERT INTO score (user_id, course_id, score) VALUES(3,1,36);
INSERT INTO score (user_id, course_id, score) VALUES(3,2,98);
INSERT INTO score (user_id, course_id, score) VALUES(3,3,100);
INSERT INTO score (user_id, course_id, score) VALUES(3,4,35);
INSERT INTO score (user_id, course_id, score) VALUES(3,5,70);

INSERT INTO score (user_id, course_id, score) VALUES(3,1,36);
INSERT INTO score (user_id, course_id, score) VALUES(3,2,98);
INSERT INTO score (user_id, course_id, score) VALUES(3,3,100);
INSERT INTO score (user_id, course_id, score) VALUES(3,4,35);
INSERT INTO score (user_id, course_id, score) VALUES(3,5,70);

```

查询语句：


