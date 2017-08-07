This README is created only for russian speaking people. If you want to use this api to create a mindtrainer, but you don't speak russian, you could send an email (zpix-dev@list.ru)
# О создании тренажера для beta.mindtrainer.ru
### Установка api
Обертка использует фреймворк Sinatra, написанный на языке ruby.
#### Windows
1. Установите [Ruby](https://rubyinstaller.org/).
2. Скачайте [репозиторий](https://github.com/zpix1/MindTrainerAPI/archive/master.zip) и распакуйте его в какую-нибудь папку, где вы будете разрабатывать тренажер.
3. Откройте командную строку Windows в папке с репозиторием.
4. Установите bundler командой
    `gem install bundler`
5. Установите sinatra командой
    `bundle install`

### Общее
Новая версия MindTrainer'a поддерживает тренажеры на разных языках программирования
Каждый  тренажер состоит из 3 частей:

1. Ядро - консольная программа, обеспечивающая проверку и генерацию задачи.
2. Интерфейс - визуальная часть тренажера, набор из CSS, JavaScript и HTML, который показывает задачу и предоставляет удобные инструменты для её решения.
3. Ник  и шаблон настроек 
### Техническая часть
#### Ядро
Ядро может быть написано на любом языке программирования, компилятор или интерпретатор которого есть на сервере или который можно туда поставить, но вот список рекомендуемых языков: C, C++, Python, Ruby, PHP, Perl

Ядро должно отвечать консольные вызовы такого типа:

1. `generate '<json config>' '<seed>'`, где `<json config>` это конфигурация в формате JSON (ассоциативный массив вида [название параметра => значение], названия берутся из шаблона настроек), a `<seed>` это целое число, которое необходимо использовать для инициализации генератора случайных чисел в ядре. Ядро должно выдать закодированный в формат JSON ассоциативный массив, имеющий 2 ключа - `question` и `answer`, задача и ответ на нее, если такой необходим для работы `check`, соответственно.
2. `check '<user answer>' '<answer>'`, где `<user answer>` ответ пользователя, `<answer>` - ответ на задачу из `generate`. В ответ программа должна выдать либо "true", либо "false" в зависимости от правильности ответа пользователя.

#### Интерфейс
Интерфейс в MindTrainer'e может быть очень гибким, поэтому к нему не предъявляется особых требований. Главное, должны быть поля для входных данных и для выходных.
Лучше будет видно в примере.

#### Ник и шаблон настроек
У каждого тренажера должен быть свой ник, позволяющий системе однозначно его определять.
Например, у анаграмм - `anagram`, у разрезаний - `razrez`.

Также, тренажер должен иметь шаблон настроек. Это массив, закодированный в формате JSON который дает системе информацию о параметрах тренажера.

Массив состоит из словарей (ассоциативных массивов) на каждый параметр, имеющих следующие ключи:

* `nick` - название параметра (строка из латинских букв, без пробелов (как название переменной)), именно оно пойдет в конфигурацию для `generate`.
* `description` - описание параметра (на русском, нужно для панели настройки тренажера у администратора)
* `type` - строка, содержащая тип параметра. Сейчас поддерживаются только `int` и `int_table` - целое число и таблица целых чисел соотвественно. Для `int` должны быть ключи `min` и `max`, минимальное и максимальное возможное значение параметра, а для `int_table` помимо `min` и `max` должны быть ключи `height` и `width` - высота и ширина таблицы.


### Пример
Напишем тренажер сложения цифр

Определимся с ником. Пусть будет `digsum`.

Определимся с шаблоном настроек. Будет 1 параметр - максимальное значение цифры. 
```JSON
[
    {
        "nick":"maxval",
        "type":"int",
        "min":0,
        "max":9,
        "description":"Максимальное значение цифры"
    }
]
```

Напишем ядро на языке Python
``` Python
import sys # Для доступа к аргументам командной строки
import json # Для парсинга JSON
from random import randint,seed # Для генерации цифр

mode = sys.argv[1] # Получем режим
if mode == 'generate': # Если надо сгенерировать задачу
    config = json.loads(sys.argv[2]) # Получаем конфигурацию
    s = int(sys.argv[3]) # Получем зерно для генератора случайных чисел
    seed(s) # Используем это зерно
    maxval = config["maxval"]
    a = randint(0, maxval) # Генерируем цифры для сложения
    b = randint(0, maxval)
    question = str(a) + " + " + str(b) + " = ?" # Вопрос
    answer = str(a+b) # Ответ
    print(json.dumps({'question':question,'answer':answer}))
elif mode == 'check': # Если надо проверить
    user_answer = int(sys.argv[2]) # Считываем ответ пользователя и настоящий ответ
    answer = int(sys.argv[3])
    if user_answer == answer: # Если правильно, то пишем true, иначе false
        print('true')
    else:
        print('false')
```
Напишем интерфейс, будет 2 файла - show и check
Тут используется технология erb, знать ее не обязательно, просто игнорируйте текст внутри `<% %>`

show.html.erb
``` HTML
<%= form_tag(controller: 'trainers', action: 'check') do%>
    <%= submit_tag('Ответить',class:"btn") %>
        <input type="reset" class="btn" value="Очистить"> <!-- Кнопка для очистки -->
        <br>
    <br>
    <font size="20"><%= @task[:question] %></font> <!-- Здесь показывается вопрос -->
    <%= label_tag(:user_answer, 'Ваш ответ: ') %> 
    <input size="10" type="number" class="form-control anagramtext" id="user_answer" name="user_answer"> <!-- Поле для ответа пользователя -->
<% end %>
```

Этот файл покажется только если ответ пользователя неверный

check.html.erb
``` HTML
Вы ошиблись! Правильный ответ: <%= @task[:answer] %>
```

Все! Осталось добавить эти файлы в систему, это займет совсем немного времени.