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
2. 
