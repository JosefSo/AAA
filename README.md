# Course Allocation with Social Utility (HBS-style Draft + Friends)

Это проект (дипломная работа) про распределение студентов по курсам при ограниченной вместимости курсов.
Идея: студент выбирает курс не только по личной полезности (предпочтения), но и с учётом того,
что на курсе могут оказаться его друзья.

Проект реализует **HBS-style snake draft** (как в Budish–Cantillon механизме), но добавляет
**социальный бонус** за совпадение с друзьями. Выход — allocation + метрики справедливости.

---

## 1) Что решаем

Есть:
- множество студентов `S`
- множество курсов `C`
- вместимость каждого курса `cap(c)` (сколько мест)
- каждый студент должен получить до `b` курсов (например `b=3`)

Есть входные таблицы:
- **Table 1 (Individual preferences)**: для каждого `(s, c)` есть `PositionA(s,c)` (ранг курса для студента) и `Score(s,c)` (сырое число; в коде оно используется только как tie-break).
- **Table 2 (Friend preferences)**: для каждого направленного `(s, f, c)` есть `PositionB(s,f,c)` — насколько студент `s` хочет быть с другом `f` именно на курсе `c` (ранг среди друзей).
- **Table 3 (Lambda)**: `λ_s` — насколько студенту важны друзья (если нет — берём default).

---

## 2) Одна главная формула (то, что максимизируем при выборе курса)

### 2.1 Utility при выборе курса в моменте (per-pick)
Когда студент `s` выбирает следующий курс `c`, он оценивает:

\[
U(s,c) = Base(s,c) + \lambda_s \cdot FriendBonus(s,c)
\]

- `Base(s,c)` — личная полезность курса (по таблице 1)
- `FriendBonus(s,c)` — бонус за друзей, которые **уже** сидят на этом курсе (реактивная версия)
- `λ_s` — вес “социальности” студента

Это именно то, что делает код: `total = base + lambda_s * friend_bonus`.  
(см. `_utility_components`):contentReference[oaicite:3]{index=3}

---

## 3) Как мы превращаем ранги в числа (Rank → Utility)

В таблицах предпочтения записаны **рангами** (`PositionA`, `PositionB`), а алгоритму нужны числа.
Поэтому мы используем функцию `posU(p, K)` (из README и соответствует использованию в проекте):

\[
posU(p, K) =
\begin{cases}
0, & p \text{ missing} \\
1, & K \le 1 \land p = 1 \\
0, & K \le 1 \land p \ne 1 \\
\frac{K - p}{K - 1}, & K > 1
\end{cases}
\]

Интуиция:
- лучший ранг `p=1` даёт `1`
- худший ранг `p=K` даёт `0`
- всё между — линейная шкала в `[0,1]`  
:contentReference[oaicite:4]{index=4}

### Мини-пример
Если курсов `|C| = 5`, то:
- `p=1 → 1.00`
- `p=2 → 0.75`
- `p=3 → 0.50`
- `p=4 → 0.25`
- `p=5 → 0.00`

---

## 4) Из чего состоит `U(s,c)` (разбираем на части)

### 4.1 Base(s,c) — личная полезность курса
Берём `PositionA(s,c)` и переводим в `[0,1]`:

\[
Base(s,c) = posU(PositionA(s,c), |C|)
\]

Если предпочтения для `(s,c)` отсутствуют — `Base=0`.:contentReference[oaicite:5]{index=5}:contentReference[oaicite:6]{index=6}

### 4.2 Pref(s,f,c) — “насколько хочу быть с другом f на курсе c”
Это направленная (asymmetric) величина: `s` может хотеть быть с `f`, но не наоборот.

\[
Pref(s,f,c) = posU(PositionB(s,f,c), K_{friend})
\]

`K_friend` — размер шкалы рангов друзей (обычно “сколько друзей/позиций”).:contentReference[oaicite:7]{index=7}:contentReference[oaicite:8]{index=8}

### 4.3 FriendBonus(s,c) — реактивный бонус за друзей
Ключевой момент: бонус **реактивный**. Он учитывает только тех друзей, которые **уже получили** курс `c`
к моменту выбора `s`.

\[
FriendBonus(s, c) = \sum_{f \in F(s)} \mathbb{1}[c \in A_f] \cdot Pref(s,f,c)
\]

где `A_f` — текущий набор курсов друга `f` (на момент выбора).  
Это прямо совпадает с `_friend_bonus_reactive` в коде.:contentReference[oaicite:9]{index=9}:contentReference[oaicite:10]{index=10}

