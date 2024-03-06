# Лабораторная работа 1
### Выполнил студент группы 6133-010402D Читоркин Егор

Для начала опишу выбранную предметную область - магазин приложений.

Магазином пользуются люди (user), у каждого пользователя есть свой профиль. Пользователи могут устанвливать себе приложения, оставлять комментарии своим приложениям, совершать покупки в приложениях.

Ниже представлена ER-диаграмма базы данных, созданная с помощью ERD-Tool в pgadmin.

![ER-диаграмма](/ЛР1/img/scheme.png)

Поясню происходящее. Есть таблцица пользователей, описываемая данными для авторизации. Для у каждого пользователя есть 1 профиль в магазине приложений, у каждого профиля есть только один пользователь - связь 1:1. Профиль содержит в себе электронную почту и телефон пользователя, а также информацию о пользователе в свободной форме. Каждый пользователь может устанавливать себе приложения - связь N:M (пользователь может установить много приложений, приложение может быть установлено многими пользователями) реализована через вспомогательную таблицу app_user, содержащую в себе пары идентификаторов приложений и пользователей. Таблица приложений содержит название приложения, количество скачиваний, жанр, цену, заработок приложения и флаг рекомендации. Пользователь может оставить свои комментарии на приложения (ставить оценку по пятибалльной шкаоле и оставлять текстовый комментарий) - связи 1:М между таблицами user-comment и app-comment. Также пользователь может совершать в приложениях покупки (стоимость покупки и timestamp покупки) - связи 1:М между таблицами user-comment и app-comment. Наконец, каждая сессия пользователя в приложениях фиксируется путем сохранения времени входа и времени выхода из приложения - связи 1:М между таблицами user-comment и app-comment.

Далее данные таблицы были создани с помощью  SQL-скриптов. Я один пример скрипта приведу, ибо они все однотипные.

```sql
CREATE TABLE IF NOT EXISTS public.profile
(
    id uuid NOT NULL,
    email character varying(64) NOT NULL,
    phone character varying(32) NOT NULL,
    about_me text,
    user_id uuid NOT NULL,
    CONSTRAINT profile_pkey PRIMARY KEY (id),
    CONSTRAINT profile_user_id_key UNIQUE (user_id),
    CONSTRAINT user_uuid FOREIGN KEY (user_id)
        REFERENCES public."user" (id)
        NOT VALID
)
```

Далее к столбцам, которые в теории могут использоваться для поиска, были созданы индексы. Приведу несколько примеров создания индекса, т.к. создавал обычние и уникальные + btree и hash.

Скрипт создания индекса по столбцу app_id таблицы session
```sql
CREATE INDEX IF NOT EXISTS idx_app_sessions
    ON public.session USING hash
    (app_id)
    TABLESPACE pg_default;
```

Скрипт создания индекса по столбцу end_date таблицы session
```sql
CREATE INDEX IF NOT EXISTS idx_end_session
    ON public.session USING btree
    (end_date ASC NULLS LAST)
    TABLESPACE pg_default;
```

Скрипт создания индекса по столбцу user_id таблицы profile
```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_user_profile
    ON public.profile USING btree
    (user_id ASC NULLS LAST)
    TABLESPACE pg_default;
```

Скрипт создания индекса по столбцам username и pwd таблицы user
```sql
CREATE UNIQUE INDEX IF NOT EXISTS idx_credentials
    ON public."user" USING btree
    (username COLLATE pg_catalog."default" ASC NULLS LAST, pwd COLLATE pg_catalog."default" ASC NULLS LAST)
    TABLESPACE pg_default;
```

Далее были разработаны типовые запросы в базу в количестве пяти штук. Далее буду приводить описание запроса и соответствующий SQL-скрипт.

