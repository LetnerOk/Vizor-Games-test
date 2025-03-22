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

