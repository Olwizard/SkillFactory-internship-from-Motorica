# SkillFactory-internship-from-Motorica
### Задача стажировки:
На входе мы имеем значения 50 сенсоров с оптомиографической манжеты и значения жестов которые выполняются в данный момент по таймингу.
Необходимо создать ML модель которая на основании значений сенсоров могла бы максимально точно распознавать жесты и передавать их на управление протеза.

#### Решение задачи:

Изначально была предпринята попытка подобрать гиперпараметры для модели обучения на комбинации значений датчиков от всех пилотов. Собранные в единое целое данные из 31 файла в итоговом варианте получили очень большой размер и долгое время на подбор гиперпараметров.
Успех этот вариант решения задачи не имел. Максимальное усреднение гиперпараметров и обучение модели привело к показаниям метрики в районе 0.63, что явно было меньше установленного на видеовстрече порога в 0.84.
Попытка подбора гиперпараметров на только одном варианте набора данных для каждого пилота и эксперимента так же не привела к улучшению итоговой метрики.

Приняв во внимание, что данные с оптомиографической манжеты на отборочном соревновании по практике имели временной сдвиг и для достижения хороших результатов метрики было необходимо устранить этот сдвиг. Было проведено более детальное исследование данных. 
Первичный анализ данных дал следующие результаты:
1. Данные от разных пилотов и разных экспериментов имели существенное отличие друг от друга по амплитуде работающих датчиков и плотности распространения сигналов. Можно сделать предположение, что различие в сигналах обусловлено не только вариативностью одевания ОМГ манжеты, но и разным аппаратным обеспечением самих датчиков и протезов.
2. В данных были обнаружены датчики которые не участвовали в процессе формирования общей частотной характеристики работы ОМГ манжеты. Были выявлены датчики которые в принципе не работали на всех экспериментах и датчики которые не работали в данном конкретном эксперименте. Значения таких явно можно было причислить к фоновому шуму, который по предположениям мог негативно влиять на процесс обучения ML модели и в итоге на качество распознавания жестов.
3. В данных явно прослеживался временной сдвиг значений датчиков и жестов.
4. Было замечено, что датчики имели всплески по амплитуде значений до момента начала изменения жестов, и во время изменения жестов. Что так же могло негативно влиять на итоговую метрику.
       
#### Была поставлена задача на поиск способов корректировки данных и устранения фонового шума.
	
Для выявления времени начала реакции датчиков был применён метод дифферента значений сенсоров. Который показал явные выбросы сигналов на которые можно было бы опираться. 
Чтобы удалить выбросы значений датчиков до начала изменения жестов было определено время начала изменения жестов и относительно него все значения датчиков до этого значения были обнулены.
	
Относительно времени когда был выброс значения сенсоров при обработке сигналов через дифферент и временем начала изменения значений жестов был определён временной сдвиг. Написана функция для корректировки сигналов жестов на это значение.

  На ранних версиях процесса обработки данных был придуман и реализован  алгоритм фильтрации рабочих и не рабочих сенсоров на методе квантильного распределения. Были выбраны квантили 0, 30, 50, 70. На отсортированных таким способом датчиках обучалась модель и в конце сравнивалась метрика по каждому варианту. В итоговый результат выводилась лучшая. 
Данный способ показал хороший результат на отборе работающих датчиков, но занимал в итоге около двух минут на все четыре варианта.

Для решения поставленной задачи проверили несколько моделей машинного обучения для выявления наилучшего варианта.

**Задейтвованные ML модели:**
* Классификатор опорных векторов ***SVC***.
* ***LogisticRegression***
* ***RandomForestClassifier***
* ***CatBoost***
* Нейронная сеть на базе ***tensorflow***
* 
Две крайние модели были исключены из дальнейшего исследования.
* Первая по причине долгого процесса обучения на CPU. Быстро получалось только на GPU но наличие его на итоговом оборудовании не гарантировалось
* Вторая по причине посредственных результатов по сравнению с классическими моделями ML

Заметив, что три отобранные модели в некоторых экспериментах показывали лучшие результаты относительно друг друга — был реализован алгоритм последовательного обучения каждой из них, с последующим сравнением результатов и вывода в итоговом решении лучшей модели.

***Результат в процентном соотношении получился такой:***

* ***SVC 19%***
* ***LogisticRegression 16%***
* ***RandomForestClassifier 65%***
      
Исходя из явного преобладания в итоговых лучших результатах модели ***RandomForest*** эта модель машинного обучения была выбрана в качестве основной для дальнейшей работы
	
Когда были получены значения исправленных жестов и исходя из того, что длина всех (кроме нулевых) жестов равняется 30 миллисекундам, была написана функция которая на интервалах изменения значений жестов усредняла значения датчиков для устранения выбросов.
Проведённая обработка сигналов датчиков сразу улучшила значения метрики с 0.63 до 0.9 и более.

При дальнейшем исследовании данных был реализован алгоритм увеличения амплитуды значений датчиков для более выраженной итоговой частотной характеристики. Что дало прирост в метрике на разных экспериментах от 0.08 до 0.1

На обработанных данных для ***ML*** модели ***RandomForestClassifier*** были подобраны гипер параметры с ограничением по количеству деревьев 100 и глубине дерева 50 так как увеличение этих значений вело к заметному времени обучения и в итоге могло отрицательно сказаться на попадание в отведённый тайминг на ***ofline*** тесте.

Препроцессинг и модель работают и показывают хорошие результаты но время на обучение модели тратится не позволительно много. Был переписан алгоритм отбора рабочих датчиков. В итоге был реализован функционал который на основании порогового значения по среднему максимальному от всех датчиков и среднему максимальному по каждому датчику отсортировал их таким же надёжным способом и за гораздо более короткое время. Так как из процесса был исключен вариант обучения модели и сравнения на конкурентной основе. Вместо двух минут работы предыдущего алгоритма получили 10 секунд.

Но и это время показалось не позволительно большим для итоговой модели. Весь код был из ***Pandas*** переписан в ***Numpy*** и оптимизирован. В итоге получил время на препроцессинг и обучение модели 0.4 секунды.

При реализации ***Ofline*** теста решалась задача применения всего препроцессинга который был реализован на тренировочных данных и основывался на интервалах значений датчиков. Он должен был быть применён на тестовых данных да и к тому же построчно. В каждый конкретный момент предсказания модели нам давалось всего по одному значению в количестве всех датчиков, вместо всего набора данных.

* Отбор работающих датчиков был реализован на удалении по индексам из ***np.array*** из предварительно собранного списка при обучении модели.
* Применение дальнейшего препроцессинга к данным был реализован через словарь, где ключами были значения датчиков построчно в виде кортежа на не обработанных данных, а значениями - обработанные, так же построчно и в соответствии с таймингом. Для исключения варианта аварийного завешения кода при условии входных данных не соответствующих имеющимся ключам словаря, через блок ***try/except*** была реализована обработка ошибки с назначением нулевого значения жестов.

Выбранный алгоритм обработки данных и обучения ML модели при прохождении ***Ofline*** теста показал правильность выбранной стратегии. Чему свидетельствуют высокая метрика и полное попадание в отведённый тайминг.
