<h1 align="center"> Decoding Game Behavior using SQL</h1>

Welcome to the world of gaming analytics, where every click, move, and decision tells a story. This project is a deep dive into the rich dataset of player interactions, uncovering insights that enhance gameplay and create engaging experiences. It's more than just data—it's about understanding how players navigate challenges, overcome obstacles, and celebrate their victories.

# Key Objectives
This project seeks to analyze and interpret player data to drive meaningful improvements in game design. The primary goals include:

- Player Engagement: Understand how players interact with the game, track their progression, and pinpoint engagement trends.
- Behavioral Insights: Analyze key metrics like kill counts, headshots, and level completion rates to identify what keeps players coming back.
- Retention Strategies: Use data-driven insights to design more immersive, rewarding, and challenging gaming experiences.

# Data Exploration
Using advanced analytics and SQL techniques, this project examines a wide array of player data:

- Interactions: Player actions, moves, and decisions across multiple game levels.
- Performance Metrics: Statistics that define success and challenges, such as skill efficiency and progression rates.
- Behavioral Patterns: Trends and anomalies that highlight key areas for improvement in gameplay design.

# Impact
The insights derived from this project are used to:

- Enhance player retention by identifying key factors that keep players engaged.
- Inform game design decisions, optimizing levels and challenges for maximum enjoyment.
- Create a more immersive experience, ensuring that every action, decision, and challenge in the game is rewarding.

# Conclusion
Through the data, we're not just improving the game; we're enriching the player experience. By understanding player behavior, we aim to create a gaming world where every move matters and every challenge is rewarding.


### Solved 16 distinct questions, uncovering patterns and insights using SQL concepts such as subqueries, joins, stored procedures, aggregate functions, CTEs, and window functions.

```sql
use game_analysis;

alter table pd modify L1_Status varchar(30);
alter table pd modify L2_Status varchar(30);
alter table pd modify P_ID int primary key;
alter table pd drop myunknowncolumn;

alter table ld drop myunknowncolumn;
alter table ld change timestamp start_datetime datetime;
alter table ld modify Dev_Id varchar(10);
alter table ld modify Difficulty varchar(15);
alter table ld add primary key(P_ID,Dev_id,start_datetime);

-- pd (P_ID,PName,L1_status,L2_Status,L1_code,L2_Code)
select * from player_details

-- ld (P_ID,Dev_ID,start_time,stages_crossed,level,difficulty,kill_count,
-- headshots_count,score,lives_earned)
select * from level_details2
```

### Q1) Extract P_ID,Dev_ID,PName and Difficulty_level of all players at level 0
```sql
select
	pd.P_ID as Player_ID, 
	ld.Dev_ID as Device_ID, 
	pd.PName as Player_Name, 
	ld.difficulty AS Difficulty_level
from player_details pd
join level_details2 ld ON pd.P_ID = ld.P_ID
where ld.level = 0;
```


### Q2) Find Level1_code wise Avg_Kill_Count where lives_earned is 2 and atleast 3 stages are crossed.
```sql
select 
	pd.L1_code as Level1_code, 
	avg(ld.kill_count) as Avg_Kill_Count 
from level_details2 ld
join player_details pd on pd.P_ID = ld.P_ID
where Lives_Earned = 2 and Stages_crossed >= 3
group by L1_code;
```

### Q3) Find the total number of stages crossed at each diffuculty level where for Level2 with players use zm_series devices. Arrange the result in decsreasing order of total number of stages crossed.
```sql
select 
	Difficulty as Difficulty_level, 
	sum(Stages_crossed) as Total_Stages_Crossed 
from level_details2 
where level = 2 and Dev_ID LIKE 'zm%'
group by Difficulty
order by Total_Stages_Crossed desc;
```

### Q4) Extract P_IDand the total number of unique dates for those players  who have played games on multiple days.
```sql
select 
	P_ID, 
	count(distinct cast(timestamp as date)) as Total_Unique_Dates  
from level_details2
group by P_ID
having count(distinct cast(timestamp as date)) > 1
```

### Q5) Find P_ID and level wise sum of kill_counts where kill_count is greater than  avg kill count for the Medium difficulty.
```sql
select
	P_ID, 
	level, 
	sum(kill_count) as Total_Kill_Count
from level_details2
where difficulty = 'Medium'
and kill_count > (select avg(kill_count) from level_details2 where Difficulty = 'Medium')
group by P_ID, level;
```

### Q6)  Find Level and its corresponding Level code wise sum of lives earned excluding level 0. Arrange in asecending order of level.
```sql
select
	ld.level as Level,
	pd.L1_code as Level1_Code,
	pd.L2_code as Level2_Code,
	sum(ld.lives_earned) as Total_Lives_Earned
from player_details pd
join level_details2 ld on pd.P_ID = ld.P_ID
where ld.level > 0
group by ld.level, pd.L1_Code, pd.L2_code
order by ld.level ASC;
```

### Q7) Find Top 3 score based on each dev_id and Rank them in increasing order using Row_Number. Display difficulty as well. 
```sql
with Top_3_Score as (
	select 
		Dev_ID,
		Difficulty,
		Score,
		ROW_NUMBER() OVER(PARTITION BY Dev_ID ORDER BY Score asc) as Row_Num
	from level_details2
)
select
	Dev_ID,
	Difficulty,
	Score,
	Row_Num
from Top_3_Score
where Row_Num <= 3
order by Dev_ID, Row_Num;
```

