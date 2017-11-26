# 一些常见的宏

**时间与日期的加减运算（Date&Time Add&Substract）**

```basic
B3: 2000/01/01 01:00:00  

D3: Add days  

E3: Add hours  

F3: Add minutes  

=DATE(YEAR(B3), MONTH(B3), DAY(B3) + $D$3) + TIME(HOUR(B3), MINUTE(B3), 0) + TIME($E$3, $F$3, 0)  
```



