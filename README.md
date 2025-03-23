# Теоретическая часть
## 1. Что такое LTV? Если у нас есть данные за первый месяц жизни приложения, как бы вы рассчитали LTV?

LTV(Life Time Value) доход, который приносит пользователь за свое время жизни в приложении, то есть до тех пор, пока он перестал им пользоваться.
Имея данные за первый месяц жизни приложения, мы не обладаем информацией о полном жизненном цикле приложения, поэтому можно посчитать LTV стандартным способом. Например:  

LTV = ARPU*t

ARPU (average revenue per user) средний доход с пользователя расчитать в свою очередь в виде:  

ARPU = total revenue (за месяц) / total users (за месяц)(MAU). Фактически получим LTV за первый месяц вида:  

LTV = total revenue (за месяц) / total users (за месяц)

## 2. Что такое ARPPU? В результате изменений в продукте ARPPU снизился. Это хорошо или плохо? Поясните ответ.

ARPPU (Average Revenue Per Paid User) средний доход с платящего пользователя. Помним, что главная цель продукта – это доход с него.
Если снижается ARPPU это плохо, но важно понять причину изменений. Если для новой фичи стоит цель повысить приток новых игроков и повысить Retention,
и эта гипотеза подтвердилась, то, ARPPU может снизиться из-за новичков, так как обычно платить начинают позже. Также ARPPU может снижаться при проведении акций.
В краткосрочной перпективе допустимо жертвовать ARPPU ради долгосрочного роста LTV. После проведения акции может возникнуть эффект каннибализации, который приведет к снижению ARPPU.
Чтобы этого избежать, нужно следить за накоплением игроками внутреигровой валюты.

## 3. При тесте новой маркетинговой кампании значительно снизилась стоимость установки, но упал и ретеншн 1 дня. Какие метрики стоит смотреть и какую картину по этим метрикам вы ожидаете увидеть, чтобы понять, это хорошо или плохо?


# Практическая часть

Приведу SQL запросы, необходимые для анализа вовлечения и монетизации когорты пользователей, установивших приложение с 2020-10-05 по 2020-10-14.

## Метрики вовлеченности
1. Retention 1-7 дней по дням установки приложения. За активность игрока в определенный день считать старт сессии.  
Результат: день установки-кол-во юзеров - процент активных игроков(R0-R7). 
```sql
WITH users AS
(
SELECT
    user_id,
    DATE(reg_time) AS install_date,
    COUNT(user_id) OVER(Partition BY DATE(reg_time)) AS users_install
FROM install
), returned_d AS 
(
SELECT
    count(DISTINCT sc.user_id) AS returned_users,
    u.users_install AS users,
    u.install_date,
    DATE(open_time) AS returned_date,
    CAST((JULIANDAY(sc.open_time) - JULIANDAY(u.install_date)) AS INTEGER) AS day_num
FROM users u
JOIN session_close sc ON u.user_id = sc.user_id
WHERE (DATE(sc.open_time) >= DATE(install_date)) AND ( DATE(sc.open_time) <= DATE(install_date, '+7 days'))
GROUP BY u.install_date, returned_date   
)
SELECT 
    rd.install_date,
    users,
    100.0 AS R0,
    ROUND(sum(CASE WHEN day_num = 1 THEN returned_users ELSE 0 END) * 100.0 / users, 2) AS R1,
    ROUND(sum(CASE WHEN day_num = 2 THEN returned_users ELSE 0 END) * 100.0 / users, 2) AS R2,
    ROUND(sum(CASE WHEN day_num = 3 THEN returned_users ELSE 0 END) * 100.0 / users, 2) AS R3,
    ROUND(sum(CASE WHEN day_num = 4 THEN returned_users ELSE 0 END) * 100.0 / users, 2) AS R4,
    ROUND(sum(CASE WHEN day_num = 5 THEN returned_users ELSE 0 END) * 100.0 / users, 2) AS R5,
    ROUND(sum(CASE WHEN day_num = 6 THEN returned_users ELSE 0 END) * 100.0 / users, 2) AS R6,
    ROUND(sum(CASE WHEN day_num = 7 THEN returned_users ELSE 0 END) * 100.0 / users, 2) AS R7
FROM returned_d rd
GROUP BY rd.install_date
```

2. Среднее кол-во сессий на активного игрока по дням с момента регистрации.  
   Результат: день - среднее кол-во сессий.
```sql
WITH active_sessions AS
(
SELECT
    sc.user_id,
    DATE(open_time) as active_date,
    COUNT(*) AS sessions_per_day
FROM session_close sc
JOIN install ON sc.user_id=install.user_id AND (DATE(open_time) >= DATE(reg_time))
WHERE duration != 0
GROUP BY 1, 2
ORDER BY 2
)
SELECT 
    active_date,
--    ROUND(SUM(sessions_per_day)*1.0/COUNT(distinct user_id), 2) AS avg_sessions_per_user,
    ROUND(AVG(sessions_per_day), 2) AS avg_sessions_per_user
FROM active_sessions
GROUP BY active_date
```