### Q8) Find first_login datetime for each device id
```sql
select 
	Dev_ID,
	min(TimeStamp) as First_Login
from level_details2 
group by Dev_ID
```

### Q9) Find Top 5 score based on each difficulty level and Rank them in  increasing order using Rank. Display dev_id as well.
```sql
with Top_5_score as (
	select
		Difficulty,
		Dev_ID,
		Score,
		RANK() OVER(PARTITION BY Difficulty ORDER BY SCORE ASC) AS Rank_Score
	from level_details2
)
select
	Difficulty,
	Dev_ID,
	Score,
	Rank_Score
from Top_5_score
where Rank_Score <=5
order by Difficulty, Rank_Score
```

### Q10) Find the device ID that is first logged in(based on start_datetime) for each player(p_id). Output should contain player id, device id and first login datetime.
```sql
with RankedLogins as (
    select
        P_ID,
        Dev_ID,
        TimeStamp as first_login_datetime,
        ROW_NUMBER() OVER (PARTITION BY P_ID ORDER BY TimeStamp) as login_rank
    from level_details2
)
select
    P_ID,
    Dev_ID,
    first_login_datetime
from RankedLogins
where login_rank = 1;
```

### Q11) For each player and date, how many kill_count played so far by the player. That is, the total number of games played by the player until that date.

#### a) window function
```sql
with Player_Game_Summary as (
	select
		P_ID,
		cast(TimeStamp as date) as game_date,
		sum(Kill_count) as Kill_Count
	from level_details2
	group by P_ID, cast(TimeStamp as date)
),
Player_Total_Game_Summary as(
	select
		P_ID,
		game_date,
		Kill_Count,
		sum(Kill_count) over(PARTITION BY P_ID ORDER BY game_date) as Total_Kill_Count_So_Far,
		sum(1) over(PARTITION BY P_ID ORDER BY game_date) as  Total_Game_Played
	from Player_Game_Summary
)
select
	P_ID,
	game_date,
	Kill_Count, 
	Total_Kill_Count_So_Far,
	Total_Game_Played
from Player_Total_Game_Summary
order by P_ID, game_date
```

#### b) without window function
```sql
select 
	ld.P_ID,
    convert(date, ld.TimeStamp) as game_date,
	sum(Kill_Count) as Kill_Count, 
    (
        select sum(ld2.kill_count)
        from level_details2 ld2
        where ld2.P_ID = ld.P_ID
        and convert(date, ld2.TimeStamp) <= convert(date, ld.TimeStamp)
    ) as Total_Kill_Count_So_far
from
    level_details2 ld
group by
    ld.P_ID,
    convert(date, ld.TimeStamp)
order by
    ld.P_ID,
    game_date;
```

### Q12) Find the cumulative sum of stages crossed over a start_datetime.
```sql
select
    TimeStamp,
    stages_crossed,
    sum(stages_crossed) over (order by TimeStamp) as Cumulative_Stages_Crossed
from level_details2
```

### Q13) Find the cumulative sum of an stages crossed over a start_datetime for each player id but exclude the most recent start_datetime.
```sql
select 
    ld.P_ID,
    max(ld.TimeStamp) as TimeStamp,
    sum(ld.stages_crossed) as cumulative_stages
from level_details2 ld
where ld.TimeStamp < (select max(TimeStamp) from level_details2 where P_ID = ld.P_ID)
group by ld.P_ID;
```

### Q14) Extract top 3 highest sum of score for each device id and the corresponding player_id.
```sql
with Top_3_score as (
    select
        Dev_ID,
        P_ID,
        sum(score) as total_score,
        ROW_NUMBER() OVER (PARTITION BY Dev_ID ORDER BY sum(score) desc) as Row_Num
    from level_details2
    GROUP BY Dev_ID, P_ID
)
select
    Dev_ID,
    P_ID,
    total_score
from Top_3_score
where Row_Num <= 3
order by Dev_ID, total_score desc;
```

### Q15) Find players who scored more than 50% of the avg score scored by sum of scores for each player_id.
```sql
with PlayerTotalScore as (
    select 
		P_ID, 
		sum(Score) as total_score
    from level_details2
    group by P_ID
)
select 
	P_ID
from PlayerTotalScore
where total_score > (select avg(total_score) * 0.5 from PlayerTotalScore);
```

### Q16) Create a stored procedure to find top n headshots_count based on each dev_id and Rank them in increasing order using Row_Number. Display difficulty as well.
```sql
create procedure TopNHeadshotsCount (@N int)
as
begin
	set nocount on;
	with RankedHeadshots as(
        select 
            P_ID,
            Dev_ID,
            headshots_count,
            difficulty,
            ROW_NUMBER() OVER(PARTITION BY Dev_ID ORDER BY headshots_count asc) as HeadshotsRank
        from level_details2
    )
    select 
		HeadshotsRank,
        P_ID,
        Dev_ID,
        headshots_count,
        difficulty
    from RankedHeadshots
    where HeadshotsRank <= @N;
end;

--run this following to execute procedure
exec TopNHeadshotsCount @N = 5;	
```

### Q17) Create a function to return sum of Score for a given player_id.
```sql
create function dbo.GetTotalScoreForPlayer( @player_id int)
returns int 
as
begin
    declare @total_score int;
    select @total_score = sum(score)
    from level_details2
    where P_ID = @player_id;
    return @total_score;
end;

--run this following to execute the following with passing values
select dbo.GetTotalScoreForPlayer(211) as TotalScoreForPlayer;
```
