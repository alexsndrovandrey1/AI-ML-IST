# Практическая работа 5

## Цель Работы

1.  Изучить возможности СУБД Clickhouse для обработки и анализ больших
    данных.

2.  Получить навыки применения Clickhouse совместно с языком
    программирования R.

3.  Получить навыки анализа метаинфомации о сетевом трафике.

4.  Получить навыки применения облачных технологий хранения, подготовки
    и анализа данных: Managed Service for ClickHouse, Rstudio Server.

## Ход работы

Установим ClickHouseHTTP

``` r
install.packages("ClickHouseHTTP", repos = "https://cran.r-project.org")
```

    Устанавливаю пакет в 'C:/Users/Andrey/AppData/Local/R/win-library/4.3'
    (потому что 'lib' не определено)

    пакет 'ClickHouseHTTP' успешно распакован, MD5-суммы проверены

    Скачанные бинарные пакеты находятся в
        C:\Users\Andrey\AppData\Local\Temp\Rtmp442Ziy\downloaded_packages

Подключение

``` r
library(tidyverse)
```

    Warning: пакет 'ggplot2' был собран под R версии 4.3.2

    Warning: пакет 'dplyr' был собран под R версии 4.3.2

    ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ✔ dplyr     1.1.4     ✔ readr     2.1.4
    ✔ forcats   1.0.0     ✔ stringr   1.5.0
    ✔ ggplot2   3.4.4     ✔ tibble    3.2.1
    ✔ lubridate 1.9.3     ✔ tidyr     1.3.0
    ✔ purrr     1.0.2     
    ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ✖ dplyr::filter() masks stats::filter()
    ✖ dplyr::lag()    masks stats::lag()
    ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

``` r
library(ClickHouseHTTP)
```

    Warning: пакет 'ClickHouseHTTP' был собран под R версии 4.3.3

``` r
library(DBI)
connection <- dbConnect(
  ClickHouseHTTP::ClickHouseHTTP(),
  host="rc1d-sbdcf9jd6eaonra9.mdb.yandexcloud.net",
  port=8443,
  user="student24dwh",
  password="DiKhuiRIVVKdRt9XON",
  db="TMdata",
  https=TRUE,
  ssl_verifypeer=FALSE)
database<-dbReadTable(connection, "data")
df<- dbGetQuery(connection, "SELECT * FROM data")
```

## Задание 1: Надите утечку данных из Вашей сети

### Важнейшие документы с результатами нашей исследовательской деятельности в области создания вакцин скачиваются в виде больших заархивированных дампов. Один из хостов в нашей сети используется для пересылки этой информации – он пересылает гораздо больше информации на внешние ресурсы в Интернете, чем остальные компьютеры нашей сети. Определите его IP-адрес.

``` r
zad1 <- df %>%
  filter(!grepl('^1[2-4].*', dst)) %>%
  group_by(src) %>%
  summarise(bytes_amount = sum(bytes)) %>%
  top_n(n = 1, wt = bytes_amount)
cat(zad1$src)
```

    13.37.84.125

filter(!grepl(‘^1\[2-4\].\*’, dst)): В этой строке происходит фильтрация
строк, где столбец dst не начинается с “12”, “13” или “14”. Функция
grepl используется для выполнения поиска по регулярному выражению, а
символ ^ обозначает начало строки. Оператор ! отрицает соответствие,
поэтому оставляются только строки, не соответствующие заданному шаблону.

group_by(src): В этой строке происходит группировка оставшихся строк по
столбцу src. Это означает, что последующие вычисления будут выполняться
для каждого уникального значения в столбце src.

summarise(bytes_amount = sum(bytes)): В этой строке вычисляется сумма
столбца bytes для каждой группы (уникального значения в столбце src).
Результат сохраняется в новом столбце с именем bytes_amount.

top_n(n = 1, wt = bytes_amount): В этой строке выбирается верхняя строка
на основе столбца bytes_amount. Параметр n указывает, что нужно оставить
только верхнюю строку, а параметр wt указывает столбец, по которому
определяется верхняя строка.

Определите его IP-адрес - 13.37.84.125.

## Задание 2: Надите утечку данных 2

### Другой атакующий установил автоматическую задачу в системном планировщике cron для экспорта содержимого внутренней wiki системы. Эта система генерирует большое количество трафика в нерабочие часы, больше чем остальные хосты. Определите IP этой системы. Известно, что ее IP адрес отличается от нарушителя из предыдущей задачи.

