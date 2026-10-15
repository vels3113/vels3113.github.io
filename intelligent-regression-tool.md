

# Хранение и обработка результатов тестовых симуляций для сравнения версий ядра

### Контекст

Тестовые симуляции генерировали большие объёмы выходных данных, которые нужно было **хранить, анализировать, воспроизводить и сравнивать** между разными версиями вычислительного ядра (регрессия точности/производительности/стабильности).
Выходные данные приходили в **индустриальных форматах**: текстовые отчёты + сырые файлы с рассчитанными кривыми.

### Задача

Организовать структурированное хранение и обработку результатов симуляций, чтобы:

* сравнивать серии экспериментов с гибкой настройкой (по схемам, параметрам, версиям, режимам);
* строить визуализации и статистику;
* менять аналитическую обработку **без перезапуска** всей цепочки (симуляции дорогие по времени).

---

## Что именно сделал

### 1) Нормализация и парсинг «сырого» вывода

* Выделил стабильный **контракт данных**: что является идентификатором эксперимента (netlist/параметры/режим/seed/версия ядра/конфиг MKL/число потоков).
* Реализовал парсеры:

  * для **текстовых отчётов** (метрики сходимости/итерации/ошибки/время/предупреждения);
  * для **кривых** (ось X: время/частота; ось Y: величины; единицы измерения; масштаб).
* Сделал устойчивость к вариациям форматов (разные версии вывода): строгая валидация + понятные диагностические сообщения (что именно не распарсилось и почему).

> *Как вы решали проблему эволюции формата вывода (новые поля, новые секции отчёта)? Была ли версия схемы/парсера?*
>
> Препроцессор входных файлов добавлял необходимые вызовы вывода данных, чтобы данные точно были в отчете или формате данных. Форматы данных - давний стандард, так что не было предусмотрено проверки версии модуля вывода

> *Что происходило, если парсер обнаруживал частично повреждённый вывод или неожиданный формат? (skip/mark failed/retry)*
>
>Парсер работал строго:
>* если критические секции отсутствовали — run помечался как failed/invalid, метрики не считались;
>* если отсутствовали второстепенные поля — сохраняли частичный результат с признаком неполноты.
>  Это важно, чтобы не “замусоривать” регрессионный анализ плохими данными и не получать ложные регрессии.

> *Были ли автотесты на парсинг и расчёт метрик? Какой минимальный набор фикстур держали, чтобы ловить регрессии анализатора?*
>
>Да, держали небольшой набор фикстур:
>* минимальный корректный отчёт,
>* кривые с разной сеткой и проверкой интерполяции/сравнения.
>
>  Pytest-ами проверяли, что парсер стабильно извлекает метрики и что расчёт ошибок (relative/absolute max) даёт ожидаемые значения.

### 2) Модель данных “DWH-подход” 

Сделал схему хранения, чтобы легко строить сравнения и отчёты:

* **Dim-таблицы (измерения)**:

  * `dim_run`: версия ядра, git hash, build flags, режимы параллелизма, версия MKL, окружение
  * `dim_case`: схема/набор параметров/тип анализа (time/freq)
  * `dim_signal`: имя сигнала/единицы/тип (V/I/…)
* **Fact-таблицы**:

  * `fact_metrics`: время выполнения, пик памяти, итерации, статус (ok/diverged)
  * `fact_curve_index`: ссылки на файлы с кривыми + их метаданные, количество точек, средняя относительная ошибка
  * при необходимости — агрегаты (min/max/rms/error bands), чтобы не таскать гигантские массивы в каждый отчёт

Это позволяло:

* быстро фильтровать/сравнивать результаты на уровне метрик;
* подгружать «тяжёлые» кривые только когда реально нужно.

> *“Средняя относительная точность” — это относительно чего? baseline-ядра, аналитического решения, другой библиотеки?*
>
> В каждом запуске предоставлялся baseline результат (часть препроцессинга)

> *Как вы определяли **relative error**, если значения близки к нулю?*
>
> были допметрики абсолютной максимальной разницы

> *Если у двух запусков разное число точек в кривой, как сравнивали? (ресемплинг/интерполяция/сравнение по ключевым точкам)*
>
>Мы сводили кривые к общему представлению перед сравнением. Если сетки по X различались, то делали интерполяцию (линейную) значения Y на точки первой кривой.
>
>  Дальше считали метрики по совпадающим X-точкам: среднюю относительную ошибку и max абсолютную разницу. Если кривые имели разные диапазоны X, то отмечался отдельный warning.


> *Какой был подход к хранению кривых: в PostgreSQL (bytea/array) или ссылки на файлы с индексом в БД? Почему?*
>
> В БД хранились ссылки на файлы, чтобы другие разработчики и инженеры могли работать с результатами стандартными инструментами или своими инструментами, разработанными под форматы файлов

### 3) Аналитический слой на Python (быстро менять анализ)

* Слой загрузки → `Pandas` DataFrame с нормализованными колонками.
* Сравнения “серий экспериментов” делались как:
  * группировки по ключам (`groupby` по case/run/version),
  * джойны baseline vs candidate,
  * расчёт метрик регрессии (ошибка по кривым, отклонения по времени, процент расхождений).
