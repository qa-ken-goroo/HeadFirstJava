# HeadFirstJava## Сервис тестирования spr-reactive
С помощью этого проекта можно проверить корректность работы стратегий spr-reactive

Проект состоит из двух основных частей:
1. код, который запускает выполнение тестов
2. тестовые данные - набор кейсов

Кейс - набор запросов и предполагаемых эталонных ответов на них. Данные кейсу находятся в папке [case name],
внутри которой лежат:
* папка request -- запросы
* папка response -- ожидаемые ответы
* файл first-stage.txt - в нем указывается код первого шага (FIRST_ROUTE, AFS_CHECK и пр.)
* файл headers.properties - тут указывается \_\_TypeId__ и CALL_CD_EXT для конкретного кейса. Если в папке кейса нет этого файла, то будет выполнена попытка найти такой файл в родительском каталоге.

Обработка кейса проходит следующим образом:
1. из файла first-stage.txt берется код первого шага, например FIRST_ROUTE
2. в файле FIRST_ROUTE.json [loanRequestId] заменяется на сгенерированный случайный UUID. В рамкак обработки кейса этот loanRequestId фиксируется для идентификации получаемых ответов.
3. производится вычисление и замена подстановочных полей в файлах request и response
4. в очередь spr-reactive-in-test отправляется сообщение с полученным текстом и заголовками из файла headers.properties 
5. из очереди spr-reactive-out-test пулятся сообщения с loanRequestId, соответсвующим кейсу
6. тело полученного сообщения сравнивается с эталонным/ожидаемым из папки response. Ниже более подробно про этот пункт

### Подстановочные поля
Подстановочные поля используются для кастомизации запросов и ответов. 
Подстановочные поля можно использовать как в файлах в файлах запросов, так и в файлах ответов (как в простых значениях, так и в аргументах блоков **matchType**)
Допустимо использовать следующие подстановочные поля:
1. [customerRequestExtId] -- случайное значение, единое в рамках одного тест-кейса
2. [loanRequestId] -- случайное значение, единое в рамках одного тест-кейса
3. [currentDttm]
4. [pastDate(число)(код единицы измерения)] - дает дату в прошлом на указанное количество дней/месяцев/лет в прошлом от текущей даты. 
Например, [pastDate0d] -- сегодняшняя дата, [pastDate1d] -- вчерашняя, [pastDate1m] -- месяц назад.
5. [today(+/-)(число)(код единицы измерения)] - дает дату в прошлом или будущем на указанное количество дней/месяцев/лет от текущей даты.
Например, [today-0d] -- сегодняшняя дата, [today-1d] -- вчерашняя, [today+1m] -- месяц вперёд.

Сравнение с эталонным json производится не для всего json, а только для специально отмеченных элементов.
Например, для того чтобы проверить значение элемента 
```
"testsElement": "testValue"
```

необходимо заменить в соответствующем файле этот элемент на
```
"testsElement": {
  "matchType": "equals",
  "value": "testValue"
}
```
В данном случае проверка будет успешной только если значение **testElement** равно (equals) значению **value**

Кроме equals в поле matchType можно указать и другие операторы из списка ниже (не чувствительны к регистру): 
* EQUALS 
* NOT_EQUALS 
* BETWEEN -- для чисел и дат
* GREATER -- для чисел и дат
* NOT_GREATER -- для чисел и дат
* DIST_DAY -- для дат
* LESS -- для чисел и дат
* NOT_LESS -- для чисел и дат
* IN_LIST 
* OF_TYPE - допустимые значения value:
  * ARRAY,
  * BOOLEAN,
  * NUMBER,
  * OBJECT,
  * STRING
