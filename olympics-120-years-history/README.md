# Olympics: 120 years of data



### Training 1

1. Find the total number of summer olympic games
2. Find for each sport, how many games where they played in
3. Compare 1 & 2

```sql
 with t1 as
	(select count(distinct games) as total_summer_games
     from  olympics_history
     where season = 'Summer')
```







