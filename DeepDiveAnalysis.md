# Deep Dive Analysis of Wine Reviews

## 1. Wine Varieties that have consistently high rating

### Objective: 
1. Identify Wine varities that are above median price in both Europe and North America.
2. Compare their mean prices and reviews 

```
sql

### finding Mean, Median, Mode for Europe & NAmerica.

select 'Mean' as Statistic, round(avg(points),2) as 'Value' 
from wine_reviews

union all

select 'Median', round(avg(points),0) as 'Value' from (
	select points, @rownum:= @rownum+1 as 'row_number', @total_rows:= @rownum
    from wine_reviews m, (select @rownum:=0) r
    where m.points is not null and m.continent = 'Europe'
    order by m.points
)as temp 
where temp.row_number in (floor((@total_rows+1)/2),floor((@total_rows+2)/2))

union all

select 'Mode' as Statistic, points as 'Value' from
(
select points, count(*) from wine_reviews group by points order by count(*) desc
limit 1
)t; 

```

```
sql
# Mean Median Mode for NAmerica



```