* COUNT
* EQUALS_AS_SET -- сравнение для массивов без учета порядка элементов и их повторов
* EQUALS_AS_SET_ -- то же, что EQUALS_AS_SET, но в элементах массива рассматриваются только указанные поля
* CONTAINS_ONLY -- проверяет, что все элементы массива одинаковые и равны указанному элементу
* CONTAINS_ONLY_ -- проверяет, что все элементы массива равны указанному, но только в объеме полей присутствующих в элементе (по принципу **EQUALS_**)
* CONTAINS -- для строк и массивов, проверяет, что объект содержит все перечисленные элементы
* CONTAINS_ -- то же, что CONTAINS, но в элементах первого уровня рассматриваются только указанные поля.
* NOT_CONTAINS
* NOT_CONTAINS_
* EXISTS - для этого типа значение value указывать не требуется
* NOT_EXISTS - для этого типа значение value указывать не требуется
* NOT_EXISTS_ - проверяет, что массив не содержит определённый элемент или вообще не существует
* MATCH_LIST - позволяет осуществить несколько проверок в одном объекте


## Примеры применения

<details>
<summary>EQUALS и NOT_EQUALS</summary>
Если нужно, чтобы element был равен "some value", то в эталонном файле должно быть указано следующее

```
"element": {
  "matchType": "equals",
  "value":"some value"
}
```

"element":"some value"  - соответствует условию
"element":"other value"  - не соответствует условию

Пример: проверяем, что decisionCdExt = "Proceed"

```
{
  "decisionResult": {
    "decisionCdExt": {
      "matchType": "equals",
      "value": "Proceed"
    }
  }
}
```
</details>


<details>
<summary>EQUALS_ - "Мягкое" равенство: сравнение будет производиться только по значениям, указанным в объекте <b>value</b></summary>
```
"element": {
  "matchType": "equals",
  "value": {
    "id": "10"
  }
}
```

"element": {
  "id": "10",
  "val": "hello"
} -- соответствует условию -- наличие дополнительного элемента **val** не влияет на проверку

"element": {
  "id": "10"
} -- тоже соответствует

"element": {
  "id": "12",
  "val": "hello"
} -- не соответствует
</details>


<details>
<summary>BETWEEN и BETWEEN_
если нужно, чтобы element был строго между 5 и 15, то в эталонном файле должно быть указано следующее</summary>
```
"element": {
"matchType": "between",
"value": [5, 15]
}
```
> "element": 6  - соответствует условию

> "element": 4  - не соответствует условию
</details>


<details>
<summary>GREATER, NOT_GREATER, LESS, NOT_LESS - если нужно, чтобы element был строго больше 16, то в эталонном файле должно быть указано следующее</summary>
```
"element": {
  "matchType": "greater",
  "value": 16
}
```
"element": 17  - соответствует условию
"element": 16  - не соответствует условию
</details>


<details>
<summary>DIST_DAY - К дате из массива value прибавляется число из этого же массива -- полученная дата должна совпадать с actual значением.</summary>
Пример 1: проверяем, что дата = 2022-10-22
```
"element": {
  "matchType": "dist_day",
  "value": ["2022-10-12", 10]
}
```
Пример 2, с подстановочным значением: проверяем, что дата **decisionGenerationDt** = сегодняшний день, а **decisionExpirationDt** = сегодняшний день + 30 дней (Для сегодняшней даты 2024-01-30 ожидаемое значение будет 2024-02-29)
```
{
  "decisionResult": {
    "decisionExpirationDt": {
      "matchType": "dist_day",
      "value": ["[today+0d]", 30]
    },
    "decisionGenDt": {
      "matchType": "equals",
      "value": "[today+0d]"
    }
  }
}
```
</details>


<details>
<summary>IN_LIST - данный оператор используется для проверки того, что элемент может принимать одно из допустимых значений.</summary>
Допустим, что element должен принимать значения только 15, 20, 25
```
"element": {
  "matchType": "in_list",
  "value": [15, 20, 25]
}
```
> "element": 15 - соответствует условию

> "element": 16 - не соответствует
</details>


