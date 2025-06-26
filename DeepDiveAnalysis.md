# Deep Dive Analysis of Wine Reviews

## 1. Wine Varieties that have consistently high rating

### Objective: 
1. Identify Wine varities that are above median price in both Europe and North America.
2. Compare their mean prices and reviews 


### Analysis
1. A wine variety, in order to be highly rated, must be above the median rating in not just one but both the continents.
2. The first step is therefore to find the median points for each continent and compare a variety's mean rating against it.

Finding the continental median

```sql
# Find Mean, Median, Mode for Europe

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
Output:
| Statistic | Value |
|-----------|-------|
| Mean      | 87.89 |
| Median    | 88    |
| Mode      | 87    |


```sql

# Find Mean, Median, Mode for NAmerica

select 'Mean' as Statistic, round(avg(points),2) as 'Value' 
from wine_reviews

union all

select 'Median', round(avg(points),0) as 'Value' from (
	select points, @rownum:= @rownum+1 as 'row_number', @total_rows:= @rownum
    from wine_reviews m, (select @rownum:=0) r
    where m.points is not null and m.continent = 'North America'
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
Output:

| Statistic | Value |
|-----------|-------|
| Mean      | 87.89 |
| Median    | 88    |
| Mode      | 87    |


Storing this information in a separate table - wine_superlatives, for easy access.

```sql
# Finding the Wine varieties with median greater than continental median.
with cte_europe as (
select  a.variety as Europe_Variety, round(avg(a.points),2) Europe_avg, avg(b.value) as Europe_med, 
count(*) Europe_count, round(avg(prices),2) as Europe_price,
case 
	when round(avg(a.points),2) >= avg(b.value) then 'y'
    else 'n' end as Above_Europe_Median
from wine_reviews a join wine_superlatives b 
on a.continent = b.continent 
and b.statistic ='Median'
and a.continent ='Europe'
group by a.continent, a.variety
having count(*) > 100
order by round(avg(a.points),2) desc
)
,

cte_NAmerica as (
select  a.variety as NAmerica_Variety, round(avg(a.points),2) NAmerica_avg, avg(b.value) as NAmerica_med,
count(*) NAmerica_count, round(avg(prices),2) as NAmerica_price,
case 
	when round(avg(a.points),2) >= avg(b.value) then 'y'
    else 'n' end as Above_North_American_Median
from wine_reviews a join wine_superlatives b 
on a.continent = b.continent 
and b.statistic ='Median'
and a.continent ='North America'
group by a.continent, a.variety
having count(*) > 100
order by round(avg(a.points),2) desc
) 
,
cte_Europe_NAmerica as (
select * from  cte_europe e left join cte_NAmerica n on e.Europe_Variety = n.NAmerica_Variety
union
select * from  cte_europe e right join cte_NAmerica n on e.Europe_Variety = n.NAmerica_Variety
)

select Europe_Variety,NAmerica_avg, Europe_avg, 
	NAmerica_med AS NAmerica_median, Europe_med as Europe_median, 
    NAmerica_price, Europe_price,
    round(NAmerica_price/NAmerica_avg,2) as NAmerican_PPR,
    round(Europe_price/Europe_avg,2) as Europe_PPR,
	NAmerica_count as NA_reviews, Europe_count as Europe_reviews
from cte_Europe_NAmerica
where Europe_Variety is not null
	and NAmerica_Variety is not null
	and Above_North_American_Median ='y'
	and Above_Europe_Median ='y'
order by NAmerica_avg desc, Europe_avg desc

```
Output:
![Q1 Output](images/Q1output.png)

### Insights:
1. All 4 varieties are rated above their local median in NA & EU, they are universally liked. 
2. BSRW - is 35% lesser in Europe, could be explored for export to the NA due to this price difference.
3. Syrah – Same rating in both continents, but in Europe it is on the expensive side. This could be a due to higher production cost. 
4. Malbec – Europe has better quality and lower price
5. Pinot Noir - price is stable across regions


## 2. Ranking Wineries on Prestige Index

### Objective:
1. Create a measure to gauge each Winery based on its rating, price and score
2. Use this measure to indentify top wineries in each continent.

### Analysis:
1. A winery can be in 2 different continents, it can a prestige index for each continent. There are 36 such Wineries in the dataset with US & EU as locations
2. US has 4800+ wineries & Europe has 8000+ wineries. Not all wineries can be used in this study. 
Let us narrow down to wineries with the greater appeal - wineries with >10 reviews.
Below is a winery distribution profile with NA as an example 
Finding the distribution of the wineries by review counts.

| **Bins**   | **1** | **1-9** | **10-99** | **100-499** | **>=500** | **Total** |
|--------------|--------|--------|-----------|-------------|-----------|-----------|
| **Review #** | 844 | 2334	| 1583  | 60      | 0      | 4821      |
| **Review %** | 18 | 48  | 33     | 1        | 0      | 100       |

~70% of the wineries have less than 10 reviews. 
For this case study we will only consider wineries with reviews >=10 in a continent.