``` r
zad2 <- df %>%
  select(timestamp, src, dst, bytes) %>%
  mutate(timestamp = hour(as_datetime(timestamp/1000))) %>%
  filter(!grepl('^1[2-4].*', dst) & timestamp >= 0 & timestamp <= 15) %>%
  group_by(src) %>%
  summarise(bytes_amount = sum(bytes)) %>%
  filter(src != "13.37.84.125") %>%
  top_n(1, wt = bytes_amount)

cat(zad2$src)
```

    12.55.77.96

select(timestamp, src, dst, bytes): Выбираем только столбцы timestamp,
src, dst и bytes из фрейма данных df. Это позволяет нам работать только
с необходимыми столбцами данных.

mutate(timestamp = hour(as_datetime(timestamp/1000))): С помощью функции
mutate преобразуем столбец timestamp в часовой формат. Для этого мы
сначала делим значения столбца на 1000, чтобы привести их к
миллисекундам, а затем используем функции as_datetime и hour для
преобразования временных меток в часы.

filter(!grepl(‘^1\[2-4\].\*’, dst) & timestamp \>= 0 & timestamp \<=
15): Фильтруем строки данных, исключая строки, где значение столбца dst
начинается с “12”, “13” или “14”. Используем регулярное выражение для
поиска соответствующих значений. Также фильтруем строки, где значение
столбца timestamp находится в диапазоне от 0 до 15, что соответствует
нерабочим часам.

group_by(src): Группируем данные по уникальным значениям в столбце src,
чтобы производить агрегацию на уровне каждого отправителя.

summarise(bytes_amount = sum(bytes)): С помощью функции summarise
создаем новый столбец bytes_amount, содержащий суммарное значение
столбца bytes для каждого отправителя.

filter(src != “13.37.84.125”): Исключаем строки, где значение столбца
src равно “13.37.84.125”, чтобы исключить известный IP-адрес.

top_n(1, wt = bytes_amount): Выбираем только одну строку с наибольшим
значением bytes_amount с помощью функции top_n. Это позволяет нам найти
IP-адрес с наибольшим объемом трафика в нерабочие часы.

Определите IP этой системы. Известно, что ее IP адрес отличается от
нарушителя из предыдущей задачи - 12.55.77.96

## Задание 3: Надите утечку данных 3

### Еще один нарушитель собирает содержимое электронной почты и отправляет в Интернет используя порт, который обычно используется для другого типа трафика. Атакующий пересылает большое количество информации используя этот порт, которое нехарактерно для других хостов, использующих этот номер порта. Определите IP этой системы. Известно, что ее IP адрес отличается от нарушителей из предыдущих задач.

``` r
zad3 <- df %>%
  select(src, port, dst, bytes) %>%
  filter(!str_detect(dst, '1[2-4].')) %>%
  group_by(src, port) %>%
  summarise(bytes_ip_port = sum(bytes), .groups = "drop") %>%
  group_by(port) %>%
  mutate(average_port_traffic = mean(bytes_ip_port)) %>%
  ungroup() %>%
  top_n(1, bytes_ip_port / average_port_traffic)
cat(zad3$src)
```

    12.30.96.87

filter(!str_detect(dst, ‘1\[2-4\].’)): Фильтрует строки, исключая те,
где значение в столбце dst соответствует шаблону ‘1\[2-4\].’. Это
позволяет исключить строки, где dst начинается с числа от 12 до 14.
group_by(src, port): Группирует строки по уникальным значениям в
столбцах src и port. summarise(bytes_ip_port = sum(bytes), .groups =
“drop”): Создает новый столбец bytes_ip_port, содержащий сумму значений
в столбце bytes для каждой группы. .groups = “drop” используется для
удаления информации о группах. group_by(port): Группирует строки по
уникальным значениям в столбце port. mutate(average_port_traffic =
mean(bytes_ip_port)): Создает новый столбец average_port_traffic,
содержащий среднее значение bytes_ip_port для каждой группы port.
ungroup(): Удаляет информацию о группировке. top_n(1, bytes_ip_port /
average_port_traffic): Оставляет только первую строку с наибольшим
отношением bytes_ip_port к average_port_traffic.

## Задание 4: Обнаружение канала управления

### Зачастую в корпоротивных сетях находятся ранее зараженные системы, компрометация которых осталась незамеченной. Такие системы генерируют небольшое количество трафика для связи с панелью управления бот-сети, но с одинаковыми параметрами – в данном случае с одинаковым номером порта. Какой номер порта используется бот-панелью для управления ботами?

``` r
zad4 <- df %>%
  group_by(port) %>%
  summarise(minBytes = min(bytes),
            maxBytes = max(bytes),
            diffBytes = max(bytes) - min(bytes),
            avgBytes = mean(bytes),
            count = n()) %>%
  filter(avgBytes - minBytes < 10 & minBytes != maxBytes) %>%
  select(port)
zad4
```

    # A tibble: 1 × 1
       port
      <int>
    1   124