<details>
<summary>OF_TYPE - можно проверить, что значение элемента имеет определенный тип, например</summary>
```
"element": {
  "matchType": "of_type",
  "value": "ARRAY"
}
```
> "element": [15, 25, 20] - соответствует условию

> "element": "[15, 25, 20]" - не соответствует условию

> "element": [{"val": "x"}] - соответствует условию
</details>


<details>
<summary>COUNT - проверяет количество элементов в массиве</summary>
```
"element": {
  "matchType": "count",
  "value": 3
}
```
> "element": [15, 25, 20] - соответствует условию

> "element": [15, 25] - не соответствует

> "element": [15, 25, 2, 4] - и это не соответствует
</details>


<details>
<summary>EQUALS_AS_SET - Этот оператор нужно использовать, когда необходимо, чтобы в массиве содержались только определенные элементы и при этом их порядок не важен</summary>
```
"element": {
  "matchType": "equals_as_set",
  "value": [15, 20, 25]
}
```
> "element": [15, 25, 20] - соответствует условию

> "element": [15, 20, 25] - тоже соответствует

> "element": [5, 20, 25] - а вот этот нет
</details>


<details>
<summary>EQUALS_AS_SET_ - "Мягкий" оператор сравнения массивов используется, если массив содержит объекты и в каждом объекте допустимы дополнительные поля, кроме указанных</summary>
Пример: проверяем, что массив "externalSystems" содержит только два элемента и ничего больше.

```
{
  "controlRoute": {
    "externalSystems": {
      "matchType": "EQUALS_AS_SET_",
      "value": [
        {
          "extSysCdExt": "PTI"
        },
        {
          "extSysCdExt": "AFS"
        }
      ]
    }
  }
}
```

Такой объект будет соответствовать условию:
```
  "controlRoute": {
    "stageCdExt": "PIM_MODELS_CHECK",
    "nextStageCdExt": "AFS_SECOND_CHECK",
    "prevStageCdExt": "AFS_CHECK",
    "externalSystems": [
      {
        "extSysCdExt": "AFS",
        "customerFormId": "3e8ee0bf-bcf0-4a38-895a-512bd35a70e6"
      },
      {
        "customerExtId": "1d94e38f-6652-4962-9082-75ab3d5ee780",
        "extSysCdExt": "PTI",
        "customerFormId": "3e8ee0bf-bcf0-4a38-895a-512bd35a70e6"
      }
    ]
  }
}
```
</details>


<details>
<summary>CONTAINS_ONLY_ - Проверяет, что все элементы массива равны указанному, но только в объеме полей, присутствующих в элементе</summary>
Пример: проверяем, что в массиве "offerMatrix" у всех элементов значение "calRiskGradeCdExt" равно "10460002"
```
{
  "decisionResult": {
    "offerMatrix": {
      "matchType": "contains_only_",
      "value": {
          "calRiskGradeCdExt": "10460002"
      }
    }
  }
}
```
</details>



<details>
<summary>CONTAINS - Если значение элемента - массив и нужно проверить, что в нем содержится одно или несколько значений, используй contains</summary>
* Пример 1, с массивом:
```
"element": {
  "matchType": "contains",
  "value": [15, 20]
}
```
> "element": [15, 25, 20] - соответствует условию

> "element": [15, 5, 25] - не соответствует условию

Пример 2, со строкой:
```
{
  "decisionResult": {
    "participants": [
      {
        "pilotString": {
          "matchType": "contains",
          "value": "PACC_VC"
        }
      }
    ]
  }
}
```
</details>




<details>
<summary>CONTAINS_ - Для массива примитивных значений работает как простой CONTAINS.</summary>
Если в массиве присутствуют объекты, то сравнение будет происходить только по полям эталонного объекта.

Пример 1: проверяем, что в offerMatrix присутствует элемент с типом Confirmed и решением Accept.
```
{
  "decisionResult": {
    "offerMatrix": {
      "matchType": "contains_",
      "value": {
        "offerDecisionCdExt": "Accept",
        "incomeConfTypeCdExt": "Confirmed"
      }
    }
   }
}
```

