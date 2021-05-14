# __Readme__

Это проект II семестра курса информатики МФТИ. На основе хэш-твблицы была реализована простейшая база данных типа "ключ-значение". Особенностью  является то, что данные хранятся в едином файле. Кроме того были проведены различные тесты при различных параметрах(способах решения коллизий, алгоритмов хэширования). Кратко опишем реализацию  базы данных: структура файла, принцип работы и основные высокоуровневые функции.
## Принцип работы
### __1.__ Структура файла базы данных  
Файл базы данных начинается с заголовка таблицы __DHeader__, он содержит в себе информацию о базе данных, включая также статистику __stat__. Статистика хранит в себе:
```cpp
typedef struct _Stat {
    uint64_t keys;      // Количество ключей
    uint16_t tables;    // Количество таблиц
    uint64_t capacity;  // Ёмкость базы данных
    uint64_t keysz;     // Размер ключа
    uint64_t valuesz;   // Размер значения
    uint16_t maxlen;    // Максимальная длина
    uint64_t nodes;     // Количество элементов (узлов)
    uint64_t collision_amount; // количество коллизий
} Stat;
```


За заголовком следует таблица __Table__,  которая тоже имеет заголовок __THeader__  и массив элементов __Node__. Структура элемента выглядит так:

```cpp
typedef struct _Node {
    int64_t keyoff;    // Смещение ключа (внутри файла)
    uint64_t keysz;    // Размер ключа
    int64_t valueoff;  // Смещение значения
    uint64_t valuesz;  // Размер значения
    int64_t next;      // Указатель на следующий элемент
} Node;
```


### __2.__  Высокоуровневые  функции

Одной из  главных структур, которые позволяют нам оперировать базой данных является курсор:
```cpp
typedef struct _Cursor {
    int fh;
    Stat* stat;     // Статистика базы данных
    THeader th;     // Текущая таблица
    off_t tableoff; // Смещение заголовка внутри файла
    uint32_t hash;  // Значение хэша
    int idx;        // Индекс текущего элемента в таблице
    Node node;      // Текущий элемент
    off_t nodeoff;  // Смещение текущего элеменьа
    Node chain;     // Текущий элемент цепочки
    off_t chainoff; // Смещение текущего элемента внутри файла
    Node prev;      // Предыдущий прочитанный элемент
    off_t prevoff;  // Смещение предыдущего прочитанного элемента
    int len;        // Текущая длина
}Cursor;
```
Его модификации зависят от способа решения коллизий, выше представлен вариант этой структуры, соответствующий решению коллизии методом цепочек.
Теперь можно перейти к высокоуровневым функциям вставки и удаления элемента:

- __Вставка элемента__

```cpp
ht_set(DB* db, const char* key, const char* value)
```
Это функция ассоциации, по ключу ```key``` мы присваиваем значение ```value```. Стоит отметить, что если элемент, соответствующий ключу найден мы обновляем его значение с помощью  функции
```cpp
_cur_set_value(Cursor* cur, const char* value)
```
Стоит отметить, что функция вставки работает по-разному на различных алгоритмах решения коллизий - подробнее это будет описано позже.

Также функция вставки в случае переполнения таблицы создает новую таблицу.

- __Удаление элемента__

```cpp
ht_del(DB* db, const char* key)
```

Функция по ключу ```key```  удаляет значение элемента.

### __2.__  Интерфейсные  функции
Теперь рассмотрим команды пользователя и функции, с которыми он взаимодействует напрямую:

___MASS___:  

Данная команда на вход получает 2 аргумента: начальный индекс ключа и количество элементов в хэш-таблице. Вся таблица заполняется парой ключ-значение ```key* <-> value*``` _(```*``` - соответствующий номер в таблице)_ с помощью функции ___SET___.

___SET___:

Функционал описан выше.

___GET___:

Данная команда получает на вход  ключ и выводит значение, оответствующее этому ключу.

___DEL___:

Данная команда получает на вход  ключ и удаляет значение, соответствующее этому ключу.

___STAT___:

Данная команда выводит статистику нашей хэш-таблицы в виде:
```
Keys: 10001
tables: 1
Avg. key size: 7.889111
Avg. value size: 9.889112
Total capacity: 141307
Avg. Fill factor: 0.070775
Max chain length: 2
Filled nodes: 9622
Node fill factor: 0.068093
Avg chain length: 1.039389
Collision amount: 379
```
## Тесты на производительность

