# Теоретическая часть
## 1. Что такое LTV? Если у нас есть данные за первый месяц жизни приложения, как бы вы рассчитали LTV?

LTV(Life Time Value) доход, который приносит пользователь за свое время жизни в приложении, то есть до тех пор, пока он не перестал им пользоваться.
Эта метрика является одной из важных метрик в игровой аналитике, так как она помогает понять, насколько ценно привлечение новых пользователей и сколько можно инвестировать в маркетинг для их привлечения.
Имея данные за первый месяц жизни приложения, мы не обладаем информацией о полном жизненном цикле приложения, но можем посчитать LTV несколькими способами. Приведу два из них.

1.	Стандартный способ, самый простой и грубый, так как он просто отражает ARPU за месяц и не учитывает реальный жизненный цикл пользователей:


    LTV = ARPU * t.

  

         	  	  	
    LTV = ARPU * t. 

ARPU (average revenue per user) средний доход с пользователя рассчитать в свою очередь в виде:  

    ARPU = total revenue (за месяц) / total users (за месяц)(MAU).

Фактически получим LTV за первый месяц вида:

    LTV = total revenue (за месяц) / total users (за месяц).


2.	Более точный способ расчета LTV на основе исторических данных – это когортный анализ
Действуем по алгоритму:
* Выделяем когорты пользователей, пришедших в определённый день.
* Считаем ARPU по каждой когорте пользователей за 1-й, 2-й, 3-й день и т. д. по формуле:
  ARPUd = Revenued / Cohort_size,
  где Revenue_d — доход всей когорты в день d, Cohort_size — число пользователей, установивших приложение в день формирования когорты.
  
* Считаем LTV на каждый день для каждой когорты, как накопительный ARPU
  LTV_0 = ARPU_0
  LTV_1 = ARPU_0 + ARPU_1
  LTV_2 = ARPU_0 + ARPU_1 + ARPU_2
  ...
  
* Получаем таблицу, где строки – это дни установки, а столбцы доход на день жизни пользователя.
  На нужный день можно посчитать LTV, как среднее по когортам.
   
Если у нас есть данные за 30 дней, и мы хотим посчитать LTV на 30-ый день, то при таком алгоритме мы сможем использовать только первую когорту.
Но способ оценки LTV через кумулятивный ARPU хорошо тем, что можно экстраполировать тренд и построить прогноз LTV.
Кривая накопительного ARPU представляет собой логарифмическую функцию, асимптотой которой будет LTV.


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

## Поиск проблемных квестов 
Воронка прохождения квестов. Кол-во юзеров, завершивших каждый квест.  
   Результат: quest - номер квеста - кол-во игроков.
```sql
WITH quests AS
(
SELECT 
    qs.user_id,
    qs.time AS time_start,
    COALESCE (qc.time, DATETIME('3000-01-01 00:00:00')) AS time_end,
    qs.quest,
    CAST(SUBSTR(qs.quest, 7, 8) AS INTEGER) AS quest_number
FROM quest_start qs
LEFT JOIN quest_complete qc ON (qs.user_id = qc.user_id)
                           AND (qs.quest = qc.quest)
)
SELECT 
    quest,
    quest_number,
    COUNT(DISTINCT user_id) AS completed_quest_players
FROM quests
WHERE time_end < DATETIME('3000-01-01 00:00:00')
GROUP BY quest
ORDER BY quest_number
```

# Визуализация и выводы
Визуализацию данных я осуществляла в Preset (open-source Apache Superset).

## Удержание и DAU

Для когорт пользователей, установивших приложение с 2020-10-05 по 2020-10-14 был рассчитан Retention по первым 7 дням.  
Можно отметить следующие наблюдения:    
1. R1 колеблется в промежутке от 41% до 62. В среднем показатель хороший. Для всех дней установки, кроме 2020-10-08 и 2020-10-10, R1 близок к 60%.
2. Для инсталлов 8 и 10 октября R1 достаточно низкий (41% и 46%), в следующие дни retention стремительно падает.
3. Самый низкий показатель R7 составляет 15.65% (для дня установки 2020-10-10). Стоит проверить работу маркетинга на 8 и 10 октября, возможно, была неудачная кампания и закупали нецелевую аудиторию.
4. Для дня установки 2020-10-09 R1 хороший, но затем резко снижается. Стоит проверить, что происходило в игре для этих пользователей на второй день. Возможно, баг или изменения, которые негативно повлияли на удержание.
5. За исключением дней закупки юзеров 8, 9 и 10 октября, R7 колеблется около 20%, что вполне неплохо для F2P игры.

![](/Figures/Retention.png)

Динамика DAU демонстрирует cильный рост до 12500 (2020-10-12), затем резкий спад (2020-10-24, DAU = 190).
В дни максимальной активности было закуплено рекордное количество пользователей — вероятно, это эффект рекламного буста. Однако падение DAU показывает, что не сработало удержание.

![](/Figures/DAU.png)

## Воволечение

Среднее количество сессий на активного игрока постепенно растет в начале периода и затем стабильно удерживается около 4 сессий за день.
К концу интервала показатель начинает уменьшаться, что может свидетельствовать о плохом удержании и потери интереса.

![](/Figures/AVG_count_session.png)

