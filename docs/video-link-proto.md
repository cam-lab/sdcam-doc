# Протокол передачи видеопотока

## Общие сведения и назначение

При работе прибора возникает необходимость в передаче на удалённый хост информации различного рода. В первую очередь сюда относится непосредственно поток кадров изображения. Кроме этого, для гибкости и удобства работы, а также для возможности расширения вместе кадрами изображения требуется передавать некоторую метаинформацию.

Описываемый протокол предназначен для эффективной передачи всего вышеперечисленного с акцентом на эффективность и простоту реализации.

## Описание протокола

### Многоканальность и синхронизация

Протокол поддерживает передачу информации от разных каналов прибора с возможностью их произвольного чередования. Это позволяет по одному физическому интерфейсу передавать несколько каналов с видеоинформацией одновременно в масштабе реального времени&nbsp;– для этого необходимо, чтобы хватало общей пропускной способности интерфейса.

Протокол обеспечивает передачу видеоинформации с разрядностью пикселов до 14 бит. Слова данных в протоколе являются 16-разрядными. Два старших бита используются в качестве признаков (тегов) канала и синхронизации кадра. Поддерживаются следующие теги:

| Name | Description |
|------|-------------|
| `CFT`  | Channel Frame Tag. Признак переключения канала. После получения слова с этим тегом и <br>до получения следующего слова с этим тегом все данные относятся к указанному каналу |
| `VST`  | Video Sync Tag. Признак синхронизации кадра или его элемента. Подробное описание приведено <br> ниже. |

Тег считается активным, если значение бита равно лог.&nbsp;1.

!!! warning "**ВАЖНО**"

    **Тег переключения канала `CFT` допускается вставлять в поток только непосредственно перед тегом** `VST`. Это обусловлено соображениями производительности&nbsp;– чтобы не вынуждать приёмник проверять каждое слово потока на `CFT`.

    Проверка каждого слова на тег `CFT` выполняется при синхронизации, когда в потоке ищутся теги синхронизации канала и кадра, а при приёме **синхронизированного** видеопотока проверка выполняется только на границах кадров и строк. Таким образом, переключение канала возможно только на границе кадра или строки.

Переключение каналов производится при передаче слова c `CFT`, в котором указан номер канала.

Если `CFT` активен, то остальные биты слова интерпретируются как показано ниже[^1]:
<p></p>

wavedrom (
    {reg: [
        {"bits": 5, "name": "Channel Number: n",  "type": 7, "attr": ""},
        {"bits": 10, "name": "Reserved: x",       "type": 1, "attr": ""},
        {"bits": 1, "name": "CFT: 1",             "type": 5, "attr": ""},
    ], config: {lanes: 1, bits: 16, vflip: false, hflip: true}
  } 
)

[^1]: Здесь и далее в описаниях полей используется нотация: `Name:Value`

!!! info "**ЗАМЕЧАНИЕ**"

    Если используется всего один канал, то `CFT` можно игнорировать и на передаче, и при приёме. Но приёмник и передатчик должны быть в этом согласованы.

    Если нет жёстких требований по экономии ресурсов и производительности, то рекомендуется выполнять реализацию по полной спецификации протокола&nbsp;– это облегчит возможное расширение функциональности (добавление каналов передачи) и уменьшит появление ошибок при изменении реализации на одной из сторон.


### Формат передачи данных по видеоканалу

#### Блоки синхронизации

Блоки синхронизации служат для организации и контроля правильного приёма видеопотока.

Если `CFT` неактивен, то следующим управляющим тегом является `VST`, который индицирует блок синхронизации &nbsp;– MDB (Meta Data Block). MDB содержит обязательную и необязательную (опциональную) части.

Поддерживается два типа блоков синхронизации:

| Name | Description |
|------|-------------|
| `FS` | Frame Sync MDB. Блок метаданных синхронизации кадра. Содержит обязательную  часть&nbsp;– различную<br> служебную информацию о кадре: циклический номер кадра, временной штамп, размерность кадра<br> и разрядность пиксела. Кроме этого MDB может содержать дополнительные элементы, относящиеся к <br> блокам расширения (см. ниже). 
| `LS` | Line Sync MDB. Блок метаданных синхронизации строки. Содержит номер строки. Остальная <br>информация является опциональной. |

Тип блока задаётся полем `FTT`&nbsp;– Frame Tag Type, а длина полем `MDBS` (MDB size):
<p></p>

