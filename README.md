# LeetCode
### 175
```
import pandas as pd

def combine_two_tables(person: pd.DataFrame, address: pd.DataFrame) -> pd.DataFrame:
    result = pd.merge(person, address, on='personId', how='left')
    result = result[['firstName', 'lastName', 'city', 'state']]
    return result
```
```
select FirstName, LastName, City, State
from Person left join Address
on Person.PersonId = Address.PersonId
;
```
### 180
```
SELECT DISTINCT
    l1.Num AS ConsecutiveNums
FROM
    Logs l1,
    Logs l2,
    Logs l3
WHERE
    l1.Id = l2.Id - 1
    AND l2.Id = l3.Id - 1
    AND l1.Num = l2.Num
    AND l2.Num = l3.Num
;
```
### 185
```
import pandas as pd
​
def top_three_salaries(employee: pd.DataFrame, department: pd.DataFrame) -> pd.DataFrame:
    
    Employee_Department = employee.merge(department, left_on='departmentId', right_on='id').rename(columns = {'name_y': 'Department'})

    Employee_Department = Employee_Department[['Department', 'departmentId', 'salary']].drop_duplicates()
    
    top_salary = Employee_Department.groupby(['Department', 'departmentId']).salary.nlargest(3).reset_index()
    
    df = top_salary.merge(employee, on=['departmentId', 'salary'])
    
    return df[['Department', 'name', 'salary']].rename(columns = {'name': 'Employee', 'salary': 'Salary'})
```
```
import pandas as pd
​
def top_three_salaries(employee: pd.DataFrame, department: pd.DataFrame) -> pd.DataFrame:

    top_salary = employee[employee.groupby('departmentId').salary.rank(method='dense', ascending=False) <= 3]

    employee_department = top_salary.merge(department, left_on='departmentId', right_on='id')[['name_y', 'name_x', 'salary']]

    return employee_department.rename(columns = {'name_y': 'Department', 'name_x': 'Employee', 'salary': 'Salary'})
```
```
SELECT d.name AS 'Department', 
       e1.name AS 'Employee', 
       e1.salary AS 'Salary' 
FROM Employee e1
JOIN Department d
ON e1.departmentId = d.id 
WHERE
    3 > (SELECT COUNT(DISTINCT e2.salary)
        FROM Employee e2
        WHERE e2.salary > e1.salary AND e1.departmentId = e2.departmentId);
```
```
WITH employee_department AS
    (
    SELECT d.id, 
        d.name AS Department, 
        salary AS Salary, 
        e.name AS Employee, 
        DENSE_RANK()OVER(PARTITION BY d.id ORDER BY salary DESC) AS rnk
    FROM Department d
    JOIN Employee e
    ON d.id = e.departmentId
    )
SELECT Department, Employee, Salary
FROM employee_department
WHERE rnk <= 3
```
### 197
```
import pandas as pd

def rising_temperature(weather: pd.DataFrame) -> pd.DataFrame:
    # Ensure the 'recordDate' column is a datetime type
    weather['recordDate'] = pd.to_datetime(weather['recordDate'])
    
    # Create a copy of the weather DataFrame with a 1 day shift 
    weather_shifted = weather.copy()
    weather_shifted['recordDate'] = weather_shifted['recordDate'] + pd.to_timedelta(1, unit='D')
    
    # Merging the DataFrames on the 'recordDate' column to find consecutive dates
    merged_df = pd.merge(weather, weather_shifted, on='recordDate', suffixes=('_today', '_yesterday'))
    
    # Finding rows where the temperature is greater on the current day compared to the previous day
    result = merged_df[merged_df['temperature_today'] > merged_df['temperature_yesterday']][['id_today']].rename(columns={'id_today': 'Id'})
    
    return result
```
```
import pandas as pd

def rising_temperature(weather: pd.DataFrame) -> pd.DataFrame:
    # Ensure the 'recordDate' column is a datetime type
    weather['recordDate'] = pd.to_datetime(weather['recordDate'])
    
    # Sorting the DataFrame by 'recordDate' to ensure the shift operation works correctly
    weather.sort_values('recordDate', inplace=True)
    
    # Creating new columns for the previous day's temperature and record date
    weather['PreviousTemperature'] = weather['temperature'].shift(1)
    weather['PreviousRecordDate'] = weather['recordDate'].shift(1)
    
    # Filtering the DataFrame to find rows where the temperature is higher 
    # than the previous day and the date is exactly one day more than the previous record date
    result = weather[
        (weather['temperature'] > weather['PreviousTemperature']) & 
        (weather['recordDate'] == weather['PreviousRecordDate'] + pd.Timedelta(days=1))
    ][['id']].rename(columns={'id': 'Id'})
    
    return result
```
```
SELECT 
    w1.id
FROM 
    Weather w1
JOIN 
    Weather w2
ON 
    DATEDIFF(w1.recordDate, w2.recordDate) = 1
WHERE 
    w1.temperature > w2.temperature;
```
```
WITH PreviousWeatherData AS
(
    SELECT 
        id,
        recordDate,
        temperature, 
        LAG(temperature, 1) OVER (ORDER BY recordDate) AS PreviousTemperature,
        LAG(recordDate, 1) OVER (ORDER BY recordDate) AS PreviousRecordDate
    FROM 
        Weather
)
SELECT 
    id 
FROM 
    PreviousWeatherData
WHERE 
    temperature > PreviousTemperature
AND 
    recordDate = DATE_ADD(PreviousRecordDate, INTERVAL 1 DAY);
```
```
SELECT 
    w1.id
FROM 
    Weather w1
WHERE 
    w1.temperature > (
        SELECT 
            w2.temperature
        FROM 
            Weather w2
        WHERE 
            w2.recordDate = DATE_SUB(w1.recordDate, INTERVAL 1 DAY)
    );
```
```
SELECT 
    w2.id
FROM 
    Weather w1, Weather w2
WHERE 
    DATEDIFF(w2.recordDate, w1.recordDate) = 1 
AND 
    w2.temperature > w1.temperature;
```
### 262
```
import pandas as pd

def trips_and_users(trips: pd.DataFrame, users: pd.DataFrame) -> pd.DataFrame:

    # Step 1: Preliminary Check
    if trips.empty or users.empty:
        return pd.DataFrame(columns=["Day", "Cancellation Rate"])

    # Step 2: Prepare Data for Client Merge
    renamed_users_for_clients = users.rename(
        columns={"users_id": "client_id", "banned": "client_banned"}
    )

    # Step 3: Client Merge
    trips_with_clients = trips.merge(
        renamed_users_for_clients, on="client_id", how="left"
    )

    # Step 4: Prepare Data for Driver Merge
    renamed_users_for_drivers = users.rename(
        columns={"users_id": "driver_id", "banned": "driver_banned"}
    )

    # Step 5: Driver Merge
    full_trips = trips_with_clients.merge(
        renamed_users_for_drivers, on="driver_id", how="left"
    )

    # Step 6: Filtering
    filtered_trips = full_trips[
        (full_trips["client_banned"] == "No")
        & (full_trips["driver_banned"] == "No")
        & (full_trips["request_at"].between("2013-10-01", "2013-10-03"))
    ]

    # Step 7: Calculate Cancellation Rate
    result = filtered_trips.groupby("request_at").apply(
        lambda group: pd.Series(
            {
                "Cancellation Rate": round(
                    (group["status"] != "completed").sum() / len(group), 2
                )
            }
        )
    )

    # Step 8: Result Presentation
    if result.empty:
        return pd.DataFrame(columns=["Day", "Cancellation Rate"])
    else:
        return result.reset_index().rename(columns={"request_at": "Day"})
```
```
import pandas as pd

def trips_and_users(trips: pd.DataFrame, users: pd.DataFrame) -> pd.DataFrame:
    # Step 1: Data Verification
    # Check if either `trips` or `users` DataFrames are empty.
    # If so, return a DataFrame with columns "Day" and "Cancellation Rate" without any data.
    if trips.empty or users.empty:
        return pd.DataFrame(columns=["Day", "Cancellation Rate"])

    # Step 2: Isolating Banned Users
    # Using boolean indexing on the `users` DataFrame, extract the IDs (`users_id`) of users who are banned.
    banned_users_ids = users[users["banned"] == "Yes"]["users_id"]

    # Step 3: Filtering Relevant Trip Data
    # Remove rows from `trips` DataFrame that have `client_id` or `driver_id` matching the IDs of banned users.
    # Retain rows in the `trips` DataFrame that have `request_at` dates within the range of '2013-10-01' to '2013-10-03'.
    selected_trips = trips[
        (~trips["client_id"].isin(banned_users_ids))
        & (~trips["driver_id"].isin(banned_users_ids))
        & (trips["request_at"].between("2013-10-01", "2013-10-03"))
    ]

    # Step 4: Aggregating Data
    # Group the data in the `selected_trips` DataFrame based on the `request_at` column.
    # For each group, calculate the cancellation rate by determining the ratio of non-completed trips to the total number of trips, rounding to two decimal places.
    aggregated_result = selected_trips.groupby("request_at").apply(
        lambda group: pd.Series(
            {
                "Cancellation Rate": round(
                    (group["status"] != "completed").sum() / len(group), 2
                )
            }
        )
    )

    # Step 5: Result Compilation
    # If the `aggregated_result` DataFrame isn't empty, reset its index and rename the `request_at` column to 'Date'.
    # If it's empty, return a DataFrame with columns "Date" and "Cancellation Rate" without any data.
    if aggregated_result.empty:
        return pd.DataFrame(columns=["Day", "Cancellation Rate"])
    else:
        return aggregated_result.reset_index().rename(columns={"request_at": "Day"})
```
```
import pandas as pd

def trips_and_users(trips: pd.DataFrame, users: pd.DataFrame) -> pd.DataFrame:
    # Step 1: Initial Check
    if trips.empty or users.empty:
        return pd.DataFrame(columns=["Day", "Cancellation Rate"])

    # Step 2: Date-based Filtering
    filtered_trips = trips[trips["request_at"].between("2013-10-01", "2013-10-03")]

    # Step 3: Merge with Non-Banned Clients
    trips_with_clients = filtered_trips.merge(
        users.loc[users["banned"] == "No", ["users_id"]],
        left_on="client_id",
        right_on="users_id",
        how="inner",
    )

    # Step 4: Merge with Non-Banned Drivers
    trip_status = trips_with_clients.merge(
        users.loc[users["banned"] == "No", ["users_id"]],
        left_on="driver_id",
        right_on="users_id",
        how="inner",
    )

    # Step 5: Calculate Day-wise Cancellation Rate
    result = trip_status.groupby("request_at").apply(
        lambda group: pd.Series(
            {"Cancellation Rate": round(
                 (group["status"] != "completed").sum() / len(group), 2
                 )
             }
        )
    )

    # Step 6: Format and Return the Result
    if result.empty:
        return pd.DataFrame(columns=["Day", "Cancellation Rate"])
    else:
        return result.reset_index().rename(columns={"request_at": "Day"})
```
```
SELECT 
  request_at AS Day, 
  ROUND(
    SUM(status != 'completed') / COUNT(*), 
    2
  ) AS 'Cancellation Rate' 
FROM 
  Trips 
  LEFT JOIN Users AS Clients ON Trips.client_id = Clients.users_id 
  LEFT JOIN Users AS Drivers ON Trips.driver_id = Drivers.users_id 
WHERE 
  Clients.banned = 'No' 
  AND Drivers.banned = 'No' 
  AND request_at BETWEEN '2013-10-01' 
  AND '2013-10-03' 
GROUP BY 
  Day
```
```
SELECT 
  request_at AS Day, 
  ROUND(
    SUM(status != 'completed') / COUNT(status), 
    2
  ) AS 'Cancellation Rate' 
FROM 
  Trips 
WHERE 
  request_at BETWEEN '2013-10-01' 
  AND '2013-10-03' 
  AND driver_id NOT IN (
    SELECT 
      users_id 
    FROM 
      Users 
    WHERE 
      banned = 'Yes'
  ) 
  AND client_id NOT IN (
    SELECT 
      users_id 
    FROM 
      Users 
    WHERE 
      banned = 'Yes'
  ) 
GROUP BY 
  Day
```
```
WITH TripStatus AS (
  SELECT 
    Request_at AS Day, 
    T.status != 'completed' AS cancelled 
  FROM 
    Trips T 
    JOIN Users C ON Client_Id = C.Users_Id 
    AND C.Banned = 'No' 
    JOIN Users D ON Driver_Id = D.Users_Id 
    AND D.Banned = 'No' 
  WHERE 
    Request_at BETWEEN '2013-10-01' 
    AND '2013-10-03'
) 
SELECT 
  Day, 
  ROUND(
    SUM(cancelled) / COUNT(cancelled), 
    2
  ) AS 'Cancellation Rate' 
FROM 
  TripStatus 
GROUP BY 
  Day;
```
### 550
```
import pandas as pd

def gameplay_analysis(activity: pd.DataFrame) -> pd.DataFrame:
    # Step 1: Find the first login date for each player
    first_login = activity.groupby('player_id')['event_date'].min().reset_index()
    
    # Step 2: Create a new column for the day before each event_date in the original DataFrame
    activity['day_before_event'] = activity['event_date'] - pd.to_timedelta(1, unit='D')
    
    # Step 3: Merge the dataframes to find rows where player logged in a day after their first login
    merged_df = activity.merge(first_login, on='player_id', suffixes=('_actual', '_first'))
    
    # Step 4: Find the rows where the actual event date matches the day after the first login date
    consecutive_login = merged_df[merged_df['day_before_event'] == merged_df['event_date_first']]
    
    # Step 5: Calculate the fraction of players that logged in again on the day after their first login
    fraction = round(consecutive_login['player_id'].nunique() / activity['player_id'].nunique(), 2)
    
    # Step 6: Create a dataframe to hold the output
    output_df = pd.DataFrame({'fraction': [fraction]})
    
    return output_df
```
```
SELECT
  ROUND(
    COUNT(A1.player_id)
    / (SELECT COUNT(DISTINCT A3.player_id) FROM Activity A3)
  , 2) AS fraction
FROM
  Activity A1
WHERE
  (A1.player_id, DATE_SUB(A1.event_date, INTERVAL 1 DAY)) IN (
    SELECT
      A2.player_id,
      MIN(A2.event_date)
    FROM
      Activity A2
    GROUP BY
      A2.player_id
  );
```
```
WITH first_logins AS (
  SELECT
    A.player_id,
    MIN(A.event_date) AS first_login
  FROM
    Activity A
  GROUP BY
    A.player_id
), consec_logins AS (
  SELECT
    COUNT(A.player_id) AS num_logins
  FROM
    first_logins F
    INNER JOIN Activity A ON F.player_id = A.player_id
    AND F.first_login = DATE_SUB(A.event_date, INTERVAL 1 DAY)
)
SELECT
  ROUND(
    (SELECT C.num_logins FROM consec_logins C)
    / (SELECT COUNT(F.player_id) FROM first_logins F)
  , 2) AS fraction;
```
### 577
```
import pandas as pd

def employee_bonus(employee: pd.DataFrame, bonus: pd.DataFrame) -> pd.DataFrame:
    # Merge Employee and Bonus tables using a left join
    result_df = pd.merge(employee, bonus, on='empId', how='left')

    # Filter rows where bonus is less than 1000 or missing
    result_df = result_df[(result_df['bonus'] < 1000) | result_df['bonus'].isnull()]

    # Select "name" and "bonus" columns
    result_df = result_df[['name', 'bonus']]

    return result_df
```
```
SELECT
    Employee.name, Bonus.bonus
FROM
    Employee
        LEFT JOIN
    Bonus ON Employee.empid = Bonus.empid
WHERE
    bonus < 1000 OR bonus IS NULL
;
```
### 584
```
SELECT name FROM customer WHERE referee_id <> 2 OR referee_id IS NULL;
```
```
SELECT name FROM customer WHERE referee_id != 2 OR referee_id IS NULL;
```
### 595
```
import pandas as pd

def big_countries(world: pd.DataFrame) -> pd.DataFrame:
    df = world[(world['area'] >= 3000000) | (world['population'] >= 25000000)]
    return df[['name', 'population', 'area']]
```
```
SELECT
    name, population, area
FROM
    world
WHERE
    area >= 3000000 OR population >= 25000000
;
```
### 601
```
import pandas as pd

def human_traffic(stadium: pd.DataFrame) -> pd.DataFrame:

    df = stadium[stadium['people'] >= 100]
    
    df['flag'] = ((df['id'].diff() == 1) & (df['id'].diff().shift(1) == 1))
    
    df = df[(df['flag'] == True)| (df['flag'].shift(-1) == True) | (df['flag'].shift(-2) == True)]
    
    rreturn df.loc[:, df.columns != 'flag'].sort_values(by='visit_date')
```
```
def human_traffic(stadium: pd.DataFrame) -> pd.DataFrame:

    stadium = stadium[stadium['people'] >= 100]

    stadium['rnk'] = range(len(stadium))

    stadium['island'] = stadium.id - stadium.rnk

    stadium['island_cnt'] = stadium.groupby(['island'], as_index=False).id.transform('count')

    return stadium[stadium['island_cnt'] >= 3][['id', 'visit_date', 'people']].sort_values(by='visit_date')
```
```
SELECT 
    DISTINCT a.*
FROM 
    stadium AS a, stadium AS b, stadium AS c
WHERE
     a.people >= 100 AND b.people >= 100 AND c.people >= 100
AND 
    (
       (a.id - b.id = 1 AND b.id - c.id = 1)
    OR (c.id - b.id = 1 AND b.id - a.id = 1)
    OR (b.id - a.id = 1 AND a.id - c.id = 1)
    )
ORDER BY visit_date
```
```
WITH base AS (
        SELECT *,
            LEAD(id, 1) OVER(ORDER BY id) AS next_id,
            LEAD(id, 2) OVER(ORDER BY id) AS second_next_id,
            LAG(id, 1) OVER(ORDER BY id) AS last_id,
            LAG(id, 2) OVER(ORDER BY id) AS second_last_id
        FROM stadium
        WHERE people >= 100 
        )
SELECT DISTINCT id, visit_date, people
FROM base 
WHERE (next_id - id = 1 AND id - last_id = 1)
    OR (second_next_id - next_id = 1 AND next_id - id = 1)
    OR (id - last_id = 1 AND last_id - second_last_id = 1)
ORDER BY visit_date
```
```
WITH stadium_with_rnk AS
(
    SELECT id, visit_date, people, rnk, (id - rnk) AS island
    FROM (
        SELECT id, visit_date, people, RANK() OVER(ORDER BY id) AS rnk
        FROM Stadium
        WHERE people >= 100) AS t0
)
SELECT id, visit_date, people 
FROM stadium_with_rnk
WHERE island IN (SELECT island 
                 FROM stadium_with_rnk 
                 GROUP BY island 
                 HAVING COUNT(*) >= 3)
ORDER BY visit_date
```
### 602
```
import pandas as pd
def most_friends(request_accepted: pd.DataFrame) -> pd.DataFrame:
    
    values = pd.concat([request_accepted["requester_id"], request_accepted["accepter_id"]]).to_frame('id')

    df = values.groupby('id', as_index=False).agg(num=('id', 'count')).sort_values('num', ascending=False).head(1)

    return df
```
```
WITH all_ids AS (
   SELECT requester_id AS id 
   FROM RequestAccepted
   UNION ALL
   SELECT accepter_id AS id
   FROM RequestAccepted)
SELECT id, 
   COUNT(id) AS num
FROM all_ids
GROUP BY id
ORDER BY COUNT(id) DESC
LIMIT 1
```
```
WITH all_ids AS (
   SELECT requester_id AS id 
   FROM RequestAccepted
   UNION ALL
   SELECT accepter_id AS id
   FROM RequestAccepted)
SELECT id, num
FROM 
   (
   SELECT id, 
      COUNT(id) AS num, 
      RANK () OVER(ORDER BY COUNT(id) DESC) AS rnk
   FROM all_ids
   GROUP BY id
   )t0
WHERE rnk=1
```
### 1068
```
import pandas as pd
​
def sales_analysis(sales: pd.DataFrame, product: pd.DataFrame) -> pd.DataFrame:
    sales_and_product = sales.merge(
        product,
        on=["product_id"]
        )
    df = sales_and_product[['product_name', 'year', 'price']]

    return df
```
```
SELECT 
    p.product_name, s.year, s.price
FROM 
    Sales s
JOIN 
    Product p
ON
    s.product_id = p.product_id
```
### 1070
```
import pandas as pd

def sales_analysis(sales: pd.DataFrame, product: pd.DataFrame) -> pd.DataFrame:
  df = sales.groupby('product_id', as_index=False)['year'].min()
  return sales.merge(df, on='product_id', how='inner')\
    .query('year_x == year_y')\
    .rename(columns={'year_x': 'first_year'})\
    [['product_id', 'first_year', 'quantity', 'price']]
```
```
SELECT 
  product_id, 
  year AS first_year, 
  quantity, 
  price 
FROM 
  Sales 
WHERE 
  (product_id, year) IN (
    SELECT 
      product_id, 
      MIN(year) AS year 
    FROM 
      Sales 
    GROUP BY 
      product_id
  );
```
### 1075
```
import pandas as pd

def project_employees_i(project: pd.DataFrame, employee: pd.DataFrame) -> pd.DataFrame:

    df = project.merge(employee, on='employee_id')
    
    df = df.groupby('project_id', as_index=False)['experience_years'].mean()
    
    return df.rename(columns={'experience_years': 'average_years'}).round(2)
```
```
SELECT 
    project_id,
    ROUND(AVG(experience_years), 2) AS average_years
FROM 
    Project p
JOIN 
    Employee e
ON 
    p.employee_id = e.employee_id
GROUP BY 
    project_id
```
### 1084
```
import pandas as pd

def sales_analysis(product: pd.DataFrame, sales: pd.DataFrame) -> pd.DataFrame:
    start_time = pd.to_datetime('2019-01-01')
    end_time = pd.to_datetime('2019-03-31')
    df = sales.groupby('product_id').filter(lambda x:
        min(x['sale_date']) >= start_time and max(x['sale_date']) <= end_time
    )
    df = df.drop_duplicates(subset = 'product_id')
    df = df.merge(product, left_on = 'product_id', right_on = 'product_id')
    return df[['product_id', 'product_name']]
```
```
SELECT DISTINCT p.product_id, p.product_name
FROM Sales s
LEFT JOIN Product p ON p.product_id = s.product_id
GROUP BY p.product_id
HAVING MIN(sale_date) >= '2019-01-01' AND MAX(sale_date) <= '2019-03-31';
```
### 1141
```
SELECT 
    activity_date AS day, 
    COUNT(DISTINCT user_id) AS active_users
FROM 
    Activity
WHERE 
    DATEDIFF('2019-07-27', activity_date) < 30 AND DATEDIFF('2019-07-27', activity_date)>=0
GROUP BY 1
```
### 1158
```

import pandas as pd

def market_analysis(
    users: pd.DataFrame, orders: pd.DataFrame, items: pd.DataFrame
) -> pd.DataFrame:

    # Step 1: Filter the orders dataframe to only include orders from the year 2019.
    df = orders.query("order_date.dt.year==2019").merge(
        # Step 2: Merge the filtered orders with the users dataframe on buyer_id and user_id.
        users,
        left_on="buyer_id",
        right_on="user_id",
        how="right",
    )

    # Step 3: Group the merged dataframe by user_id and join_date, then count the number of items (orders) for each user.
    result = df.groupby(["user_id", "join_date"]).item_id.count()

    # Step 4: Format the output by resetting the index and renaming the columns for clarity.
    return result.reset_index().rename(
        columns={"user_id": "buyer_id", "item_id": "orders_in_2019"}
    )
```
```
SELECT 
  u.user_id AS buyer_id, 
  join_date, 
  COUNT(o.order_id) AS orders_in_2019 
FROM 
  Users u 
  LEFT JOIN Orders o ON u.user_id = o.buyer_id 
  AND YEAR(order_date)= '2019' 
GROUP BY 
  u.user_id 
ORDER BY 
  u.user_id
```
### 1164
```
SELECT
  product_id,
  10 AS price
FROM
  Products
GROUP BY
  product_id
HAVING
  MIN(change_date) > '2019-08-16'
UNION ALL
SELECT
  product_id,
  new_price AS price
FROM
  Products
WHERE
  (product_id, change_date) IN (
    SELECT
      product_id,
      MAX(change_date)
    FROM
      Products
    WHERE
      change_date <= '2019-08-16'
    GROUP BY
      product_id
  )
```
```
SELECT
  UniqueProductId.product_id,
  IFNULL (LastChangedPrice.new_price, 10) AS price
FROM
  (
    SELECT DISTINCT
      product_id
    FROM
      Products
  ) AS UniqueProductIds
  LEFT JOIN (
    SELECT
      Products.product_id,
      new_price
    FROM
      Products
      JOIN (
        SELECT
          product_id,
          MAX(change_date) AS change_date
        FROM
          Products
        WHERE
          change_date <= "2019-08-16"
        GROUP BY
          product_id
      ) AS LastChangedDate USING (product_id, change_date)
    GROUP BY
      product_id
  ) AS LastChangedPrice USING (product_id)
```
```
SELECT
  product_id,
  IFNULL (price, 10) AS price
FROM
  (
    SELECT DISTINCT
      product_id
    FROM
      Products
  ) AS UniqueProducts
  LEFT JOIN (
    SELECT DISTINCT
      product_id,
      FIRST_VALUE (new_price) OVER (
        PARTITION BY
          product_id
        ORDER BY
          change_date DESC
      ) AS price
    FROM
      Products
    WHERE
      change_date <= '2019-08-16'
  ) AS LastChangedPrice USING (product_id);
```
### 1179
```
SELECT
  id,
  SUM(IF (month = "Jan", revenue, null)) AS Jan_Revenue,
  SUM(IF (month = "Feb", revenue, null)) AS Feb_Revenue,
  SUM(IF (month = "Mar", revenue, null)) AS Mar_Revenue,
  SUM(IF (month = "Apr", revenue, null)) AS Apr_Revenue,
  SUM(IF (month = "May", revenue, null)) AS May_Revenue,
  SUM(IF (month = "Jun", revenue, null)) AS Jun_Revenue,
  SUM(IF (month = "Jul", revenue, null)) AS Jul_Revenue,
  SUM(IF (month = "Aug", revenue, null)) AS Aug_Revenue,
  SUM(IF (month = "Sep", revenue, null)) AS Sep_Revenue,
  SUM(IF (month = "Oct", revenue, null)) AS Oct_Revenue,
  SUM(IF (month = "Nov", revenue, null)) AS Nov_Revenue,
  SUM(IF (month = "Dec", revenue, null)) AS Dec_Revenue
FROM
  Department
GROUP BY
  id;
```
```
SELECT
  Ids.id,
  January.revenue AS Jan_Revenue,
  Feburary.revenue AS Feb_Revenue,
  March.revenue AS Mar_Revenue,
  April.revenue AS Apr_Revenue,
  May.revenue AS May_Revenue,
  June.revenue AS Jun_Revenue,
  July.revenue AS Jul_Revenue,
  August.revenue AS Aug_Revenue,
  September.revenue AS Sep_Revenue,
  October.revenue AS Oct_Revenue,
  November.revenue AS Nov_Revenue,
  December.revenue AS Dec_Revenue
FROM
  (
    SELECT DISTINCT
      id
    FROM
      Department
  ) AS Ids
  LEFT JOIN Department AS January ON (
    Ids.id = January.id
    AND January.month = "Jan"
  )
  LEFT JOIN Department AS Feburary ON (
    Ids.id = Feburary.id
    AND Feburary.month = "Feb"
  )
  LEFT JOIN Department AS March ON (
    Ids.id = March.id
    AND March.month = "Mar"
  )
  LEFT JOIN Department AS April ON (
    Ids.id = April.id
    AND April.month = "Apr"
  )
  LEFT JOIN Department AS May ON (
    Ids.id = May.id
    AND May.month = "May"
  )
  LEFT JOIN Department AS June ON (
    Ids.id = June.id
    AND June.month = "Jun"
  )
  LEFT JOIN Department AS July ON (
    Ids.id = July.id
    AND July.month = "Jul"
  )
  LEFT JOIN Department AS August ON (
    Ids.id = August.id
    AND August.month = "Aug"
  )
  LEFT JOIN Department AS September ON (
    Ids.id = September.id
    AND September.month = "Sep"
  )
  LEFT JOIN Department AS October ON (
    Ids.id = October.id
    AND October.month = "Oct"
  )
  LEFT JOIN Department AS November ON (
    Ids.id = November.id
    AND November.month = "Nov"
  )
  LEFT JOIN Department AS December ON (
    Ids.id = December.id
    AND December.month = "Dec"
  );
```
### 1251
```
import pandas as pd

def average_selling_price(prices: pd.DataFrame, units_sold: pd.DataFrame) -> pd.DataFrame:
    # Join prices and units_sold tables based on product_id
    merged_data = prices.merge(units_sold, on='product_id')
    
    # Filter the merged data to only include purchase_date within the price period
    filtered_data = merged_data[
        (merged_data['purchase_date'] >= merged_data['start_date']) &
        (merged_data['purchase_date'] <= merged_data['end_date'])
    ]
    
    # Calculate revenue for each sale
    filtered_data['revenue'] = filtered_data['price'] * filtered_data['units']
    
    # Group by product_id and calculate the average selling price
    result = filtered_data.groupby('product_id')['revenue'].sum() / filtered_data.groupby('product_id')['units'].sum()
    
    # Round the result to two decimal places
    result = result.round(2).reset_index()
    result.columns = ['product_id', 'average_price']

    # Find all unique product_id
    prices = pd.DataFrame(columns=['product_id'], data=prices['product_id'].unique())

    # Left merge on prices and retain products with zero sales units.
    result = prices.merge(result[['product_id', 'average_price']], on='product_id', how='left').fillna(0)
    
    return result
```
```
SELECT
    p.product_id,
    IFNULL(ROUND(SUM(p.price * u.units) / SUM(u.units), 2), 0) AS average_price
FROM
    Prices AS p
LEFT JOIN
    UnitsSold AS u
ON
    p.product_id = u.product_id
    AND u.purchase_date BETWEEN p.start_date AND p.end_date
GROUP BY
    p.product_id;
```
### 1393
```
def solution(stocks: pd.DataFrame) -> pd.DataFrame:
    # Approach: groupby, apply
    # Helper function to update prices in stocks DataFrame
    def helper(operation, price):
        if operation == "Buy":
            return -int(price)
        elif operation == "Sell":
            return int(price)
        
    # Update 'price' column based on if 'operation' is 'Buy' or 'Sell'
    Stocks['price'] = Stocks.apply(lambda x: helper(x['operation'], x['price']), axis=1)
    
    # Groupby 'stock_name' and sum over 'price' column
    # Rename summed column to 'capital_gain_loss'
    df = Stocks.groupby(by='stock_name')['price'].sum().reset_index(name='capital_gain_loss')
    
    return df
```
```
SELECT 
    stock_name,
    SUM(
        CASE 
            WHEN operation = 'buy' THEN -price
            WHEN operation = 'sell' THEN price
        END
    ) AS capital_gain_loss
FROM Stocks
GROUP BY stock_name
```
### 1407
```
SELECT 
    u.name, 
    IFNULL(SUM(distance),0) AS travelled_distance
FROM 
    Users u
LEFT JOIN 
    Rides r
ON 
    u.id = r.user_id
GROUP BY 
    u.id
ORDER BY 2 DESC, 1 ASC
```
### 1581
```
import pandas as pd

def find_customers(visits: pd.DataFrame, transactions: pd.DataFrame) -> pd.DataFrame:

   visits_no_trans = visits[~visits.visit_id.isin(transactions.visit_id)]
   
   df = visits_no_trans.groupby('customer_id', as_index=False)['visit_id'].count()
    
   return df.rename(columns={'visit_id': 'count_no_trans'})
```
```
import pandas as pd

def find_customers(visits: pd.DataFrame, transactions: pd.DataFrame) -> pd.DataFrame:

   visits_no_trans = visits.merge(transactions, on='visit_id', how='left')

   visits_no_trans = visits_no_trans[visits_no_trans.transaction_id.isna()]

   df = visits_no_trans.groupby('customer_id', as_index=False)['visit_id'].count()

   return df.rename(columns={'visit_id': 'count_no_trans'})
```
```
SELECT 
  customer_id, 
  COUNT(visit_id) AS count_no_trans 
FROM 
  Visits 
WHERE 
  visit_id NOT IN (
    SELECT 
      visit_id 
    FROM 
      Transactions
  ) 
GROUP BY 
  customer_id
```
```
SELECT 
  customer_id, 
  COUNT(*) AS count_no_trans 
FROM 
  Visits AS v 
  LEFT JOIN Transactions AS t ON v.visit_id = t.visit_id 
WHERE 
  t.visit_id IS NULL 
GROUP BY 
  customer_id
```
### 1587
```
SELECT 
    DISTINCT a.name, b.balance
FROM 
    Users a
JOIN (
    SELECT 
        account, SUM(amount) as balance
    FROM 
        Transactions
    GROUP BY 1
    HAVING balance>10000) b
ON 
    a.account = b.account
```
```
SELECT 
    u.name, SUM(t.amount) AS balance
FROM 
    Users u
JOIN 
    Transactions t
ON 
    u.account = t.account
GROUP BY u.account
HAVING 
    balance > 10000
```
### 1633
```
import pandas as pd

def users_percentage(users: pd.DataFrame, register: pd.DataFrame) -> pd.DataFrame:
    # Calculate the total number of unique users
    total_users = users["user_id"].nunique()

    # Count the distinct user_id in each contest_id and calculate the percentage
    register_grouped = (
        register.groupby("contest_id")["user_id"]
        .nunique()
        .reset_index(name="count_unique_users")
    )

    # Calculate the percentage
    register_grouped["percentage"] = (
        register_grouped["count_unique_users"] / total_users
    ) * 100

    # Round the percentage to 2 decimal places
    register_grouped["percentage"] = register_grouped["percentage"].round(2)

    # Sort the results by percentage in descending order and then by contest_id
    register_grouped = register_grouped.sort_values(
        by=["percentage", "contest_id"], ascending=[False, True]
    )

    # Select only the contest_id and percentage columns
    final_df = register_grouped[["contest_id", "percentage"]]

    return final_df
```
```
SELECT 
  contest_id, -- The ID of the contest
  ROUND(
    COUNT(DISTINCT user_id) * 100 / ( -- Calculate the percentage of users
      SELECT 
        COUNT(user_id) -- Total number of unique users
      FROM 
        Users
    ), 
    2
  ) AS percentage -- The percentage of users registered for each contest, rounded to 2 decimal places
FROM 
  Register -- The table containing registration information
GROUP BY 
  contest_id -- Group the data by contest ID
ORDER BY 
  percentage DESC, -- Order the results by percentage in descending order
  contest_id; -- Then order by contest ID for ties
```
### 1661
```
import pandas as pd
​
def get_average_time(activity: pd.DataFrame) -> pd.DataFrame:

    activity['timestamp'] = activity.apply(lambda x: x.timestamp * -1 if x.activity_type == 'start' else x.timestamp, axis=1)

    sum_machine_process = activity.groupby(['machine_id', 'process_id'], as_index=False)['timestamp'].sum()

    mean_machine = sum_machine_process.groupby(['machine_id'], as_index=False)['timestamp'].mean().round(3).rename(columns = {'timestamp': 'processing_time'})
    
    return mean_machine
```
```
import pandas as pd

def get_average_time(activity: pd.DataFrame) -> pd.DataFrame:
    
    start_df = activity[activity['activity_type'] == 'start']
    
    end_df = activity[activity['activity_type'] == 'end']
    
    merge_df = end_df.merge(start_df, on = ['machine_id', 'process_id'])
    
    df = merge_df.assign(processing_time = merge_df['timestamp_x'] - merge_df['timestamp_y']).groupby(['machine_id'], as_index=False)['processing_time'].mean().round(3)

    return df
```
```
SELECT 
    machine_id,
    ROUND(SUM(CASE WHEN activity_type='start' THEN timestamp*-1 ELSE timestamp END)*1.0
    / (SELECT COUNT(DISTINCT process_id)),3) AS processing_time
FROM 
    Activity
GROUP BY machine_id
```
```
SELECT a.machine_id, 
       ROUND(AVG(b.timestamp - a.timestamp), 3) AS processing_time
FROM Activity a, 
     Activity b
WHERE 
    a.machine_id = b.machine_id
AND 
    a.process_id = b.process_id
AND 
    a.activity_type = 'start'
AND 
    b.activity_type = 'end'
GROUP BY machine_id
```
### 1729
```
SELECT user_id, COUNT(user_id) AS followers_count
FROM followers
GROUP BY user_id
ORDER BY user_id ASC;
```
### 1731
```
import pandas as pd

def count_employees(employees: pd.DataFrame) -> pd.DataFrame:
    # Group employees by their manager to calculate the count of reports and the average age
    by_manager = employees.groupby("reports_to", as_index=False).agg(
        reports_count=("employee_id", "size"),  # Count of reports per manager
        average_age=("age", "mean"),  # Average age of reports
    )

    # Adjust for banker's rounding by adding a very small number before rounding
    by_manager["average_age"] = (by_manager["average_age"] + 1e-12).round(0)

    # Merge the aggregated data with the original employees DataFrame to get the names of managers
    merged = by_manager.merge(
        employees[["employee_id", "name"]],
        how="left",
        left_on="reports_to",
        right_on="employee_id",
    )

    # Since the merge introduces '_x' and '_y' suffixes for overlapping column names, correct this
    # Also, directly rename columns to match expected output format without intermediate steps
    merged.rename(
        columns={
            "employee_id_y": "employee_id",  # This is the actual manager's ID
        },
        inplace=True,
    )

    # Select the columns in the order that matches the expected output
    final_output = merged[["employee_id", "name", "reports_count", "average_age"]]

    return final_output
```
```
SELECT 
  mgr.employee_id, 
  mgr.name, 
  COUNT(emp.employee_id) AS reports_count, 
  ROUND(
    AVG(emp.age)
  ) AS average_age 
FROM 
  employees emp 
  JOIN employees mgr ON emp.reports_to = mgr.employee_id 
GROUP BY 
  employee_id 
ORDER BY 
  employee_id
```
```
SELECT 
  reports_to AS employee_id, 
  (
    SELECT 
      name 
    FROM 
      employees e1 
    WHERE 
      e.reports_to = e1.employee_id 
  ) AS name, 
  COUNT(reports_to) AS reports_count, 
  ROUND(
    AVG(age)
  ) AS average_age 
FROM 
  employees e 
GROUP BY 
  reports_to 
HAVING 
  reports_count > 0 
ORDER BY 
  employee_id
```
### 1757
```
import pandas as pd

def find_products(products: pd.DataFrame) -> pd.DataFrame:
    df = products[(products['low_fats'] == 'Y') & (products['recyclable'] == 'Y')]

    df = df[['product_id']]
    
    return df
```
```
SELECT
    product_id
FROM
    Products
WHERE
    low_fats = 'Y' AND recyclable = 'Y'
```
### 1789
```
import pandas as pd

def find_primary_department(employee: pd.DataFrame) -> pd.DataFrame:
    # 1. Employees with primary_flag set to 'Y'
    filtered_by_flag = employee[employee['primary_flag'] == 'Y'][['employee_id', 'department_id']]

    # 2. Employees that appear exactly once in the Employee table
    unique_employees = employee.groupby('employee_id').filter(lambda x: len(x) == 1)[['employee_id', 'department_id']]

    # 3. Combine both DataFrames using concat and drop duplicates
    result = pd.concat([filtered_by_flag, unique_employees]).drop_duplicates().reset_index(drop=True)
    
    #4. Return result
    return result
```
```
import pandas as pd

def find_primary_department(employee: pd.DataFrame) -> pd.DataFrame:
    # 1. Calculate EmployeeCount as the number of rows for each employee_id
    employee["EmployeeCount"] = employee.groupby("employee_id")[
        "employee_id"
    ].transform("size")

    # 2. Filter based on the EmployeeCount or primary_flag
    result = employee[
        (employee["EmployeeCount"] == 1) | (employee["primary_flag"] == "Y")
    ][["employee_id", "department_id"]]

    # 3. Return result
    return result
```
```
-- Retrieving employees with primary_flag set to 'Y'
SELECT 
  employee_id, 
  department_id 
FROM 
  Employee 
WHERE 
  primary_flag = 'Y' 
UNION 
-- Retrieving employees that appear exactly once in the Employee table
SELECT 
  employee_id, 
  department_id 
FROM 
  Employee 
GROUP BY 
  employee_id 
HAVING 
  COUNT(employee_id) = 1;
```
```
SELECT 
  employee_id, 
  department_id 
FROM 
  (
    SELECT 
      *, 
      COUNT(employee_id) OVER(PARTITION BY employee_id) AS EmployeeCount
    FROM 
      Employee
  ) EmployeePartition 
WHERE 
  EmployeeCount = 1 
  OR primary_flag = 'Y';
```
### 1890
```
SELECT 
    user_id, 
    MAX(time_stamp) AS last_stamp
FROM 
    Logins
WHERE 
    YEAR(time_stamp) = 2020
GROUP BY 1;
```
```
SELECT
    DISTINCT user_id,
    FIRST_VALUE(time_stamp)OVER(PARTITION BY user_id ORDER BY time_stamp DESC) AS last_stamp
FROM
    Logins
WHERE EXTRACT(Year FROM time_stamp) = 2020;
```
### 1965
```
import pandas as pd

def find_employees(employees: pd.DataFrame, salaries: pd.DataFrame) -> pd.DataFrame:

    return pd.DataFrame(
        {"employee_id": sorted(set(employees.employee_id) ^ set(salaries.employee_id))}
    )
```
```
import pandas as pd

def find_employees(employees: pd.DataFrame, salaries: pd.DataFrame) -> pd.DataFrame:
    # Merge the employees and salaries DataFrames on 'employee_id', including all records from both.
    merged_df = pd.merge(employees, salaries, on="employee_id", how="outer")

    # Identify rows with missing values in any column.
    missing_data_df = merged_df[merged_df.isna().any(axis=1)]

    # Select only the 'employee_id' column and sort the IDs.
    result_df = missing_data_df[["employee_id"]].sort_values(by="employee_id")

    return result_df
```
```
SELECT 
  T.employee_id 
FROM 
  (
    SELECT 
      * 
    FROM 
      Employees 
      LEFT JOIN Salaries USING(employee_id) 
    UNION 
    SELECT 
      * 
    FROM 
      Employees 
      RIGHT JOIN Salaries USING(employee_id)
  ) AS T 
WHERE 
  T.salary IS NULL 
  OR T.name IS NULL 
ORDER BY 
  employee_id;
```
```
SELECT 
  employee_id 
FROM 
  Employees 
WHERE 
  employee_id NOT IN (
    SELECT 
      employee_id 
    FROM 
      Salaries
  ) 
UNION 
SELECT 
  employee_id 
FROM 
  Salaries 
WHERE 
  employee_id NOT IN (
    SELECT 
      employee_id 
    FROM 
      Employees
  ) 
ORDER BY 
  employee_id ASC
```
### 2877
```
import pandas as pd

def createDataframe(student_data: List[List[int]]) -> pd.DataFrame:
    column_names = ["student_id", "age"]
    result_dataframe = pd.DataFrame(student_data, columns=column_names)
    return result_dataframe
```
### 2878
```
import pandas as pd

def getDataframeSize(players: pd.DataFrame) -> List:
    return [players.shape[0], players.shape[1]]
```
### 2879
```
import pandas as pd

def selectFirstRows(employees: pd.DataFrame) -> pd.DataFrame:
    return employees.head(3)
```
### 2880
```
import pandas as pd

def selectData(students: pd.DataFrame) -> pd.DataFrame:
    return students.loc[students["student_id"] == 101, ["name", "age"]]
```
### 2881
```
import pandas as pd

def createBonusColumn(employees: pd.DataFrame) -> pd.DataFrame:
    employees['bonus'] = employees['salary'] * 2
    return employees
```
### 2882
```
import pandas as pd

def dropDuplicateEmails(customers: pd.DataFrame) -> pd.DataFrame:
    customers.drop_duplicates(subset='email', keep='first', inplace=True)
    return customers
```
### 2883
```
import pandas as pd

def dropMissingData(students: pd.DataFrame) -> pd.DataFrame:
    students.dropna(subset=['name'], inplace=True)
    return students
```
### 2884
```
import pandas as pd

def modifySalaryColumn(employees: pd.DataFrame) -> pd.DataFrame:
    employees['salary'] = employees['salary'] * 2
    return employees
```
### 2885
```
import pandas as pd

def renameColumns(students: pd.DataFrame) -> pd.DataFrame:
    students = students.rename(
        columns={
            "id": "student_id",
            "first": "first_name",
            "last": "last_name",
            "age": "age_in_years",
        }
    )
    return students
```
### 2886
```
import pandas as pd

def changeDatatype(students: pd.DataFrame) -> pd.DataFrame:
    students = students.astype({'grade': int})
    return students
```
### 2887
```
import pandas as pd

def fillMissingValues(products: pd.DataFrame) -> pd.DataFrame:
    products['quantity'].fillna(0, inplace=True)
    return products
```
### 2888
```
import pandas as pd

def concatenateTables(df1: pd.DataFrame, df2: pd.DataFrame) -> pd.DataFrame:
    return pd.concat([df1, df2], axis=0)
```
### 2889
```
import pandas as pd

def pivotTable(weather: pd.DataFrame) -> pd.DataFrame:
    ans = weather.pivot(index='month', columns='city', values='temperature')
    return ans
```
### 2890
```
import pandas as pd

def meltTable(report: pd.DataFrame) -> pd.DataFrame:
    report = report.melt(
        id_vars=["product"],
        value_vars=["quarter_1", "quarter_2", "quarter_3", "quarter_4"],
        var_name="quarter",
        value_name="sales",
    )
    return report
```
### 2891
```
def findHeavyAnimals(animals: pd.DataFrame) -> pd.DataFrame:
    filtered_animals = animals[animals['weight'] > 100]
    sorted_animals = filtered_animals.sort_values(by='weight', ascending=False)
    names = sorted_animals[['name']]
    return names
```
```
import pandas as pd

def findHeavyAnimals(animals: pd.DataFrame) -> pd.DataFrame:
    return animals[animals['weight'] > 100].sort_values(by='weight', ascending=False)[['name']]
```