Время, проведенное в игре, коррелирует с этими выводами. В зависимости от порядкового номера сессии игроки проводят, в среднем около 600-700 сек., но после 140 сессии время падает и появляются значительные флуктуации.
Есть аномальные выбросы в нескольких сессиях, которые соответствуют единичным игрокам, находящимся в игре около 1.5 часов.
Аномально большое время для сессии 199, связанное с единичным юзером, было исключено из графика. Юзер 117411339 играл 32215 сек, то есть полный рабочий день.
Стоит проверить, возможно, это был ваш QA специалист!:)  

А если серьезно, то для получения чистых данных, необходимо проверить, прерываются ли сессии игроков при отсутствии активности определенное время.
Возможно, стоит ввести такое ограничение и принудительно завершать сессию при отсутствии активности, чтобы избегать таких ложных аномалий.

![](/Figures/AVG_duration_session.png)

## Монетизация

Динамика Revenue по дням показывает рост до 2020-10-14, а затем падение. Аномальный всплеск 9 октября соответствует большой покупке уникального юзера 47862140.
Он совершил в один день 6 платежей в общей сумме на 39 ед. Динамика Revenue хорошо коррелирует с DAU. В последний день выручка минимальна,
как и число активных пользователей. Нужно поработать над привлечением и удержанием пользователей, так как в данном случае, платить уже некому.

ARPU, ARPPU изменяются равномерно с некоторыми точечными всплесками и падениями. Пик 9 октября соответствует дорогостоящей покупки юзера  47862140.
23 октября основной вклад в доход внесло несколько юзеров. Так как DAU в этот день самый низкий, ARPPU и особенно ARPU реагируют чутко.

ARPU минимален 2020-10-10, ARPPU также близок к минимуму. В сочетании с Retention напрашивается вывод, что в этот день пришел неподходящий трафик.

![](/Figures/ARPU_ARPPU.png)

Среднее число сессий перед первым платежом (7.93) указывает на то, что монетизация наступает не сразу.
Этот вывод подтверждается графиками конверсии игроков в платящих по уровням и квестам. Основной пик приходится на 5-8 уровни.
С 10 уровня конверсия резко падает. Вероятно, в этот момент падает вовлечение, игроки теряют интерес или не видят ценности внутриигровых покупок. 

![](/Figures/Conversion_to_paid_users.png)

Наибольший вклад в revenue приносят квесты quest_26, quest_0, quest_20, quest_30, quest_36 и quest_23.
Есть квесты с низким revenue, но высокой конверсией в платежи (quest_11, quest_12, quest_13, quest_17).
Quest_23 имеет максимальную конверсию в платящих, а вклад ревенью не такой внушающий. Нужно проверить, как их можно монетизировать эффективнее.
Аномально высокая выручка на quest_0 (17.55). Обычно на этом уровне монетизация минимальна, основная цель – вовлечение пользователя и закрепление интереса.
Большую часть revenue здесь сгенерировал юзер 47862140 с двумя крупными платежами на общую сумму 15.2 с разницей в 20 сек.

![](/Figures/Revenue_by_quest.png)

## Воронка прохождения квестов. Поиск проблемных мест

Воронка прохождения квестов показывает резкие отвалы на квестах 3, 6, 13, 21, 26, 30. Особенно заметно значительное число отвалов на 3 квесте, что сильно влияет на удержание пользователей и будущий доход.
Такое поведение может свидетельствовать о его чрезмерной сложности, недостатке мотивации или баге.
По таблице квестов можно оценить промежуток времени от начала квеста до его завершения, но возможно, этот интервал будет включать и паузы в прохождении квестов.
Но даже учитывая этот факт, время от начала квеста до завершения составляет 24 минуты для 0 квеста, 35 – для первого, 5 минут для 2 квеста и 121 для третьего!
Этот факт свидетельствует о сложностях в прохождении этого квеста. С такими данными хорошо бы оценить баланс расходников игры (энергии, монет, рубинов и т.д.).
Если энергия в дефиците, например, это может привести к торможению в расчистке и продвижению по квесту.  Если с балансом все в порядке могут быть баги или плохие туториалы.

![](/Figures/Funnel_churn.png)

Для проведения A/B теста лучше начать с изменений на третьем квесте. Сформируем гипотезу по изменению сложности квестов:
Снижение сложности квестов повысит их завершение и общую вовлеченность.
Необходимо отслеживать монетизацию, например, ARPU, он не должен снизиться по тестовой группе; должны уменьшиться отвалы (воронка прохождения квестов), среднее время прохождения квестов также должно уменьшиться.

Для более качественного анализа не хватает системы расходников, действий в банке, прогресса по локациям. В таблицу payment я бы добавила источник оплаты. Важно также отслеживать Daily bonus.
Дополню таблицу ивентами:

![](/Figures/Events_table.png)

Дополненные события позволили бы расширить и уточнить анализ в следующих направлениях:  

1. Система расходников поможет отслеживать их баланс, выявлять дефициты и накопления, чтобы корректировать плавность прохождения квестов.
   Если энергия быстро уходит, а пополнение медленное, будут отвалы.
   Если основной расход идёт на конкретный ресурс, его балансировка может быть ключевой.  
2. Отслеживание открытия банка даст нам знание и уверенность, что у игрока в очередном релизе открывается основной источник нашего дохода. Кроме того, если многие игроки заходят в банк, но не совершают платеж, нужно поработать над офферами.
3. Отслеживание прохождения локаций даст оценку скорости поглощения юзером контента. Таким способом можно контролировать скорость разработки новых локаций.
4. Отслеживание получения юзером ежедневной награды – важная метрика, влияющая на возврат игроков. Если бонус не получают, закрывают окно, надо переделать.