group_by(port) - группирует данные по столбцу port, чтобы выполнить
агрегацию на уровне каждого уникального значения port.

summarise(minBytes = min(bytes), maxBytes = max(bytes), diffBytes =
max(bytes) - min(bytes), avgBytes = mean(bytes), count = n()) -
выполняет агрегацию данных, где:

minBytes - минимальное значение в столбце bytes в каждой группе.
maxBytes - максимальное значение в столбце bytes в каждой группе.
diffBytes - разница между максимальным и минимальным значениями в
столбце bytes в каждой группе. avgBytes - среднее значение в столбце
bytes в каждой группе. count - количество строк в каждой группе.
filter(avgBytes - minBytes \< 10 & minBytes != maxBytes) - фильтрует
результаты, оставляя только те группы, где разница между средним и
минимальным значением bytes меньше 10 и минимальное значение bytes не
равно максимальному.

select(port) - выбирает только столбец port в итоговом результате.

Какой номер порта используется бот-панелью для управления ботами - 124

## Задание 5: Обнаружение P2P трафика

### Иногда компрометация сети проявляется в нехарактерном трафике между хостами в локальной сети, который свидетельствует о горизонтальном перемещении (lateral movement). В нашей сети замечена система, которая ретранслирует по локальной сети полученные от панели управления бот-сети команды, создав таким образом внутреннюю пиринговую сеть. Какой уникальный порт используется этой бот сетью для внутреннего общения между собой?

``` r
zad5 <- dbGetQuery(connection, "
  SELECT port
  FROM (
    SELECT port, max(bytes) - min(bytes) as anomaly
    FROM data
    WHERE (src LIKE '12.%' OR src LIKE '13.%' OR src LIKE '14.%')
      AND (dst LIKE '12.%' OR dst LIKE '13.%' OR dst LIKE '14.%')
    GROUP BY port
  ) AS subquery
  ORDER BY anomaly DESC
  LIMIT 1
")
cat(zad5$port)
```

    115

dbGetQuery(connection, “SELECT port …”): Выполняет SQL-запрос к базе
данных через соединение connection. “SELECT port FROM …”: Запрос
выбирает столбец port из результирующей таблицы. SELECT port,
max(bytes) - min(bytes) as anomaly FROM data …: В подзапросе
группируются данные по столбцу port, а затем вычисляется разница между
максимальным и минимальным значением bytes для каждого порта. Разница
сохраняется в столбце anomaly. WHERE (src LIKE ‘12.%’ OR src LIKE ‘13.%’
OR src LIKE ‘14.%’) AND (dst LIKE ‘12.%’ OR dst LIKE ‘13.%’ OR dst LIKE
‘14.%’): Применяет фильтр к данным, оставляя только те строки, где src
или dst начинаются с чисел 12, 13 или 14. GROUP BY port: Группирует
данные по уникальным значениям в столбце port. ORDER BY anomaly DESC:
Сортирует результаты по столбцу anomaly в порядке убывания, чтобы порт с
наибольшим значением anomaly был первым. LIMIT 1: Ограничивает
результаты только одной строкой, чтобы получить порт с наибольшим
значением anomaly.

## Задание 6: Чемпион малвари.

### Нашу сеть только что внесли в списки спам-ферм. Один из хостов сети получает множество команд от панели C&C, ретранслируя их внутри сети. В обычных условиях причин для такого активного взаимодействия внутри сети у данного хоста нет.

Определите IP такого хоста.

``` r
zad6 <- df %>%
  filter(str_detect(src, "^12\\.") | str_detect(src, "^13\\.") | str_detect(src, "^14\\.")) %>%
  filter(str_detect(dst, "^12\\.") | str_detect(dst, "^13\\.") | str_detect(dst, "^14\\.")) %>%
  group_by(src) %>%
  summarise(count = n()) %>%
  arrange(desc(count)) %>%
  slice_head(n = 1)
zad6 |> collect()
```

    # A tibble: 1 × 2
      src         count
      <chr>       <int>
    1 13.42.70.40 65109