1. Получение данных для авторизации пользователей, у которых количество установленных  приложений больше среднего
```sql
SELECT username, pwd
FROM "user"
WHERE "id" IN (
    SELECT user_id
    FROM app_user
    GROUP BY user_id
    HAVING COUNT(*) > (
        SELECT AVG(Counted)
        FROM (
            SELECT user_id, COUNT(*) AS Counted
            FROM app_user
            GROUP BY user_id
        ) AS SubQuery
    )
);
```
2. Получить жанр приложений, в которые было выполнено больше всего входов за все время, и количество этих приложений
```sql
SELECT genre, COUNT(DISTINCT app.id)
FROM app
JOIN "session" ON app.id = "session".app_id
GROUP BY app.genre
HAVING COUNT(*) = (
    SELECT MAX(Counted)
    FROM (
        SELECT COUNT(*) AS Counted
        FROM app
        JOIN "session" ON app.id = "session".app_id
        GROUP BY app.genre
    ) AS SubQuery
);
```
3. Получить электронную почту пользователя(-ей), который оставил наибольшее количество негативных отзывов (stars = 1) приложениям жанра с наилучшим средним рейтингом
```sql
SELECT email FROM profile WHERE user_id in (SELECT user_id
FROM (
    SELECT user_id, COUNT(stars) AS Counted_Stars
    FROM "comment" 
    JOIN app ON "comment".app_id = app.id 
    WHERE stars = 1 AND genre = (
        SELECT genre FROM "comment" 
        JOIN app ON "comment".app_id = app.id 
        GROUP BY genre 
        HAVING AVG(stars) = (
            SELECT MAX(mean) FROM (
                SELECT genre, AVG(stars) AS mean 
                FROM "comment" 
                JOIN app ON "comment".app_id = app.id 
                GROUP BY genre
            ) AS SubQuery
        )
    ) 
    GROUP BY "comment".user_id
) AS SubQuery2
WHERE Counted_Stars = (
    SELECT MAX(Counted_Stars) FROM (
        SELECT user_id, COUNT(stars) AS Counted_Stars
        FROM "comment" 
        JOIN app ON "comment".app_id = app.id 
        WHERE stars = 1 AND genre = (
            SELECT genre FROM "comment" 
            JOIN app ON "comment".app_id = app.id 
            GROUP BY genre 
            HAVING AVG(stars) = (
                SELECT MAX(mean) FROM (
                    SELECT genre, AVG(stars) AS mean 
                    FROM "comment" 
                    JOIN app ON "comment".app_id = app.id 
                    GROUP BY genre
                ) AS SubQuery
            )
        ) 
        GROUP BY "comment".user_id
    ) AS SubQuery3
)
);
```
4. Получить средний рейтинг каждой группы приложений по жанру, учитывая только те приложения, у которых поличество пользователей ниже среднего и в который был выполнен вход в течение последней недели
```sql
SELECT genre, AVG(stars)
FROM app
JOIN "comment" ON app.id = "comment".app_id
WHERE app.id IN (
	SELECT app_id
	FROM app_user
	WHERE app_id IN (
		SELECT app_id
		FROM "session"
		WHERE start_date BETWEEN CURRENT_DATE - INTERVAL '1 week' AND CURRENT_DATE
	)
	GROUP BY app_id
	HAVING COUNT(user_id) < (
		SELECT AVG(Counted)
		FROM (
			SELECT app_id, COUNT(user_id) AS Counted
			FROM app_user
			GROUP BY app_id
		)
		AS SubQuery
	)
)
GROUP BY genre;
```
5. Получить название приложения(-ий), у которого(-ых) суммарное время всех сессий максимально
```sql
SELECT title
FROM app
WHERE "id" IN (
	SELECT app_id
	FROM (
		SELECT app_id, SUM(EXTRACT(EPOCH FROM (end_date - start_date)) / 3600) AS time_in_app
		FROM "session"
		GROUP BY app_id
	) AS SubQuery
	WHERE time_in_app = (
		SELECT MAX(time_in_app) FROM (
			SELECT app_id, SUM(EXTRACT(EPOCH FROM (end_date - start_date)) / 3600) AS time_in_app
			FROM "session"
			GROUP BY app_id
		) AS MaxSubQuery
	)
);
```

