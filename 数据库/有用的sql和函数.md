
# 有用的sql和函数

## 随机获取记录
EXPLAIN EXTENDED
SELECT
  *
FROM
  `user_info`
ORDER BY RAND()
LIMIT 1 SHOW WARNINGS;


EXPLAIN EXTENDED SELECT t1.* 
FROM `user_info` AS t1 JOIN (SELECT ROUND(RAND() * (SELECT MAX(user_id) FROM `user_info`)) AS user_id) AS t2 
WHERE t1.user_id>= t2.user_id
ORDER BY t1.user_id ASC LIMIT 1;

SHOW WARNINGS;


EXPLAIN EXTENDED SELECT user_info.* 
FROM user_info 
WHERE user_id >= (SELECT ROUND( MAX(user_id) * RAND()) FROM `user_info` ) 
ORDER BY user_id LIMIT 1;

SHOW WARNINGS;

SELECT t1.* 
FROM `table` AS t1 JOIN (SELECT ROUND(RAND() * ((SELECT MAX(id) FROM `table`)-(SELECT MIN(id) FROM `table`))+(SELECT MIN(id) FROM `table`)) AS id) AS t2 
WHERE t1.id >= t2.id 
ORDER BY t1.id LIMIT 1;

参考链接： https://blog.csdn.net/wenniuwuren/article/details/47269197

## 常用函数


-- --  抛弃小数
-- select round(123.4); 
-- -- 向下取整
-- select floor(123.4);
-- -- 向上取整
-- select ceil(123.4);