---

## 5) Как студент реально выбирает курс (feasible set + tie-break)

### 5.1 Какие курсы вообще можно выбрать (feasible set)
Студент не может:
- выбрать курс без мест
- выбрать курс, который уже у него есть

\[
C_s = \{ c \in C \mid cap\_left(c) > 0 \ \land \ c \notin A_s \}
\]
:contentReference[oaicite:11]{index=11}:contentReference[oaicite:12]{index=12}

### 5.2 Правило выбора (детерминизм + tie-break)
Мы берём курс с максимальным значением по лексикографическому ключу:

\[
c^\* = \arg\max_{c \in C_s}
\Big(
\text{round}(U(s,c), 9),
-PositionA(s,c),
Score(s,c),
rnd(s,c),
CourseID(c)
\Big)
\]

Интуитивно это значит:
1) максимизируем `U(s,c)` (и “склеиваем” почти равные значения округлением до `1e-9`)  
2) если равны — предпочитаем **лучший ранг** (меньше `PositionA`)  
3) если равны — предпочитаем больше `Score`  
4) если равны — seeded random  
5) если равны — стабильный `CourseID`  
:contentReference[oaicite:13]{index=13}:contentReference[oaicite:14]{index=14}

---

## 6) Алгоритм: Phase A (HBS snake draft)

1) перемешиваем студентов фиксированным seed  
2) делаем `draft_rounds` раундов  
3) в нечётных раундах порядок прямой, в чётных — обратный (“snake”)  
4) каждый студент делает один pick по правилу выше

Это реализовано в `_run_initial_draft`.:contentReference[oaicite:15]{index=15}

---

## 7) Post-phase (после драфта): два режима улучшений

После драфта можно включить `post_iters` итераций улучшения:

### 7.1 Mode = `swap` (локальный поиск)
Ищем лучший обмен “курс ↔ курс” между двумя студентами, который увеличивает глобальное благосостояние.

Для оценки используется **order-independent welfare** по финальной аллокации:

\[
W_s = \sum_{c \in A_s}
\left[
Base(s,c) + \lambda_s \cdot \sum_{f \in F(s)} \mathbb{1}[c \in A_f] \cdot Pref(s,f,c)
\right]
\]
\[
W = \sum_{s \in S} W_s
\]

И применяем swap только если \(\Delta W > 0\).  
(Это соответствует `_student_welfare` и логике “apply best improving swap”.):contentReference[oaicite:16]{index=16}:contentReference[oaicite:17]{index=17}:contentReference[oaicite:18]{index=18}

### 7.2 Mode = `add-drop` (HBS-style)
Делаем проход по студентам в случайном порядке. Для каждого студента:
- кандидаты = его текущие курсы + любые курсы, где ещё есть места
- оцениваем каждый кандидат по `U(s,c)`
- оставляем топ-`b` курсов (и обновляем capacity)

Это реализовано в `_run_add_drop_improvement`.:contentReference[oaicite:19]{index=19}:contentReference[oaicite:20]{index=20}

---

## 8) Что именно измеряем (метрики)

После финального распределения считаем:
- суммарную полезность
- Gini (неравенство) по нормализованной полезности
- базовые/социальные компоненты отдельно
- заполнение курсов, overlaps по друзьям и т.д.

Нормализация делается так:
- для каждого студента считаем `BaseSum_s`, `FriendSum_s`, `Total_s = BaseSum_s + λ_s*FriendSum_s`
- считаем верхние границы `MaxBase_s` и `MaxTotalUpper_s` (без учёта capacity и “reactive” ограничения)
- делим фактическое на upper bound → получаем значения в `[0,1]`
- уже по этим нормализованным считаем Gini

Код: `_compute_metrics`, где считаются `per_student_total_norm` и `compute_gini_index`.:contentReference[oaicite:21]{index=21}:contentReference[oaicite:22]{index=22}

---

## 9) Выходные файлы

- `allocation.csv` — результаты драфта (Phase A)
- `post_allocation.csv` — события post-phase (swap/add-drop)
- `summary.csv` — total utility + Gini по нормализованной total/base
- `metrics_extended.csv` — расширенные метрики (Jain, Theil, Atkinson, percentiles, fill-rate, overlaps и т.д.):contentReference[oaicite:23]{index=23}

---

## 10) Quick start

Требования: Python 3.10+, внешних зависимостей нет.

Сгенерировать пример входных таблиц:
```bash
python generate/generate_tables.py --students 200 --courses 8 --seed 42