Пример 2:
```
"element": {
  "matchType": "contains_",
  "value": [
    {"v": 1},
    {"v": 2}
  ]
}
```
> "element": [
  {
    "v": 1,
    "h": 2,
  },
  {
    "v": 2,
    "h": 4
  }
] - соответствует условию

Т.е. несмотря на наличие элемента "h" в объектах массива сравнение будет вестись только по значению элемента "v".
Поменять местами эталонное значение и сравниваемое не получится, т.е. для такого сравнения
```
"element": {
  "matchType": "contains_",
  "value": [
    {
      "v": 1,
      "h": 2,
    },
    {
      "v": 2,
      "h": 4
    }
  ]
}
```
> "element": [
  {"v": 1},
  {"v": 2}
] - такой элемент уже не будет соответствовать условию

Если внутри элементов массива есть объекты и эти объекты выбраны для сравнения, то они должны быть строго равны эталонному объекту по всем полям.

```
"element": {
  "matchType": "contains_",
  "value": [
    {
      "v": 1,
      "val": {
        "v1: "a"
      }
    },
    {
      "v": 2
    }
  ]
}
```
```
"element": [
  {
    "v": 1,
    "val": {
      "v1": "a",
      "v2": "b"
    },
    "h": 2,
  },
  {
    "v": 2,
    "h": 4
  }
] - не соответствует условию, т.к. есть лишнее поле "v2": "b" в объекте "val"
``` 
</details>


<details>
<summary>NOT_CONTAINS_ - для массива примитивных значений работает как простой NOT_CONTAINS.</summary>
Если в массиве присутствуют объекты, то сравнение будет происходить только по полям эталонного объекта.

Пример: проверяем, что в массиве "offerMatrix" нет объектов, одновременно содержащих "liabilityTypeCdExt": "07" и "offerRouteCdExt": "11440003".
Предполагается, что сам массив "offerMatrix" существует, иначе объект не будет соответствовать условию.
```
{
  "decisionResult": {
    "offerMatrix": {
      "matchType": "not_contains_",
      "value": {
        "liabilityTypeCdExt": "07",
        "offerRouteCdExt": "11440003"
      }
    }
  }
}
```
</details>


<details>
<summary>NOT_EXISTS - проверяет, что объект не существует. Value указывать не нужно.</summary>
Пример: проверка на отсутствие массива stopFactors, т.е. любых стоп-факторов
```
{
  "decisionResult": {
    "stopFactors": {
      "matchType": "not_exists"
    }
  }
}
```
</details>


<details>
<summary>NOT_EXISTS_ - проверяет, что массив не содержит определённый элемент или вообще не существует</summary>
Пример 1:
```
{
  "array1": {
    "matchType": "not_exists_",
    "value": {
        "v": 1,
     }
  },
  "array2": {
    "matchType": "not_exists_",
    "value": {
        "v": "A",
     }
  }
}
```
> {
  "array1": [
    {"v": 2}
  ]
> } - такой объект соответствует обоим условиям: в массиве array1 нет элемента v=1, массив array2 не существует

> {
"array2": [
{"v": "A"}, {"v": "B"}
]
> } - не соответствует условию 2

Пример 2: проверка на отсутствие стоп-фактора RMT004. Массив stopFactors не существует либо содержит коды, отличные от RMT004.
```
{
  "decisionResult": {
    "stopFactors": {
      "matchType": "not_exists_",
      "value" "RMT004
    }
  }
}
```
</details>

<details>
<summary>MATCH_LIST - позволяет осуществлять несколько проверок для одного объекта.</summary>
Пример: проверяем, что в "pilotString" не содержится подстрока ";NEWCH" и одновременно сожержится подстрока ";RD_NEWCH10000"

