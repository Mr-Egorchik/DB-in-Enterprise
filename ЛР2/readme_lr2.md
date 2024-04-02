![изображение](https://github.com/Mr-Egorchik/DB-in-Enterprise/assets/78895530/a8441b5b-699d-49fd-9d3e-ae02e6a42bc8)# Лабораторная работа 2
### Выполнил студент группы 6133-010402D Читоркин Егор

Продолжаем мучить магазин приложений. Теперь мы хотим научиться понимать, надо ли магазину рекомендовать приложение. У нас есть хороший датасет с 500 приложениями и большим количеством информации. Выберем наиболее интересные параметры, которые могут пригодиться:

- Жанр
- Количество пользователей
- Количество запущенных сессий
- Общее время сессий
- Количество покупок в приложении
- Общая сумма всех покупок в приложении
- Общее количество отзывов
- Количество отзывов с оценкой 1⭐
- Количество отзывов с оценкой 2⭐
- Количество отзывов с оценкой 3⭐
- Количество отзывов с оценкой 4⭐
- Количество отзывов с оценкой 5⭐
- Цена приложения

Для этого был реализован следующий SQL-запрос:
```sql
WITH UserCounts AS (
    SELECT app_id, COUNT(user_id) AS user_count
    FROM app_user
    GROUP BY app_id
),
SessionCounts AS (
    SELECT app_id, COUNT(id) AS session_count, SUM(EXTRACT(EPOCH FROM (end_date - start_date))) AS time_in_app
    FROM "session"
    GROUP BY app_id
),
CashCounts AS (
	SELECT app_id, COUNT(id) as payment_count, SUM(cash) AS pays_to_app
	FROM payment
	GROUP BY app_id
),
CommentCounts AS (
	SELECT
    	app_id,
		COUNT(*) AS total_reviews,
		COUNT(CASE WHEN stars = 1 THEN 1 END) AS one_star,
		COUNT(CASE WHEN stars = 2 THEN 1 END) AS two_stars,
		COUNT(CASE WHEN stars = 3 THEN 1 END) AS three_stars,
		COUNT(CASE WHEN stars = 4 THEN 1 END) AS four_stars,
		COUNT(CASE WHEN stars = 5 THEN 1 END) AS five_stars
	FROM comment
	GROUP BY app_id
)
SELECT 
    app.id, 
    app.title, 
    app.genre,
    COALESCE(UserCounts.user_count, 0) AS user_count,
    COALESCE(SessionCounts.session_count, 0) AS session_count,
    COALESCE(SessionCounts.time_in_app, 0) AS total_time,
    COALESCE(CashCounts.payment_count, 0) AS payment_count,
    COALESCE(CashCounts.pays_to_app, 0) AS total_cash,
    COALESCE(CommentCounts.total_reviews, 0) AS total_reviews,
    COALESCE(CommentCounts.one_star, 0) AS one_star,
    COALESCE(CommentCounts.two_stars, 0) AS two_stars,
    COALESCE(CommentCounts.three_stars, 0) AS three_stars,
    COALESCE(CommentCounts.four_stars, 0) AS four_stars,
    COALESCE(CommentCounts.five_stars, 0) AS five_stars,
    app.price,
    app.recommend
FROM 
    app
LEFT JOIN 
    UserCounts ON app.id = UserCounts.app_id
LEFT JOIN 
    SessionCounts ON app.id = SessionCounts.app_id
LEFT JOIN 
    CashCounts ON app.id = CashCounts.app_id
LEFT JOIN 
    CommentCounts ON app.id = CommentCounts.app_id;

```

Думаю, комментарии по запросу излишни, ибо мы просто собираем всякие данные из разных табличек, связывая их с приложением.

Далее начинаем питонить. Выбранный классификатор - RandomForestClassifier.

Было бы неплохо знать еще и средний рейтинг приложения. Запускать sql-запрос заново не хочется, поэтому получим это с помощью библиотеки pandas.

```py
df['mean_rate'] = (df['one_star'] + 2*df['two_stars'] + 3*df['three_stars'] + 4*df['four_stars'] + 5*df['five_stars']) / df['total_reviews']
```

Также выделим из этого датасета признаки и целевую переменную, а категориальную переменную `genre` закодируем с помощью One-Hot encoder

```py
data = df.drop(['id', 'title', 'recommend'], axis=1)
data = pd.get_dummies(data=data, columns=['genre'], drop_first=False)
target = df['recommend']
```

Далее делим данные на обучающие и тестовые
```py
X_train, X_test, y_train, y_test = train_test_split(data, target, test_size = 0.3, random_state = 42)
```

Создаем baseline-модель, чтобы было с чем сравнивать
```py
baseline = RandomForestClassifier()
baseline.fit(X_train, y_train)
```

Получаем следующий `classification report`:

||precision|recall|f1-score|support|
|-|---------|------|--------|-------|
|False|0.98|0.97|0.97|116|
|True|0.89|0.94|0.91|34|
|accuracy|||0.96|150|

Слишком хорошо, постараемся этот результат хотя бы не ухудшить.

Отметим, что очень много признаков в датасете, хотелось бы их как-то уменьшить. Для этого используем `SelectKBest`, считая метрики как среднее по `cross_val_score`:

```py
crossval_score = []

for i in range(1, data.shape[1] + 1):
    skb = SelectKBest(score_func=f_classif, k=i)
    Xreduced = skb.fit_transform(data, target)
    scores = cross_val_score(baseline, Xreduced, target, cv = 5)
    crossval_score.append(scores.mean())
```

Получаем вот такой незамысловатый график:
![Мега-график](/ЛР2/img/plot.png)

Видим максимум в точках $k=9, 10$. Далее работаем с $k=9$ (мы же хотели уменьшить количество признаков, поэтому берем меньшее значение).

Теперь перейдем к подбору гиперпараметров. Будем подбирать
- n_estimators - число «деревьев» в «случайном лесу»
- max_features - число признаков для выбора расщепления
- min_samples_leaf - минимальное число объектов в листьях

Для поиска оптимальных гиперпараметров используем `RandomSearchCV`. Он быстрее обычного `GridSearchCV` и дает значения, близкие к оптимуму.

```py
clf = RandomizedSearchCV(baseline, grid, random_state=42)
search = clf.fit(X_new, target)
```

Получаем следующие значения
- n_estimators - 90
- max_features - 2
- min_samples_leaf - 6

Наконец, тестируем получившуюся после всех танцев с бубном модель. И надеемся, что стало хотя бы не хуже...

||precision|recall|f1-score|support|
|-|---------|------|--------|-------|
|False|0.97|0.98|0.98|116|
|True|0.94|0.91|0.93|34|
|accuracy|||0.97|150|

Accuracy даже стало побольше!
