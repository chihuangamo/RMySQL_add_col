R Notebook
================

Connecting MySQL
================

``` r
library(RMySQL)
con <- dbConnect(MySQL(), 
                 dbname = "taiwan_stats", #資料庫的名字，裡面會有很多張table。在MySQL叫"schema"
                 host = "140.112.153.64", #MySQL所在主機的IP位置。
                 user = "rclass",
                 password = "rrrgogogo") #帳號密碼
```

Getting data from MySQL tables
==============================

View all tables from schema
---------------------------

taiwan\_stats這個schema裡有縣市(county)、鄉鎮市(town)、村里(village)三張表

``` r
result <- dbSendQuery(con, "SHOW TABLES FROM taiwan_stats")
fetch(result, n=-1)
```

    ##             Tables_in_taiwan_stats
    ## 1                           county
    ## 2 health_insurance_population_temp
    ## 3                             town
    ## 4                          village

Getting data from a table
-------------------------

``` r
dbSendQuery(con, "SET NAMES gbk")#資料庫的預設編碼為gbk，不傳送這個query回來會是亂碼
```

    ## <MySQLResult:16646144,0,16>

``` r
result <- dbSendQuery(con, "SELECT * FROM town")#*表示全部欄位
result_df <- fetch(result, n=-1)#送出query後，要把結果從MySQL抓回來
head(result_df) 
```

    ##   county   town
    ## 1 宜蘭縣 三星鄉
    ## 2 宜蘭縣 大同鄉
    ## 3 宜蘭縣 五結鄉
    ## 4 宜蘭縣 冬山鄉
    ## 5 宜蘭縣 壯圍鄉
    ## 6 宜蘭縣 宜蘭市

Updating tables
===============

Reading your data
-----------------

以鄉鎮健保納保人數資料為範例

``` r
health_insurance_population <- read.csv("health_insurance_population.csv", fileEncoding = "utf-8")
```

Writing a temporary table
-------------------------

由於R的dataframe沒法直接插入到MySQL的table，因此必須先在MySQL先將你的data存成一個table，再將他與行政區的table合併。  
"temporary = T" 表示這個table是暫時性的，當你切斷連線後就會刪掉，避免浪費空間。

``` r
dbWriteTable(con, value = health_insurance_population, name = "health_insurance_population_temp", row.names=F, overwrite = T, temporary = T)
```

    ## [1] TRUE

Joining tables
--------------

### Adding a column of the variable

先在行政區的table使用ADD COLUMN增加一個欄位，後面要加入變項的資料類。("INT"表示整數）  
語法："ALTER TABLE \_\_\_ ADD COLUMN \_\_\_ \_\_\_"空格依序填入要改變的table名稱、欄位名稱、資料類型。

``` r
dbSendQuery(con,
            "ALTER TABLE town ADD COLUMN health_insurance_population INT")
```

    ## <MySQLResult:411322480,0,23>

### Joing tables and update

UPDATE表示對table進行變更  
"UPDATE town as p1"  
-&gt; 語法中用p1表示town，較為方便和簡潔  
"LEFT JOIN health\_insurance\_population\_temp as p2  
ON p1.county = p2.county AND p1.town = p2.town"  
-&gt; 語法中用p2表示health\_insurance\_population\_temp、LEFT JOIN時兩個table的county, town欄位要一樣  
"SET p1.health\_insurance\_population = p2.health\_insurance\_population"  
-&gt; UPDATE後p1的health\_insurance\_population欄位會等於p2的health\_insurance\_population欄位  
語法："UPDATE \_\_\_\_ as p1  
LEFT JOIN \_\_\_\_ as p2  
ON p1.\_\_\_\_ = p2.\_\_\_\_ (有需要用更多欄位JOIN再以AND連接)  
SET p1.\_\_\_ = p2.\_\_\_\_\_"

``` r
dbSendQuery(con,
            "UPDATE town as p1 
            LEFT JOIN health_insurance_population_temp as p2 
            ON p1.county = p2.county AND p1.town = p2.town
            SET p1.health_insurance_population = p2.health_insurance_population")
```

    ## <MySQLResult:403963160,0,24>

Getting the updated table
-------------------------

可以看到新增的欄位啦

``` r
result <- dbSendQuery(con, "SELECT * FROM town")
result_df <- fetch(result, n=-1)
head(result_df)
```

    ##   county   town health_insurance_population
    ## 1 宜蘭縣 三星鄉                       20771
    ## 2 宜蘭縣 大同鄉                        5945
    ## 3 宜蘭縣 五結鄉                       38579
    ## 4 宜蘭縣 冬山鄉                       51659
    ## 5 宜蘭縣 壯圍鄉                       23685
    ## 6 宜蘭縣 宜蘭市                       93079

Disconnet
=========

操作完後將與MySQL的連結關掉

``` r
dbDisconnect(con)
```

    ## [1] TRUE