wavedrom (
    {reg: [
        {"bits": 13, "name": "MDBS: 0..N",  "type": 7, "attr": ""},
        {"bits": 1,  "name": "FTT:0/1", "type": 2, "attr": ""},
        {"bits": 1,  "name": "VST: 1",   "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",   "type": 5, "attr": ""},
    ], config: {lanes: 1, bits: 16, vflip: false, hflip: true}
  } 
)

`MDBS` указывает длину блока, не включая слова с тегом, т.е. может иметь величину от \(0\) до \(2^{13} = 8192\) слова.

#### Блоки расширений

Блоки расширений являются опциональными и присутствуют тогда, когда необходимо передавать дополнительную информацию об элементах видеопотока&nbsp;– кадрах и строках изображения. Например, передавать с каждым кадром дополнительную информацию о параметрах аппаратуры, при которых был получен данный кадр: время экспозиции, коэффициенты усиления элементов видеотракта и т.п. 

!!! warning "**ВНИМАНИЕ**"

    Протокол никак не регламентирует содержимое таких блоков, он только учитывает их длину в MDB. Формирование, разбор MDB и правильность информации в них лежат на протоколах более высоких уровней. Описываемый протокол задаёт только размер MDB, чтобы передающая и приёмная стороны могли обеспечивать передачу и приём блоков расширений, не имея сведений об их содержимом.


### Передача кадра

В общем случае передача начинается с переключения канала при помощи использования тега `CFT`. За ним следует кадровый блок метаинформации, обозначающий начало передачи кадра и содержащий служебную необходимою информацию о кадре, которая может использоваться на приёмной стороне. К обязательной части MDB относится номер кадра, временной штамп, размерность кадра и разрядность пиксела. 

!!! info "**ЗАМЕЧАНИЕ**"

    Размерность кадра и разрядность пиксела позволяют избежать жёсткого кодирования этих параметров на приёмной стороне и гибко подстраиваться под произвольные размеры кадра "на лету", что является актуальным при работе с разными приборами и в многоканальном режиме.

    Номер кадра и временная отметка полезны для контроля видеопотока&nbsp;– они позволяют засекать пропуски кадров, "джиттер" их поступления, оценивать задержки реакции на управляющие воздействия и т.д.

Ниже приведёт фрагмент, начинающийся с переключения канала и содержащий кадровый MDB. К обязательной части кадрового MDB относится циклический номер кадра&nbsp;– 32-разрядное беззнаковое целое и временной штамп&nbsp;– 64-разрядное беззнаковое целое. 

В каждом передаваемом кадре циклический номер кадра увеличивается на 1&nbsp;– используется обычный двоичный счётчик. При достижении максимального значения происходит переполнение и счёт начинается с 0.

Протокол не накладывает никаких ограничений на алгоритм формирования временного штампа. Самая простая реализация&nbsp;– помещать туда значение счётчика тактов внутренней тактовой частоты камеры.

#### Кадровый MDB