filter(str_detect(src, “^12.”) | str_detect(src, “^13.”) |
str_detect(src, “^14.”)): filter(): Функция, используемая для отбора
строк данных. str_detect(src, “^12\\”): Функция из пакета stringr,
которая проверяет, соответствует ли строка src шаблону ^12\\. Шаблон
^12\\ означает, что строка должна начинаться с 12.. |: Логический
оператор “ИЛИ”, объединяющий несколько условий. Повторяется для ^13\\ и
^14\\, таким образом фильтруя строки, где src начинается с 12., 13., или
14.. filter(str_detect(dst, “^12.”) | str_detect(dst, “^13.”) |
str_detect(dst, “^14.”)): Аналогично предыдущему шагу, этот фильтр
отбирает строки, где dst начинается с 12., 13., или 14.. group_by(src):
Группировка данных по столбцу src. Это позволяет применять агрегирующие
функции к каждой группе отдельно. summarise(count = n()): summarise():
Создает новый фрейм данных с одним или несколькими сводными столбцами.
count = n(): Создает столбец count, который содержит количество строк в
каждой группе (где группа определяется по src). arrange(desc(count)):
Сортировка фрейма данных по столбцу count в порядке убывания (desc
означает “по убыванию”). slice_head(n = 1): Выбирает первую строку из
отсортированного фрейма данных. В данном контексте это строка с
максимальным значением count.

## Задание 7: Скрытая бот-сеть

### В нашем трафике есть еще одна бот-сеть, которая использует очень большой интервал подключения к панели управления. Хосты этой продвинутой бот-сети не входят в уже обнаруженную нами бот-сеть. Какой порт используется продвинутой бот-сетью для коммуникации?

``` r
zad7 <- dbGetQuery(connection, "
  SELECT port, timestamp
  FROM data
  WHERE timestamp = (
    SELECT max(timestamp)
    FROM data
  )
")
cat(zad7$port)
```

    83

dbGetQuery(connection, “SELECT port …”): Выполняет SQL-запрос к базе
данных через соединение connection. “SELECT port, timestamp FROM data
…”: Запрос выбирает столбцы port и timestamp из таблицы data. WHERE
timestamp = (SELECT max(timestamp) FROM data): Фильтрует строки,
оставляя только те, у которых значение столбца timestamp равно
максимальному значению timestamp в таблице data. SELECT max(timestamp)
FROM data: В подзапросе выбирается максимальное значение timestamp из
таблицы data.

### Задание 8: Внутренний сканнер

Одна из наших машин сканирует внутреннюю сеть.

Что это за система?

``` r
zad8 <- df %>%
  filter(str_detect(src, "^12\\.") | str_detect(src, "^13\\.") | str_detect(src, "^14\\.")) %>%
  filter(str_detect(dst, "^12\\.") | str_detect(dst, "^13\\.") | str_detect(dst, "^14\\.")) %>%
  group_by(src) %>%
  summarise(
    time = mean(timestamp, na.rm = TRUE),
    coun = n_distinct(dst)
  ) %>%
  arrange(time) %>%
  slice_head(n = 1)
zad8
```

    # A tibble: 1 × 3
      src            time  coun
      <chr>         <dbl> <int>
    1 12.35.59.94 1.58e12   200

filter(str_detect(src, “^12.”) | str_detect(src, “^13.”) |
str_detect(src, “^14.”)): filter(): Функция для отбора строк данных.
str_detect(src, “^12\\”): Проверяет, соответствует ли строка src шаблону
^12\\. Шаблон ^12\\ означает, что строка должна начинаться с 12.. |:
Логический оператор “ИЛИ”, объединяющий несколько условий. Повторяется
для ^13\\ и ^14\\, таким образом фильтруя строки, где src начинается с
12., 13., или 14.. filter(str_detect(dst, “^12.”) | str_detect(dst,
“^13.”) | str_detect(dst, “^14.”)): Аналогично предыдущему шагу, этот
фильтр отбирает строки, где dst начинается с 12., 13., или 14..
group_by(src): Группировка данных по столбцу src. Это позволяет
применять агрегирующие функции к каждой группе отдельно. summarise(time
= mean(timestamp, na.rm = TRUE), coun = n_distinct(dst)): summarise():
Создает новый фрейм данных с одним или несколькими сводными столбцами.
time = mean(timestamp, na.rm = TRUE): Вычисляет среднее значение столбца
timestamp для каждой группы, игнорируя NA значения. coun =
n_distinct(dst): Вычисляет количество уникальных значений в столбце dst
для каждой группы. arrange(time): Сортировка фрейма данных по столбцу
time в порядке возрастания. slice_head(n = 1): Выбирает первую строку из
отсортированного фрейма данных. В данном контексте это строка с
минимальным средним значением timestamp.

## Вывод

Я научился использовать ClickHouse для обработки и анализа больших
объемов данных вместе с языком программирования R. ClickHouse
обеспечивает быструю и эффективную обработку данных, что позволяет мне
проводить сложные вычисления и анализировать большие объемы данных с
использованием R. Это открывает новые возможности для работы с большими
наборами данных и позволяет мне принимать более информированные решения
на основе анализа данных.
