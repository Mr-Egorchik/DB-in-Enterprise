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
