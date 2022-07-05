## Comparative between performance of different data formats on AWS ATHENA
---
OBS:
* CSV contains 100.000 rows - 65.3 MB
* PARQUET contains 2.000.000 rows - 83.8 MB
* ORC contains 2.000.000 rows - 143.8 MB

```sql
-- count all values from table
select
	count(*)
from
	covid_csv
-- Run time: 1.127 sec Data scanned: 65.32 MB

select 
    count(*) 
from 
    covid_parquet
-- Run time: 781 ms Data scanned: -

select 
    count(*) 
from 
    covid_orc
-- Run time: 1.007 sec Data scanned: 5.56 KB

-- select 100 elements from table
select 
    * 
from 
    covid_csv 
limit 100
-- Run time: 1.144 sec Data scanned: 204.33 KB

select 
    * 
from 
    covid_parquet 
limit 100
-- Run time: 1.315 sec Data scanned: 83.56 MB

select 
    * 
from 
    covid_orc
limit 100
-- Run time: 1.133 sec Data scanned: 62.40 MB

-- total cases per million by year, month, local ordered by year desc and month asc
select 
    location, 
    year(cast(date as date)) as year, 
    month(cast(date as date)) as month, 
    cast(sum(total_cases_per_million) as decimal(20,10)) as total_cases_per_million 
from covid_csv 
group by 
    location, 
    year(cast(date as date)), 
    month(cast(date as date))
order by 
    year(cast(date as date)) desc, 
    month(cast(date as date)) asc
-- Run time: 1.887 sec Data scanned: 65.32 MB

select 
    location, 
    year(cast(date as date)) as year, 
    month(cast(date as date)) as month, 
    cast(sum(total_cases_per_million) as decimal(20,10)) as total_cases_per_million 
from covid_parquet 
group by 
    location, 
    year(cast(date as date)), 
    month(cast(date as date))
order by 
    year(cast(date as date)) desc, 
    month(cast(date as date)) asc
-- Run time: 3.207 sec Data scanned: 4.96 MB

select 
    location, 
    year(cast(date as date)) as year, 
    month(cast(date as date)) as month, 
    cast(sum(total_cases_per_million) as decimal(20,10)) as total_cases_per_million 
from covid_orc
group by 
    location, 
    year(cast(date as date)), 
    month(cast(date as date))
order by 
    year(cast(date as date)) desc, 
    month(cast(date as date)) asc
-- Run time: 1.229 sec Data scanned: 9.33 MB

-- total cases from locals that had cases only in 2022
select
	location,
	population,
	sum(cast(total_cases as double)) as total_cases
from
	covid_csv
where
	location in ('Brazil', 'Canada')
	and date between ('2020-01-01') and ('2021-01-01')
group by
	location,
	population
-- Run time: 1.994 sec Data scanned: 65.32 MB

select
	location,
	population,
	sum(total_cases) as total_cases
from
	covid_parquet
where
	location in ('Brazil', 'Canada')
	and date between ('2020-01-01') and ('2021-01-01')
group by
	location,
	population
-- Run time: 1.671 sec Data scanned: 5.52 MB

select
	location,
	population,
	sum(total_cases) as total_cases
from
	covid_orc
where
	location in ('Brazil', 'Canada')
	and date between ('2020-01-01') and ('2021-01-01')
group by
	location,
	population
-- Run time: 746 msc Data scanned: 8.35 MB

-- locals, category from new cases and daily count with ranking based on occurrence from category type by location
select
	location,
	new_cases_category,
	count(*) as qtd,
	rank() OVER (PARTITION BY location
ORDER BY
	count(*) DESC) AS rank
from
	(
	select
		iso_code,
		continent,
		location,
		date,
		case
			when cast(total_cases as double) is null then 0.0
			else cast(total_cases as double)
		end as total_cases,
		case
			when cast(new_cases as double) is null then 0.0
			else cast(new_cases as double)
		end as new_cases,
		case
			when cast(total_deaths as double)  is null then 0.0
			else cast(total_deaths as double)
		end as total_deaths,
		case
			when cast(new_cases as double) > 20 then 'Highest'
			when cast(new_cases as double) > 15 then 'Very High'
			when cast(new_cases as double) > 10 then 'High'
			when cast(new_cases as double) > 7 then 'Medium'
			when cast(new_cases as double) > 4 then 'Low'
			when cast(new_cases as double) > 1 then 'Lowest'
			else 'No data'
		end as new_cases_category
	from
		covid_csv)
group by
	location,
	new_cases_category
order by
	location,
	rank,
	count(*) desc
-- Run time: 1.723 sec Data scanned: 65.32 MB

select
	location,
	new_cases_category,
	count(*) as qtd,
	rank() OVER (PARTITION BY location
ORDER BY
	count(*) DESC) AS rank
from
	(
	select
		iso_code,
		continent,
		location,
		date,
		case
			when total_cases is null then 0.0
			else total_cases
		end as total_cases,
		case
			when new_cases is null then 0.0
			else new_cases
		end as new_cases,
		case
			when total_deaths is null then 0.0
			else total_deaths
		end as total_deaths,
		case
			when new_cases > 20 then 'Highest'
			when new_cases > 15 then 'Very High'
			when new_cases > 10 then 'High'
			when new_cases > 7 then 'Medium'
			when new_cases > 4 then 'Low'
			when new_cases > 1 then 'Lowest'
			else 'No data'
		end as new_cases_category
	from
		covid_parquet)
group by
	location,
	new_cases_category
order by
	location,
	rank,
	count(*) desc
-- Run time: 1.128 sec Data scanned: 2.83 MB

select
	location,
	new_cases_category,
	count(*) as qtd,
	rank() OVER (PARTITION BY location
ORDER BY
	count(*) DESC) AS rank
from
	(
	select
		iso_code,
		continent,
		location,
		date,
		case
			when total_cases is null then 0.0
			else total_cases
		end as total_cases,
		case
			when new_cases is null then 0.0
			else new_cases
		end as new_cases,
		case
			when total_deaths is null then 0.0
			else total_deaths
		end as total_deaths,
		case
			when new_cases > 20 then 'Highest'
			when new_cases > 15 then 'Very High'
			when new_cases > 10 then 'High'
			when new_cases > 7 then 'Medium'
			when new_cases > 4 then 'Low'
			when new_cases > 1 then 'Lowest'
			else 'No data'
		end as new_cases_category
	from
		covid_orc)
group by
	location,
	new_cases_category
order by
	location,
	rank,
	count(*) desc
-- Run time: 984 ms Data scanned: 5.29 MB

-- distinct location with the hightest total death per million
select
	distinct location
from
	covid_csv
where
	total_deaths_per_million = (
	select
		max(total_deaths_per_million)
	from
		covid_csv )
-- Run time: 2.347 sec Data scanned: 130.64 MB

select
	distinct location
from
	covid_parquet
where
	total_deaths_per_million = (
	select
		max(total_deaths_per_million)
	from
		covid_parquet )
-- Run time: 1.796 sec Data scanned: 8.18 MB

select
	distinct location
from
	covid_orc
where
	total_deaths_per_million = (
	select
		max(total_deaths_per_million)
	from
		covid_orc )
-- Run time: 1.087 sec Data scanned: 11.49 MB