* Визуализация:
  * overlay кривых baseline/candidate в областях наибольшего расхождения
  * heatmap/boxplot по метрикам на батче схем,
  * отчёты “top regressions”.


> *Какие запросы были самыми тяжёлыми в PostgreSQL (group-by по сериям, join baseline/candidate)? Как оптимизировали: индексы, партиционирование, материализованные представления?*
>
>Самые тяжёлые были:
>* агрегаты по группам (по схемам/параметрам/машинам/числу потоков).
>
>Оптимизация:
>* нормализовали таблицы: `runs`, `cases`, `metrics`, `artifacts`;
>* индексы на `run_id`, `case_id`, `(kernel_version, machine, threads)`, и на поля фильтрации в Superset;
>* для типовых дешбордов использовали предагрегацию (материализованные представления или периодические summary-таблицы), чтобы не гонять group-by.

> *Как обеспечивали, чтобы Superset-дэшборды не “убивали” базу при широких фильтрах? (кэш, pre-aggregation, row-level security, лимиты)*
>
>Мы давали Superset работать с агрегированными таблицами/вью, а не с “сырыми” метриками; включали кэширование на уровне Superset/БД и следили за планами запросов (EXPLAIN) для самых популярных дашбордов.

> *Какие пороги регрессии считались “критичными” (runtime, iterations, accuracy)? Где это было зафиксировано: код, конфиг, таблица правил?*
>
>Пороги были конфигурируемые (в конфиге анализа), потому что разные типы схем имели разную чувствительность.
>
>Типично:
>* runtime regression: >X% (например 10–20% для критичных бенчей),
>* iterations: рост выше порога,
> * accuracy: средняя относительная ошибка > T или max abs diff > A.
>  В Superset пороги использовались как фильтры/алерты на дашбордах, а формально они были заданы в конфигурации анализа, чтобы можно было менять критерии без пересчёта симуляций.


### 4) Воспроизводимость и “не перезапускать флоу”

* Для каждого запуска сохранял:

  * точный набор входов (версия схемы/параметры),
  * версию ядра/конфиги,
  * хэш файлов результатов,
  * состояние окружения (минимально: CPU модель, число потоков).
* Аналитика работала по уже сохранённым артефактам: можно было менять формулы/метрики/графики без пересчёта симуляций.

> *Как выглядел уникальный ключ эксперимента? Что входило в идентификатор run: версия ядра, параметры, конфиг потоков, seed, формат вывода?*
> Версия ядра, параметры, для которых варьировались значения, количество потоков, машина

---

## Результат (что можно говорить на скрининге)

* Появился воспроизводимый “dataset” для регрессии: сравнение версий ядра стало системным, а не ручным.
* Ускорилась диагностика проблем: стало видно, где регрессия по точности/времени, и на каких классах схем.
* Анализ стал итеративным: новые критерии/визуализации добавлялись без перезапуска симуляций.


# Structure
image: mock superset dashboard
1. One more time on the challenge: outline configurable simulation run keys, input files
2. Proposed solution: parsing + processing + vizualization
image: application architecture
input -> configuring —> running -> parsing -> visualizing 
3. Caveats: collaboration and decision-making
4. Impact: faster and higher quality feature delivery, less negative customer feedback

## Edit Core Algorithms, Monitor Core Behaviors

For simulation software developers, the real art lies in validating new implementation. Especially, when hundreds and thousands of engineers rely on your software daily. 

My team was dealing with the same bottleneck each release cycle: even minor changes to core simulation algorithms demanded thorough testing and analysis of thousands of simulation results. We lacked a tool which would monitor simulation characteristics and track how they varied with input parameters across different configurations — test cases, builds (identified by git hashes), and machines.

## Breaking Down Regression Testing 

Although the team had an established regression testing system, it lacked the flexibility needed for in-depth core algorithm testing. I proposed a new modular workflow to improve configurability and enable targeted metrics for specific changes.

The new system's configuration layer consisted of three components: a test preprocessor, runner, and results parser. A Python application reads a single config file with git hash for building the simulation package, test paths, and the simulation and environment parameters to vary. It generates all necessary files and orchestrates the simulation runs. Upon completion, it parses run characteristics from industry-standard report files into a PostgreSQL database.

Once simulation results are in the database, the analysis component calculates useful aggregates of runs such as max runtime across multiple runs or simulation parameter values, and avg relative error on pre-selected waveforms. 

## Exploring Testing Results with the Team

Beyond testing automation, the application streamlined our feature validation workflow. It generated Superset dashboard based on the user-defined templates, accessible through the team's Superset server instance. All the developers were able to see reports and reports' source files and create custom views of the results database. We visualized pre-defined testing metrics and explored new ones directly through the Superset interface.

## What did the Tool Bring After All?
The tool transformed our development process in several key ways. It established a standardized approach for setting up tests and analyzing results, dramatically accelerating work for both developers and QA engineers. By making core algorithm validation more transparent, it simplified communication of development status across teams and to management. Ultimately, it enabled better-informed development decisions and helped us deliver a higher-quality product to our customers.
