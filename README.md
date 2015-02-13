﻿# http_mystem
HTTP сервер вокруг программы Mystem от Яндекса (https://tech.yandex.ru/mystem/). Этот сервер был написан в качестве практики при изучении языка GO.

##Как это запустить
1. Скопилировать программу: **go build http_mystem.go**
2. Скачать [mystem](https://tech.yandex.ru/mystem/)
3. Скопировать/переименовать файл **http_mystem.json.example** в **http_mystem.json** и установить необходимые наcтройки
4. Запустить программу: **./http_mystem** . Путь к файлу с настройками можно указать в качестве первого параметра, при запуске сервера, например: **./http_mystem /user/configs/http_mystem.json** . Если путь не указан, будет ипользован http_mystem.json из текущего каталога.

##Параметры конфигурационного файла
**host** - адрес интерфейса, на котором будет запущен сервер, например: **127.0.0.1**.

**port** - порт, на котором сервер будет принимать запросы, например: **8989**.

**reg_filter** - фильтрующее регулярное выражение, например: **^[-а-яёА-ЯЁ]*$**. 
Если слово не подходит под регулярное выражение, то оно не обрабатывается. Это сделано потому, что на "некорректные" слова, 
например - "тест1", mystem может не ответить и тогда весь "поток", работающий с этой запущенной версией программы, зависнет в ожидании ответа.

**mystem_path** - путь к исполняемому файлу mystem, например: **mystem/mystem.exe**.

**mystem_options** - опции с которыми будет запущена mystem, например:
```json
[
    "-n",
    "--format json"
]
```
Опция **--format** так же влияет формат ответа сервера, т.е. если задана опция **--format json**, то и сервер ответит в формате json.
- Если формат **text**, то результат - это строка где данные для каждого слова отделяются символом конца строки("\n").
- Если формат **json**, то результат - это массив с данными по каждому слову.
- Если формат **xml**, то результат - это xml документ, с элементам **&lt;w&gt;** на каждое слово.
- Опции **-e** игнорируется, используется кодировка - UTF-8.
- Опции **-d** игнорируется, т.к. сервер обрабатывает только слова, а не предложения/словосочетания. И возможно "зависание" потока.
- Опции **-с** игнорируется, по той же причине, почему игнорируется "-d". Так же, возможно, некорректное получение ответа от mystem.
- Опция **-n** будет добавлена автоматически, т.к. нужна для того, чтобы сервер мог получать ответы от mystem.

**mystem_workers** - сколько версий программы запустить для обработки запросов, например: **2**.

**mystem_answer_size** - размер ответа mystem в байтах, например: **10000**. Может быть больше чем размер ответа mystem, но не меньше, лучше ставить с запасом. 
Если будет задано меньшее количество байт, чем размер ответа mystem, то будет считан не весь результат работы mystem. 
Соответственно будет возвращен некорректный результат и дальнейшие ответы этого "потока" будут некорректными, так как в следующих ответах могут содержаться
куски результата предыдущего запроса. 

**channel_buffer** - размер буфера очереди слов для обработки, например: **10**. Т.е. сколько слов может быть добавлено в очередь без ожидания.

**max_word_length** - максимальная длина слова, например: **50**. Если слово длинее, то оно будет усечено до указанной длины.

**max_words** - максимальное количество слов для обработки в одном запросе, например: **100**. Если слов больше, сервер вернет http код 413.

##Как работать
POST или GET запросом шлем, в переменной **words[]**, слово для обработки.

##Пример
```
curl http://127.0.0.1:8989 --data-urlencode words[]=проверяли --data-urlencode words[]=тестировали
```

Сервер запущен со следующими настройками:
```json
{
    "host": "127.0.0.1",
    "port": 8989,
    "reg_filter": "^[-а-яёА-ЯЁ]*$",
    "mystem_path": "mystem/mystem",
    "mystem_options": [
        "-n",
        "--format text",
        "-w",
        "-i",
        "-g",
        "-s",
        "--generate-all",
        "--weight"
    ],
    "mystem_workers": 2,
    "mystem_answer_size": 10000,
    "channel_buffer": 10,
    "max_word_length": 50,
    "max_words": 100
}
```
Результат в формате text:
```
проверяли{проверять:1.00=V,пе=прош,мн,изъяв,несов}
тестировали{тестировать:1.00=V,несов=прош,мн,изъяв,пе}
```

Результат в формате json:
```json
[{
    "analysis": [{
        "lex":"проверять",
        "wt":1,
        "gr":"V,пе=прош,мн,изъяв,несов"
    }],
    "text":"проверяли"
},
{
    "analysis": [{
        "lex":"тестировать",
        "wt":1,
        "gr":"V,несов=прош,мн,изъяв,пе"
    }],
    "text":"тестировали"
}]
```
Результат в формате xml:
```xml
<?xml version="1.0" encoding="utf-8"?>
<html>
<body>
    <se>
        <w>
            проверяли<ana lex="проверять" wt="1" gr="V,пе=прош,мн,изъяв,несов" />
        </w>
        <w>
            тестировали<ana lex="тестировать" wt="1" gr="V,несов=прош,мн,изъяв,пе" />
        </w>
    </se>
</body>
</html>
```