![Профилирование](https://github.com/ksartamonov/data-base/blob/stat/pictures/testing.png)

Построим график зависимости времени вставки, от количества ключей в базе. Как видно, в приближении время вставки пропорционально количеству ключей.

![График 1](https://github.com/ksartamonov/data-base/blob/stat/pictures/testplot(set).png)

Построим графики зависимости времени поиска и удаления элемента по ключу от количества ключей.

![График 2](https://github.com/ksartamonov/data-base/blob/stat/pictures/testplot(get).png)

В приближении можно сказать, что обе операции выполняются за константное время, независимо
от количества ключей в таблице.

![График 3](https://github.com/ksartamonov/data-base/blob/stat/pictures/testplot(del).png)

## Анализ различных функций хэширования

В проекте было произведено анализ различных функций хеширования. Некоторые функции были взяты с открытых git-hab`ов, некоторые написаны с помощью алгоритмов для 32 бит.

Критерии, которые важны для некриптографических хеш-функций:

* Выходные значения должны иметь равномерное распределение (uniform distribution). Этот критерий важен для построения хеш-таблиц, чтобы скорость поиска была оптимальна.  

*  Каждый входной бит должен в 50% случаев влиять на каждый выходной бит (avalanche — лавинный критерий). Этот критерий важен для использования хеша в качестве уникального идентификатора 
документа. Также, иногда хеш может иметь равномерное распределение, но проваливать лавинный критерий — в этом случае будут ключи, для которых хеш не будет давать равномерное распределение.

* Между парами бит на выходе не должно быть корреляции. Это рассматривать не будем, т.к. обычно с этим критерием проблемы нет.

* Скорость. Все нижеперечисленные хеши на порядок быстрее, чем криптографические.

В работе были использованы 5 хеш-функций:

1) FNV32
2) CRC32
3) murmur2_32
4) murmur3_32
5) rot1333

Теперь немного подробнее о каждой хеш-функции:

* #### FNV-1a

2003 год. Скорость: 4.5 циклов на байт.
Очень простая функция.
1. Равномерное распределение: плохое
2. Лавинный критерий: плохо. 

* #### CRC32

1975 год, скорость: 1.2–0.13 цикла на байт.
1. Равномерное распределение: плохое распределение нижних бит. Используйте верхние биты для хеш-таблицы, когда можно использовать все протестированные биты — до 16 верхних бит (до 216 слотов хеш-таблицы).
2. Лавинный критерий: очень плохо. Лавинный эффект отсутствует. Сказывается назначение хеша: его выходные биты спроектированы так, чтобы обнаруживать ошибки в канале передачи данных. Также, у CRC есть важное свойство: можно разбить сообщение на части, посчитать хеш для каждой части, потом объединить сообщение, и посчитать хеш всего сообщения из хешей его частей.

* #### MurmurHash 2

2008 год.
Есть проблема с равномерностью распределения некоторых ключей.

* #### MurmurHash 3

2011 год, скорость: 1.87 циклов на байт.
1. Равномерное распределение: хорошее.
2. Лавинный критерий: отлично.

* #### Rot13

Это вариация шифра Цезаря, разработанного в Древнем Риме, быстрая хеш-функция (быстрее SRC32)
1. Равномерное распределение: среднее.
2. Лавинный критерий: плохой.

По теории "лучшей" хеш-функцией должна была оказаться - murmur3, а "наихудшей" - SRC32, в виду плохого распределения и лавинного эфекта. 







## Анализ различных способов разрешения коллизий

## Компиляция и запуск:

Чтобы скомпилировать и собрать программу в исполняемый файл, воспользуйтесь командой
```
make
```
в терминале. При этом программа будет скомпилирована без использования оптимизаций и дополнительных ключей. Запуск программы осуществляется при помощи команды
```
./hashdbtest <filename> <hashfunction>
```
Аргумент filename -- имя файла, содержащего базу данных. Если файла нет в текущей дериктории, он будет создан, если же в текущей дериктории уже был файл, содержащий данные, программа откроет его и продолжит работу уже с имеющимися данными.

Аргумент hashfunction -- название функции хэшироваия, которую будет использовать программа. Доступные хэш-функции:
* rot1333
* murmur2_32
* murmur3_32
* CRC32
* FNV32

Заметим так же, что если пользователь не укажет в аргументах командной строки название хэш-функции, сделает ошибку в названии или укажет хэш-функцию, недоступную для использования, то по умолчанию программа будет использовать rot1333.

В программе предусмотрен вариант компиляции с дополнительной оптимизацией -Ofast.
Для этого используйте в терминале команду
```
make Ofast
```
При этом, после завершения и сборки, в терминале появится детальная информация о времени компиляии

Для того, чтобы посмотреть время работы программы, неоюходимо запустить ее с параметром time:
```
time ./hashdbtest <filename> <hashfunction>
```

## Профилирование и тесты

Временами бывает нужно отпрофилировать производительность программы или потребление памяти в программе на C++ и C. В данном проекте так же доступна такая возможность.
Чтобы начать профилирование, воспользуйтесь функцией:
```
make profile
```
Инструкция Makefile выполнить компиляцию со всеми необходимыми ключами, после чего программа будет запущена. При этом в терминале будет отображаться детальная информация о времение работы программы и сообщения callgrind. После завершения работы программы в текущей дериктории будет сформирван файл callgrind.out.*,

Для просмотра результатов будем использовать программу KCachegrind, который умеет работать с форматом отчетов callgrind, для построения графиков используем gnuplot.

В программе предусмотрено два режима работы: пользовательский режим и режим эксперемента. В режиме эксперемента программа собирает данные о времени работы различных функций и записывает их в файл data.txt для последующего использования. Для того, чтобы включить режим экспереента перейтите в файл hashdbtest.c и раскоментируйте 18 строчку:
```
//#define STAT
```
```
#define STAT
```
После этого заново скомпилируйте программу (см. пункт Компиляция и запуск).

## bush-скрипты

Проект содержит 3 bush-скрипта для автоматизации проведения тестов. Для того, чтобы использовать bush-скрипт, его нужно сделать исполняемым, для этого используем команду
```
chmod +x script.sh
```
После этого файл .sh может быть запущен:
```
./script.sh
```