Далее я занялся наполнением этой прекрасной базы данных. Я не стал делать хранимые процедуры, а обратился к языку Python. Числовые данные генерировались с помощью функций библиотеки random, тогда как текстовые данные - посредством библиотеки faker. Далее я подробнее опишу создание случайных данных для таблицы profile, так как остальные таблицы генерировались по аналогии. С самими скриптами можно ознакомиться, если открыть файлик [generate_data.ipynb](https://github.com/Mr-Egorchik/DB-in-Enterprise/blob/3a1c1bf76bb0e78398d9dc9cb940874a84bb0870/%D0%9B%D0%A01/generate_data.ipynb)

```py
profiles = []

for i in range(len(users)):
    user_id = users[i]['id']
    id = uuid.uuid4()
    users[i]['profile_id'] = id
    email = fake.email()
    phone = fake.phone_number()
    about_me = fake.text(128)
    profiles.append(
        {
            'id': id,
            'email': email,
            'phone': phone,
            'about_me': about_me,
            'user_id': user_id,
        }
    )
```

Хранил я все в виде массивов, содержащих однотипные словари. Идентификатор просто генерировался через функцию uuid4() библиотеки uuid. Остальные данные являются текстовыми, поэтому, чтобы в них был хоть какой-то смысл, я их генерировал с помощью библиотеки faker. Так, функция email() генерирует электронный адрес, phone_number() - номер телефона, text(int: n) - текст указанной длины.

Далее эти данные были сохранены в базу данных посредством библиотеки psycopg2. Далее тоже приведу пример вставки данных в таблицу profile, с осталными все аналогично с  точностью до названий столбцов.

```py

cur = conn.cursor()

for profile in profiles:
    cur.execute('INSERT INTO profile (id, email, phone, about_me, user_id) VALUES (%s, %s, %s, %s, %s)',
                (profile['id'], profile['email'], profile['phone'], profile['about_me'], profile['user_id']))
```

Что здесь происходит? Перед этим мы подключились к базе данных, тем самым создали объект connection. Далее мы создаем курсор с помощью метода cursor() и сохраняем данные путем вызова метода execute() у курсора, в который передаем запрос на вставку данных и параметры этого запроса.

В таблицы суммарно было вставлено порядка 5 миллионов строк, большинство которых пришлось на таблицу sessions. Генерация данных заняла где-то 1.5 минуты, все инсерты - тоже 1.5, но уже часа...

Далее была протсетирована работа всех запросов при стандартной конфигурации сервера. Были получены следующие показатели времени выполнения запросов (усредненные):

|Номер запроса|1|2|3|4|5|
|-------------|-|-|-|-|-|
|Время выполнения, мс|477|10677|1420|335|1685|

Запрос 2 оказался самым прожорливым, так что надо как-то его оптимизировать.

![Оптимизируем ](/ЛР1/img/dr-ratio.avif 'Да начнется оптимизация всего, что движется!')
*Да начнется оптимизация всего, что движется!*

Далее я приведу таблицу изменения параметров сервера и изменения времени выполнения запроса, после чего приведу определнны пояснения и комментарии. Будем ориентироваться на второй запрос, но заодно и проследим за поведением остальных запросов.

|Эксперимент|Запрос 1 (мс)|Запрос 2 (мс)|Запрос 3 (мс)|Запрос 4 (мс)|Запрос 5 (мс)|random_page_cost|shared_buffer (Мб)|work_mem (Мб)|
|-----------|--------|--------|--------|--------|--------|----------------|------------------|-------------|
|0|477|10677|1420|335|1685|4|128|4|
|1|278|11445|823|285|1603|2|128|4|
|2|205|9476|712|279|1548|1.1|128|4|
|3|188|5918|717|223|1542|1.1|1024|4|
|4|190|5660|698|231|1583|1.1|4096|4|
|5|203|5602|714|217|1563|1.1|1024|128|

Я изменял 3 параметра конфигурации сервера:
- random_page_cost: у postgres задается "цена" случайного и последовательного доступа к памяти в виде относительных единиц, по дефолту стояли параметры 4 и 1. Так как это относительные величины, то достаточно менять только один параметр. Также я выяснил, что соотношение 4:1 не подходит для SSD-дисков, а у меня как раз SSD, и его надо уменьшать. Действительно, сведение этого отношения практически к уровню 1:1 время выполнения запросов уменьшилось. Причем лучшее всех ускорились Запрос 1 и Запрос 3 (примерно в 2 раза), тогда как наиболее тяжелый Запрос 2 ускорился примерно на 12%.
- shared_buffer: разделяемый буфер оперативной памяти. По дефолту стояло 128 Мб, поэтому я сходу увеличил его до 1 Гб, т.к. устройство располагает 16 Гб ОЗУ. Время выполнения Запроса 2 значительно уменьшилось, тогда как время остальных запросов изменилось мало. Скорее всего, остальные запросы уже достигли пика своей производительности и ускорять их уже некуда. Для интереса я попробовал выделить на разделяемый буфер 25% ОЗУ, т.е. 4 Гб (так как источники в интернете рекомендовали именное такое значение). Буфер вырос в 4 раза, однако полученный эффект оказался не особо фантастическим (часть запросов замедлилась, а ускорение оставшихся несоразмерно увеличению выделению ресурсов), поэтому в дальнешем я вернул значение этого параметра в размере 1 Гб.
- work_mem: данный параметр важен в задачах сортировки. Хоть у меня в запросах сортировка как таковая отсутствует, я все равно решил посмотреть, что произойдет при изменении этого параметра. Я увеличил его с дефолтных 4 Мб до 128 Мб. В целом, ситуация особо не изменилась.

Таким образом, после изменений конфигурации сервера postgres удалось получить следющие ускорения запросов:

||Запрос 1|Запрос 2|Запрос 3|Запрос 4|Запрос 5|
|-----------|--------|--------|--------|--------|--------|
|Ускорение|2.54|1.91|2.03|1.54|1.09|

Т.е., в среднем в 2 раза время выполнение запросов усменьшилось, довольно приятный результат.

Наконец, далее я перешел к изменению самих запросов. Далее будут описаны стратегии изменения запросов и итоговые SQL-скрипты этих запросов.

- Запрос 1. Здесь я решил попробовать сделать поменьше вложенных запросов, использовав join. Получился вот такой запрос:
```sql
WITH AvgCount AS (
    SELECT AVG(Counted) AS AvgCount
    FROM (
        SELECT user_id, COUNT(*) AS Counted
        FROM app_user
        GROUP BY user_id
    ) AS SubQuery
)
SELECT u.username, u.pwd
FROM "user" u
JOIN app_user au ON u.id = au.user_id
GROUP BY u.id
HAVING COUNT(*) > (SELECT AvgCount FROM AvgCount);
```
Однако, время работы такого запроса составило 244 мс, что хуже изначального варианта.
- Запрос 2. Запрос подразумевает поиск объекта с максимальной характеристикой, так что столь длинный запрос можно сильно упростить, если использовать DENSE_RANK и ORDER BY.
```sql
SELECT genre, app_count
FROM (
    SELECT 
        genre, 
        COUNT(DISTINCT app.id) AS app_count,
        DENSE_RANK() OVER (ORDER BY COUNT(DISTINCT app.id) DESC) as rank
    FROM 
        app 
    JOIN 
        "session" ON app.id = "session".app_id 
    GROUP BY 
        app.genre
) AS ranked_genres
WHERE rank = 1;
```
Время выполнения такого запроса составило 4691 мс, что на целую секунду меньше, чем предыдущий лучший результат.
- Запрос 3. В этом запросе похожая идея, только с использованием ROWNUMB
```sql
SELECT email FROM profile WHERE user_id in (WITH RankedComments AS (
    SELECT user_id, COUNT(stars) AS Counted_Stars,
           ROW_NUMBER() OVER (ORDER BY COUNT(stars) DESC) AS Rank
    FROM "comment" 
    JOIN app ON "comment".app_id = app.id 
    WHERE stars = 1 AND genre = (
        SELECT genre FROM "comment" 
        JOIN app ON "comment".app_id = app.id 
        GROUP BY genre 
        HAVING AVG(stars) = (
            SELECT MAX(mean) FROM (
                SELECT genre, AVG(stars) AS mean 
                FROM "comment" 
                JOIN app ON "comment".app_id = app.id 
                GROUP BY genre
            ) AS SubQuery
        )
    ) 
    GROUP BY "comment".user_id
)
SELECT user_id
FROM RankedComments
WHERE Rank = 1
);
```
Использование сего прекрасного оператора уменьшило время выполнения запроса до 353 мс, т.е. еще в 2 раза.
- Запрос 4. Здесь же, чтобы исбавиться от дублирования одних и тех же запросов, было использовано CTE. Создание временных таблиц позволяет не выполнять постоянно один и тот же запрос, а просто вычленять из него необходимые данные.
```sql
WITH UserCounts AS (
    SELECT app_id, COUNT(user_id) AS user_count
    FROM app_user
    GROUP BY app_id
),
AvgUserCount AS (
    SELECT AVG(user_count) AS avg_user_count
    FROM UserCounts
),
SessionApps AS (
    SELECT app_id
    FROM "session"
    WHERE start_date BETWEEN CURRENT_DATE - INTERVAL '1 week' AND CURRENT_DATE
),
FilteredApps AS (
    SELECT app_id
    FROM UserCounts
    WHERE user_count < (SELECT avg_user_count FROM AvgUserCount)
    INTERSECT
    SELECT app_id FROM SessionApps
)
SELECT genre, AVG(stars)
FROM app
JOIN "comment" ON app.id = "comment".app_id
WHERE app.id IN (SELECT app_id FROM FilteredApps)
GROUP BY genre;
```
Время выполнения запроса уменьшилось до 154 мс.
- Запрос 5. Последний запрос также был переработан с применением CTE.
```sql
WITH session_summary AS (
    SELECT
        app_id,
        SUM(EXTRACT(EPOCH FROM (end_date - start_date)) / 3600) AS time_in_app
    FROM
        "session"
    GROUP BY
        app_id
),
max_time_in_app AS (
    SELECT
        MAX(time_in_app) AS max_time
    FROM
        session_summary
)
SELECT
    title
FROM
    app
JOIN
    session_summary ON app.id = session_summary.app_id
JOIN
    max_time_in_app ON session_summary.time_in_app = max_time_in_app.max_time;
```
Время выполнения запроса составило 1314 мс.

Таким образом итоговое ускорение запросов составило:

||Запрос 1|Запрос 2|Запрос 3|Запрос 4|Запрос 5|
|-----------|--------|--------|--------|--------|--------|
|Ускорение|2.54|2.27|4.02|2.18|1.28|

Очень интересные результаты. Скорее всего, последний запрос в целом составлен так, что как его ни реализуй, ему ускориться особо некуда, тогда как запрос 3 изначально был сделан крайне неэффективно, и изменение структуры запроса сразу дало двукратное ускорение (к тому же, при конфигурации сервера мы увеличили work_mem, важный при выполнении оператора ORDER BY). Ну и самый прожорливый запрос удалось достаточно неплохо ускорить, более чем в 2 раза.