```
{
  "decisionResult": {
    "participants": [
      {
        "pilotString": {
          "matchType": "match_list",
          "value": [
            {
              "matchType": "not_contains",
              "value": ";NEWCH"
            },
            {
              "matchType": "contains",
              "value": ";RD_NEWCH10000"
            }
          ]
        }
      }
    ]
  }
}
```
</details>


## Особенности работы с массивами
Рассмотрим следующий json объект
```
{
  "x": [
    {"z": 1},
    {"z": 2}
  ]
}
```

для следующего сравнения результат будет неопределенным
```
{
  "x": [
    {
      "z": {
        "matchType": "equals",
        "value": 1
      }
    }
  ]
}
```
Проблема тут заключается в том, что при поиске элемента z будет получено 2 результата: 1 и 2.
Т.е. для алгоритма оба значения находятся "по одному адресу". Следует избегать таких ситуаций.

Хорошая новость в том, что таких ситуаций будет немного. Чаще всего элементы в массиве имеют уникальные поля - идентификаторы.
Для иллюстрации дополним пример выше
```
{
  "x": [
    {
      "z": 1,
      "y": 1
    },
    {
      "z": 2,
      "y": 2
    }
  ]
}
```

```
{
  "x": [
    {
      "z": {
        "matchType": "equals",
        "value": 1
      },
      "y": 2
    }
  ]
}
```
Тут уже неоднозначности не будет, т.к. элементы массива обросли дополнительным элементом "y", который и разделяет однородные элементы.

Эта же особенность может стать причиной необъяснимых (на первый взгляд) ошибок: когда значение элемента "y" может меняться в разных ответах. Следует очень аккуратно выбирать элементы для маршрутизации.


## Запуск
```
gradle test + _параметры_
```


## Параметры запуска
* _testFolders_ - через запятую перечисляются кейсы, которые нужно проверить. В папке resources/spr находятся наборы кейсов. Внутри наборов находятся кейсы (необходимо называть кейсы test1, test2 и т.д.).
Также наборы кейсов могут быть объединены в логические группы, тогда структура папок выглядит как resources/spr/\<groupFolder>/\<caseSet>/test1.
  * если необходимо запустить набор кейсов или несколько наборов кейсов, то в параметр _testFolders_ необходимо передать названия папок с наборами кейсов.
  Например, testFolders = "caseSet1, caseSet2"
  * если наборы нужных кейсов объединены в группу, то нужно указать путь, начиная с имени группы. 
  Например, testFolders = "groupFolder1/caseSet1, groupFolder1/caseSet3"
  * если необходимо запустить целую группу кейсов, то в параметр передаётся имя группы с символом "/" в конце.
  Например, testFolders = "groupFolder1/" 
  * если из набора кейсов необходимо выполнить только некоторые, то нужно в квадратных скобках перечислить номера нужных кейсов. Например demo[1, 3-6] -- тогда будут выполнены кейсы:
    * demo/test1
    * demo/test3
    * demo/test4
    * demo/test5
    * demo/test6
* _testMaxSecondsPerCase_ - максимальное количество секунд выделяемое на обработку одного кейса.
Тесты завершатся ошибкой, если их выполнение (в секундах) занимает больше testMaxSecondsPerCase * [количество тестов в прогоне]
* _testServers_ -- сервер кафки 
* _logReportType_ без параметра = полный репорт в консоль, с параметром short - сокращенный.


## Заголовки
Необходимые заголовки можно добавлять в файл headers.properties в рамках конкретного кейса, набора кейсов или всех кейсов в каталоге spr.

## Локальный запуск
Для локального запуска нужно прописать параметрт -DtestSevers=localhost

## Работа в teamcity
Основная платформа для выполнения данного тесте - teamcity. По умолчанию используется ветка ♂master♂ (в работе находится возможность выбора ветки при старте).
Запуск осуществляется вручную (опять же в работе старт по вебхукам и по расписанию). При старте необходимо указать параметр **Папки для поиска кейсов** 


## UI
Собрать:
```
./gradlew clean copyDependencies thinJar
```