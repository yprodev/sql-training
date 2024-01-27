# Olympics: 120 years of data



### Training 1

1. Find the total number of summer olympic games
2. Find for each sport, how many games where they played in
3. Compare 1 & 2

```sql
with

t1 as (
	select count(distinct games) as total_summer_games
	from olympics_history oh 
	where season = 'Summer'
),

t2 as (
	select distinct sport, games
	from olympics_history oh 
	where season = 'Summer'
	order by games
),

t3 as (
	select sport, count(games) as nomber_of_games
	from t2
	group by sport
)

select * from t3 join t1 on t1.total_summer_games = t3.nomber_of_games;
```



### Training 2

1. Find the athlete with the biggest number of gold medals
2. There will be several athletes with the same amount of gold medals
3. Rank those with the same amount
4. Filter top 5

```sql
with
	t1 as (
		select name, count(1) as total_medals
		from olympics_history oh 
		where medal = 'Gold'
		group by name
		order by count(1) desc
	),
	
	t2 as (
		select *, dense_rank() over(order by total_medals desc) as rnk
		from t1
	)
	
select *
from t2
where rnk <= 5;
```



### Training 3

List down total gold, silver and bronze medals won by each country

You need `tablefunc` to create to use `crosstab()` creation.

```sql
create extension tablefunc;
```

Query

```sql
 select
	ohnr.region as country,
	oh.medal, 
	count(1) as total_medals
from olympics_history oh
join olympics_history_noc_regions ohnr 
	on oh.noc = ohnr.noc
where medal <> "NA"
group by ohnr.region, medal
order by ohnr.region, medal;
```

Query with `crosstab()`

```sql
select
	country,
	coalesce(gold, 0) as gold,
	coalesce(silver, 0) as silver,
	coalesce(bronze, 0) as bronze
from crosstab('
	select
		ohnr.region as country,
		oh.medal, 
		count(1) as total_medals
	from olympics_history oh
	join olympics_history_noc_regions ohnr 
		on oh.noc = ohnr.noc
	where medal <> ''NA''
	group by ohnr.region, medal
	order by ohnr.region, medal
', 
'values (''Bronze''), (''Gold''), (''Silver'')'
) as result(
	country		varchar,
	bronze 		bigint,
	gold 		bigint,
	silver 		bigint	
)
order by gold desc, silver desc, bronze desc;
```



### Training 4

Identify which country won the most gold, most silver and most bronze medals in each Olympic games.

```sql
with temp as (
	select
		substring(games_country, 1, position(' - ' in games_country) - 1) as games,
		substring(games_country, position(' - ' in games_country) + 3) as country,
		coalesce(gold, 0) as gold,
		coalesce(silver, 0) as silver,
		coalesce(bronze, 0) as bronze
	from crosstab('
		select
			concat(oh.games, '' - '', ohnr.region) as games_country,
			oh.medal, 
			count(1) as total_medals
		from olympics_history oh
		join olympics_history_noc_regions ohnr 
			on oh.noc = ohnr.noc
		where medal <> ''NA''
		group by oh.games, ohnr.region, medal
		order by oh.games, ohnr.region, medal
	', 
	'values (''Bronze''), (''Gold''), (''Silver'')'
	) as result(
		games_country		varchar,
		bronze 				bigint,
		gold 				bigint,
		silver 				bigint	
	)
	order by games_country
)
select
	distinct games,
	concat(
		first_value(country) over(partition by games order by gold desc),
		' - ',
		first_value(gold) over(partition by games order by gold desc)
	) as max_gold,
	concat(
		first_value(country) over(partition by games order by gold desc),
		' - ',
		first_value(silver) over(partition by games order by silver desc)
	) as max_silver,
	concat(
		first_value(country) over(partition by games order by gold desc),
		' - ',
		first_value(bronze) over(partition by games order by bronze desc)
	) as max_bronze
from temp
order by games;

```