wavedrom (
    {reg: [
        {"bits": 5, "name": "Channel Number: n",   "type": 7, "attr": ""},
        {"bits": 10,  "name": "Reserved: x", "type": 1, "attr": ""},
        {"bits": 1,  "name": "CFT: 1",  "type": 5, "attr": ""},

        {"bits": 13, "name": "MDBS: 15", "type": 7, "attr": ""},
        {"bits": 1,  "name": "FTT: 0",  "type": 2, "attr": ""},
        {"bits": 1,  "name": "VST: 1",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Frame Number[ 7:0]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Frame Number[15:8]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Frame Number[23:16]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Frame Number[31:24]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Timestamp[ 7:0]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Timestamp[15:8]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Timestamp[23:16]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Timestamp[31:24]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Timestamp[39:32]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Timestamp[47:40]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Timestamp[55:48]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 8, "name": "Timestamp[63:56]", "type": 7, "attr": ""},
        {"bits": 6, "name": "x",   "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 14, "name": "Frame Width", "type": 7, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 14, "name": "Frame Height", "type": 7, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 14, "name": "Pixel Width", "type": 7, "attr": ""},
        {"bits": 1,  "name": "VST: 0",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

    ], config: {lanes: 17, bits: 272, vflip: false, hflip: true}
  } 
)

#### Строчный MDB и начало потока данных (пикселов)

Непосредственно за кадровым блоком следует строчный, индицирующий начало передачи первой строки кадра:
<p></p>

wavedrom (
    {reg: [
        {"bits": 13, "name": "MDBS: 1", "type": 7, "attr": ""},
        {"bits": 1,  "name": "FTT: 1",  "type": 2, "attr": ""},
        {"bits": 1,  "name": "VST: 1",  "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",  "type": 5, "attr": ""},

        {"bits": 12, "name": "Line Number", "type": 7, "attr": ""},
        {"bits": 2,  "name": "Reserved",    "type": 1, "attr": ""},
        {"bits": 1,  "name": "VST: 0",      "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",      "type": 5, "attr": ""},

        {"bits": 14, "name": "Pixel 0", "type": 7, "attr": ""},
        {"bits": 1,  "name": "VST: 0",      "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",      "type": 5, "attr": ""},

        {"bits": 14, "name": "Pixel 1", "type": 7, "attr": ""},
        {"bits": 1,  "name": "VST: 0",      "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",      "type": 5, "attr": ""},

        {"bits": 14, "name": "Pixel 2", "type": 7, "attr": ""},
        {"bits": 1,  "name": "VST: 0",      "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",      "type": 5, "attr": ""},

        {"bits": 14, "name": "Pixel 3", "type": 7, "attr": ""},
        {"bits": 1,  "name": "VST: 0",      "type": 6, "attr": ""},
        {"bits": 1,  "name": "CFT: 0",      "type": 5, "attr": ""},

    ], config: {lanes: 6, bits: 96, vflip: false, hflip: true}
  } 
)

После того, как все пикселы строки переданы, следует следующий строчный MDB, указывающий номер следующий строки. Это продолжается до тех пор, пока не будут переданы все строки кадра. После чего начинается передача следующего кадра&nbsp;– передаётся кадровый MDB со своими полями, за ним строчный MDB первой строки этого кадра и т.д.:

wavedrom (
{signal: [
  {node: '.F..............G...H'},
  {node: '....A.....B.....M'},
  {name: 'CFT', wave: '10.......|.....|...|'},
  {name: 'VST', wave: 'x10.10...|10...|10.|', },
  {name: 'FTT', wave: 'x0..10...|10...|...|'},
  {name: 'Data', wave: '634.34.5.|34.5.|34.|', 
          data: ['CN', 'MS', 'FMDB', 'MS', 'LMDB', 'Pix Data',
                                     'MS', 'LMDB', 'Pix Data', 
                 'MS', 'FMDB', ],node : '.C..L.....D.....E' },
  {},
  
], 
   edge: [
     'A<->B Line',
     'B<->M LIne',
     'L-A',
     'B-D',
     'G-E',
     'C-F',
     'F<->G Frame',
     'G<-|>H Frame'
   ],
   config: { hscale: 1 },
  }
)

где:

    CN:       Channel Number
    MS:       MDB Size
    FMDB:     Frame MDB
    LMDB:     Line MDB
    Pix Data: Row pixel data stream

### Приём видеопотока

При приёме алгоритм следующий:

  1. сканируется весь поток на предмет поиска тегов `CFT` и `VST`. При нахождении `VST` проверяется
  тип MDB, если это строка, то сканирование продолжается, если это кадровый MDB, то это означает
  начало синхронизации; 
  1. производится разбор кадрового MDB: извлекаются номер кадра и временной штамп, фиксируются параметры кадра&nbsp;– его размерность и разрядность пиксела;
  1. ожидается приём строчного MDB. Если приходит что-то другое, то кадр считается испорченным, и приём снова переходит к п. 1;
  1. производится разбор строчного MDB&nbsp;– фиксируется номер строки. Строки внутри при передаче не обязаны следовать строго одна за другой[^2], но все они должны быть без ошибок приняты до прихода следующего кадра. В противном случае кадр считается испорченным, все принятые данные отбрасываются и приём начинается с п.1;
  1. принимаются данные строки (значения пикселов) и помещаются в память по адресу, соответствующему номеру строки. Если принята последняя строка в кадре (контролировать можно, например, по количеству принятых строк), то по окончании осуществляется переход к следующему пункту, иначе&nbsp;– к п.3;
  1. Принятый кадр передаётся потребителю, а приёмник переходит к ожиданию следующего кадра, начиная с п.1.

  Если осуществляется приём видеопотока от нескольких каналов, то приёмник должен иметь соответствующее количество экземпляров логики приёма&nbsp;– контекстов, которые переключаются при приёме `CFT`.

  [^2]: Существуют форматы кадров, которые формируют последовательность строк не строго по порядку&nbsp;– например, при чересстрочной развёртке.