3. Cредняя длительность сессии в зависимости от порядкового номера сессии с исключением юзера 117411339 из сессии 199 с аномальной продолжительностью.  
   Результат: номер сессии - средняя продолжительность.
```sql
WITH num_sesssions AS
(
SELECT 
    *,
    ROW_NUMBER() OVER(Partition BY user_id ORDER BY open_time) AS number_session
FROM session_close
WHERE user_id != 117411339
)
SELECT
    number_session,
    ROUND(avg(duration), 2) AS average_duration
FROM num_sesssions
GROUP BY number_session
```

4. Проверка аномалий в длительности сессий  
```sql
WITH num_sesssions AS
(
SELECT 
    *,
    ROW_NUMBER() OVER(Partition BY user_id ORDER BY open_time) AS number_session
FROM session_close
)
SELECT *
FROM num_sesssions
WHERE number_session in (141, 154, 125, 198, 199)
ORDER BY number_session
```

## Метрики монетизации
1. Revenue по дням
```sql
SELECT
   DATE(time) AS day,
   SUM(amount) AS revenue
FROM payment
GROUP BY  DATE(time)
```

2. ARPU, DAU по дням  
   Результат: день - DAU - ARPU.
```sql
SELECT 
    DATE(open_time) AS day,
    COUNT(DISTINCT sc.user_id) AS DAU,
    COALESCE(SUM(amount)/COUNT(DISTINCT sc.user_id), 0) AS ARPU
FROM session_close sc
LEFT JOIN payment p ON sc.user_id = p.user_id AND DATE(sc.open_time) =  DATE(p.time)
GROUP BY DATE(open_time)
```

3. ARPPU по дням  
   Результат: день - ARPPU.
```sql
SELECT 
    DATE(time) AS day,
    SUM(amount)/COUNT(DISTINCT user_id) AS ARPPU
FROM payment
GROUP BY DATE(time)
ORDER BY DATE(time)
```

4. Кол-во сконвертированных игроков в плательщики по уровням.  
   Результат: level - кол-во игроков, кто совершил первый платеж на данном уровне.
```sql
WITH first_payments AS
(
SELECT
    user_id,
    DATE(MIN(time)) AS first_payment
FROM payment
GROUP BY user_id
)
SELECT 
    level,
    COUNT(DISTINCT l.user_id) AS users_with_first_pay
 FROM level_up l
 JOIN first_payments fp ON l.user_id = fp.user_id
                       AND DATE(l.time) = fp.first_payment
GROUP BY level
ORDER BY level
```

5. Кол-во сконвертированных игроков в плательщики по квестам.

```sql
WITH first_payments AS (
SELECT
    user_id,
    MIN(time) AS first_payment
FROM payment
GROUP BY user_id
), quests AS (
SELECT 
    qs.user_id,
    qs.time AS time_start,
    COALESCE (qc.time, DATETIME('3000-01-01 00:00:00')) AS time_end,
    qs.quest
FROM quest_start qs
LEFT JOIN quest_complete qc ON (qs.user_id = qc.user_id)
                           AND (qs.quest = qc.quest)
)
SELECT 
    q.quest,
 --   CAST(SUBSTR(quest, 7, 8) AS INTEGER) AS quest_number,  --Для корректной сортировки по квестам можно извлечь его номер и записать в отдельное поле
    COUNT(DISTINCT q.user_id) AS first_payed_users
FROM  quests q
JOIN first_payments fp ON (q.user_id) = fp.user_id 
                      AND fp.first_payment BETWEEN time_start AND time_end
GROUP BY q.quest
```

6. Суммарное ревенью игроков в зависимости от квестов, которые были активны в момент совершения покупки.  
   При наличии нескольких активных квестов разделить ревенью в равной степени на каждый квест.  
   Результат: quest - номер квеста - суммарное кол-во ревенью.  
   Таблицу с квестами соединяется с таблицей оплат с помощью Left join. Один платеж соответствует одному юзеру, поэтому если у юзера есть одновременно активные два квета,
   соединение даст две строки с разными квестами, но одним времене  оплаты. В таком случае оконная функция будет считать одновременно активные квесты в момент этой оплаты.
```sql
WITH quests AS
(
SELECT 
    qs.user_id,
    qs.time AS time_start,
    COALESCE (qc.time, DATETIME('3000-01-01 00:00:00')) AS time_end,
    qs.quest
FROM quest_start qs
LEFT JOIN quest_complete qc ON (qs.user_id = qc.user_id)
                           AND (qs.quest = qc.quest)
)
, pyaments_in_quest AS (
SELECT 
    qs.user_id,
    qs.quest,
    amount,
    p.time,
    amount/(COUNT(qs.quest) OVER (PARTITION BY p.time)) AS amount_per_active_quests
FROM quests qs
INNER JOIN payment p ON (qs.user_id = p.user_id)
           AND (p.time BETWEEN time_start AND time_end)
ORDER BY qs.quest

)
SELECT 
    quest,
   CAST(SUBSTR(quest, 7, 8) AS INTEGER) AS quest_number,
    SUM(amount_per_active_quests)
FROM pyaments_in_quest
GROUP BY quest
ORDER BY quest_number
```
