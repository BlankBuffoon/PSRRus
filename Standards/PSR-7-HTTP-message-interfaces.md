# [PSR-7](https://www.php-fig.org/psr/psr-7/): HTTP message interfaces

Этот документ описывает общие интерфейсы для представления HTTP сообщений, как описано в [RFC 7230](https://tools.ietf.org/html/rfc7230) и [RFC 7231](https://tools.ietf.org/html/rfc7231), и URI для использования с сообщениями HTTP, как описано в [RFC 3986](https://tools.ietf.org/html/rfc3986).

HTTP-сообщения являются основой веб-разработки. Веб-браузеры и HTTP-клиенты, такие как cURL, создают HTTP-запрос, который отправляется на веб-сервер, предоставляющий HTTP-ответ. Серверный код получает HTTP-запрос и возвращает ответное сообщение HTTP.

Сообщения HTTP обычно абстрагируются от конечного потребителя, но как разработчики, мы как правило должны знать, как они структурированы и как получить доступ к ним или манипулировать ими для выполнения наших задач, будь то выполнение запроса к HTTP API или обработка входящего запроса.

Каждое сообщение HTTP-запроса имеет определенную форму:

```httpspec
POST /path HTTP/1.1
Host: example.com

foo=bar&baz=bat
```

Первая строка запроса является "строкой запроса" и содержит по порядку метод HTTP-запроса, целевой объект запроса (обычно либо абсолютный URI, либо путь на веб-сервере) и версию протокола HTTP. Далее следует один или несколько заголовков HTTP, пустая строка и текст сообщения.

Ответные сообщения HTTP имеют схожую структуру:

```httpspec
HTTP/1.1 200 OK
Content-Type: text/plain

This is the response body
```

Первая строка является "строкой состояния" и содержит по порядку версию протокола HTTP, код состояния HTTP и "формулировка причины" - удобочитаемое описание кода состояния. Как и сообщение запроса, за ним следует один или несколько заголовков HTTP, пустая строка и тело сообщения.

Интерфейсы, описанные в этом документе, являются абстракциями вокруг HTTP-сообщений и составляющих их элементов.

> Ключевые слова '**ДОЛЖНЫ**', '**НЕ ДОЛЖНЫ**', '**ТРЕБУЕТСЯ**', '**НУЖНО**', '**НЕ НУЖНО**', '**СЛЕДУЕТ**', '**НЕ СЛЕДУЕТ**', '**РЕКОМЕНДУЕТСЯ**', '**МОЖЕТ**' и '**НЕОБЯЗАТЕЛЬНО**' в этом документе должны толковаться так, как описано в [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Ссылки

- [RFC 2119](https://tools.ietf.org/html/rfc2119)
- [RFC 3986](https://tools.ietf.org/html/rfc3986)
- [RFC 7230](https://tools.ietf.org/html/rfc7230)
- [RFC 7231](https://tools.ietf.org/html/rfc7231)

## 1. Описание

### 1.1 Сообщения

HTTP-сообщение - это либо запрос от клиента к серверу, либо ответ от сервера клиенту. Это описание определяет интерфейсы для сообщений HTTP `Psr\Http\Message\RequestInterface` и `Psr\Http\Message\ResponseInterface` соответственно.

Оба интерфейса `Psr\Http\Message\RequestInterface` и `Psr\Http\Message\ResponseInterface` расширяют `Psr\Http\Message\MessageInterface`. В то время как интерфейс `Psr\Http\Message\MessageInterface` **МОЖЕТ** быть реализован напрямую, разработчики **ДОЛЖНЫ** **реализовать** интерфейсы `Psr\Http\Message\RequestInterface` и `Psr\Http\Message\ResponseInterface`.

В дальнейшем пространство имен `Psr\Http\Message` будет опущено при обращении к этим интерфейсам.

### 1.2 HTTP-Заголовки

#### Имена полей заголовка без учета регистра

HTTP-сообщения содержат имена полей заголовка без учета регистра. Заголовки извлекаются по имени из классов, реализующих интерфейс `MessageInterface` без учета регистра. Например, извлечение заголовка `foo` вернет тот же результат, что и извлечение заголовка `FoO`. Аналогично, установка заголовка `Foo` перезапишет любое ранее установленное значение заголовка в `foo`.

```php
$message = $message->withHeader('foo', 'bar');

echo $message->getHeaderLine('foo');
// Выведет: bar

echo $message->getHeaderLine('FOO');
// Выведет: bar

$message = $message->withHeader('fOO', 'baz');
echo $message->getHeaderLine('foo');
// Выведет: baz
```

Несмотря на то, что заголовки могут быть получены без учета регистра, исходный регистр ДОЛЖЕН быть сохранен реализацией, в частности, при получении с помощью `getHeaders()`.

Несоответствующие HTTP-приложения могут зависеть от определенного случая, поэтому пользователю полезно иметь возможность диктовать регистр HTTP-заголовков при создании запроса или ответа.

#### Заголовки с несколькими значениями

Чтобы разместить заголовки с несколькими значениями, но при этом обеспечить удобство работы с заголовками в виде строк, заголовки могут быть извлечены из экземпляра `MessageInterface` в виде массива или строки. Используйте метод `getHeaderLine()` для получения значения заголовка в виде строки, содержащей все значения заголовка без учета регистра имени, перечисленныые через запятую. Используйте `getHeader()`, чтобы по имени заголовка, получить массив всех значений конкретного заголовка нечувствительного к регистру.

```php
$message = $message
    ->withHeader('foo', 'bar')
    ->withAddedHeader('foo', 'baz');

$header = $message->getHeaderLine('foo');
// $header содержит: 'bar,baz'

$header = $message->getHeader('foo');
// ['bar', 'baz']
```

Примечание: не все значения заголовка могут быть объединены с помощью запятой (например, `Set-Cookie`). При работе с такими заголовками потребители классов основанных на `MessageInterface` **ДОЛЖНЫ** полагаться на метод `getHeader()` для извлечения таких многозначных заголовков.

#### Заголовок Host

В запросах, заголовок `Host`, обычно отражает хостовую часть от URI, а также хост, используемый при установлении TCP-соединения. Однако, спецификация HTTP для каждого из них позволяет заголовку `Host` отличаться.

Если заголовок `Host` не установлен, то во время создания, реализация ДОЛЖНА попытаться установить заголовок `Host` из предоставленного URI.

`RequestInterface::withUri()` по умолчанию заменит заголовок `Host` возвращаемого запроса, на заголовок c `Host` соответствующего части хоста переданного из `UriInterface`.

Вы можете отказаться от сохранения исходного состояния `Host` заголовка, передав `true` для второго (`$preserveHost`) аргумента. Когда для этого аргумента установлено значение `true`, возвращаемый запрос не обновит `Host` заголовок возвращаемого сообщения, если только сообщение не содержит `Host` заголовка.

Таблица ниже иллюстрирует, что `getHeaderLine('Host')` вернет запрос, возвращаемый `withUri()` с `$preserveHost` значением аргумента `true` для различных начальных запросов и URI.

|REQUEST HOST HEADER |REQUEST HOST COMPONENT |URI HOST COMPONENT |RESULT|
|---|---|---|---|
|''|''|''|''|
|''|foo.com|''|foo.com|
|''|foo.com|bar.com|foo.com|
|foo.com|''|bar.com|foo.com|
|foo.com|bar.com|baz.com|foo.com|

- 1 `Host` значение заголовка перед началом работы.
- 2 Основной компонент URI, созданный в запросе перед операцией.
- 3 Хост-компонент URI, вводимого через `withUri()`.

### 1.3 Потоки

HTTP-сообщения состоят из начальной строки, заголовков и тела. Тело HTTP-сообщения может быть очень маленьким или чрезвычайно большим. Попытка представить тело сообщения в виде строки может легко занять больше памяти, чем предполагалось, поскольку тело должно храниться полностью в памяти. Попытка сохранить тело запроса или ответа в памяти лишила бы использование этой реализации возможности работать с большими телами сообщений. `StreamInterface` используется для того, чтобы скрыть детали реализации при чтении или записи потока данных. Для ситуаций, когда строка была бы подходящей реализацией сообщения, могут использоваться встроенные потоки, такие как `php://memory` и `php://temp`.

`StreamInterface` предоставляет несколько методов, которые позволяют эффективно считывать, записывать и пересекать потоки.

Потоки раскрывают свои возможности с помощью трех методов: `isReadable()`, `isWritable()` и `isSeekable()`. Эти методы могут использоваться участниками stream collaborators, чтобы определить, соответствует ли stream их требованиям.

Каждый экземпляр stream будет обладать различными возможностями: он может быть доступен только для чтения, только для записи или с возможностью чтения-записи. Он также может разрешать произвольный произвольный доступ (поиск вперед или назад к любому местоположению) или только последовательный доступ (например, в случае сокета, канала или потока на основе обратного вызова).

Наконец, `StreamInterface` определяет `__toString()` метод, упрощающий получение или отправку всего основного содержимого сразу.

В отличие от интерфейсов запроса и ответа, `StreamInterface` не моделирует неизменяемость. В ситуациях, когда фактический поток PHP обернут, обеспечить неизменность невозможно, поскольку любой код, который взаимодействует с ресурсом, потенциально может изменить его состояние (включая положение курсора, содержимое и многое другое). Мы рекомендуем, чтобы реализации использовали потоки только для чтения для запросов на стороне сервера и ответов на стороне клиента. Потребители должны знать о том факте, что экземпляр stream может быть изменяемым и, как таковой, может изменять состояние сообщения; если вы сомневаетесь, создайте новый экземпляр stream и прикрепите его к сообщению, чтобы обеспечить соблюдение состояния.

### 1.4 Цели запросов и URI

Согласно RFC 7230, сообщения-запросы содержат "целевой запрос" в качестве второго сегмента строки запроса. Целевой запрос может иметь одну из следующих форм:

- **Исходная форма**, которая состоит из пути и, если присутствует, строки запроса; это часто называют относительным URL. Сообщения, передаваемые по TCP, обычно имеют исходную форму; схема и данные полномочий обычно представлены только через переменные CGI.
- **Абсолютная форма**, которая состоит из схемы, полномочий ("[user-info@]host[:port]", где элементы в скобках необязательны), пути (если присутствует), строки запроса (если присутствует) и фрагмента (если присутствует). Это часто упоминается как абсолютный URI и является единственной формой для указания URI, как описано в RFC 3986. Эта форма обычно используется при отправке запросов к HTTP-прокси.
- **Форма полномочий**, которая состоит только из полномочий. Обычно используется только в запросах CONNECT для установления соединения между HTTP-клиентом и прокси-сервером.
- **Звездочка-форма**, которая состоит исключительно из строки `*` и которая используется с методом ОПЦИЙ для определения общих возможностей веб-сервера.

Помимо этих целевых объектов запроса, часто существует "эффективный URL", который отделен от целевого объекта запроса. Эффективный URL-адрес не передается в HTTP-сообщении, но он используется для определения протокола (http / https), порта и имени хоста для отправки запроса.

Эффективный URL-адрес представлен символом `UriInterface`. `UriInterface`моделирует URI HTTP и HTTPS, как указано в RFC 3986 (основной вариант использования). Интерфейс предоставляет методы для взаимодействия с различными частями URI, что избавит от необходимости повторного анализа URI. Он также определяет `__toString()` метод для приведения смоделированного URI к его строковому представлению.

При получении целевого запроса с помощью `getRequestTarget()`, по умолчанию этот метод будет использовать объект URI и извлекать все необходимые компоненты для построения _исходной формы_. _Исходная форма_ на сегодняшний день является наиболее распространенной целью запроса.

Если конечный пользователь желает использовать одну из трех других форм, или если пользователь хочет явно переопределить целевой запрос, это можно сделать с помощью `withRequestTarget()`.

Вызов этого метода не влияет на URI, поскольку он возвращается из `getUri()`.

Например, пользователь может захотеть отправить запрос в форме звездочки на сервер:

```php
$request = $request
    ->withMethod('OPTIONS')
    ->withRequestTarget('*')
    ->withUri(new Uri('https://example.org/'));
```

Этот пример может в конечном итоге привести к HTTP-запросу, который выглядит следующим образом:

```http
OPTIONS * HTTP/1.1
```

Но HTTP-клиент сможет использовать действующий URL (от `getUri()`), для определения протокола, имени хоста и TCP-порта.

HTTP-клиент ДОЛЖЕН игнорировать значения `Uri::getPath()` и `Uri::getQuery()` и вместо этого использовать значение, возвращаемое `getRequestTarget()`, которое по умолчанию объединяет эти два значения.

Клиенты, которые решили не реализовывать 1 или более из 4 целевых форм запроса, все равно ДОЛЖНЫ использовать `getRequestTarget()`. Эти клиенты ДОЛЖНЫ отклонять целевые запросы, которые они не поддерживают, и НЕ ДОЛЖНЫ возвращаться к значениям из `getUri()`.

`RequestInterface` предоставляет методы для получения целевого запроса или создания нового экземпляра с предоставленным целевым запросом. По умолчанию, если в экземпляре специально не составлен целевой запрос, `getRequestTarget()` вернет исходную форму составленного URI (или "/", если URI не составлен). `withRequestTarget($requestTarget)` создает новый экземпляр с указанной целью запроса и, таким образом, позволяет разработчикам создавать сообщения-запросы, которые представляют три другие формы-цели запроса (абсолютную форму, форму полномочий и форму звездочки). При использовании составленный экземпляр URI все еще может быть полезен, особенно в клиентах, где он может использоваться для создания соединения с сервером.

### 1.5 Запросы на стороне сервера

`RequestInterface` обеспечивает общее представление сообщения HTTP-запроса. Однако запросы на стороне сервера требуют дополнительной обработки из-за особенностей серверной среды. Обработка на стороне сервера должна учитывать общий интерфейс шлюза (CGI) и, более конкретно, абстракцию PHP и расширение CGI через его серверные API (SAPI). PHP обеспечил упрощение маршалинга входных данных с помощью суперглобальных объектов, таких как:

- `$_COOKIE`, который десериализует и обеспечивает упрощенный доступ к HTTP-файлам cookie.
- `$_GET`, который десериализует и обеспечивает упрощенный доступ к аргументам строки запроса.
- `$_POST`, который десериализует и обеспечивает упрощенный доступ к параметрам urlencoded, отправленным через HTTP POST; в общем, это можно рассматривать как результаты синтаксического анализа тела сообщения.
- `$_FILES`, который предоставляет сериализованные метаданные при загрузке файлов.
- `$_SERVER`, который предоставляет доступ к переменным среды CGI / SAPI, которые обычно включают метод запроса, схему запроса, URI запроса и заголовки.

`ServerRequestInterface` расширяется `RequestInterface` для обеспечения абстракции вокруг этих различных суперглобальных объектов. Такая практика помогает уменьшить привязку потребителей к суперглобальным объектам, а также поощряет и продвигает возможность тестирования запросов потребителей.

Запрос сервера предоставляет одно дополнительное свойство, "атрибуты", чтобы предоставить потребителям возможность анализировать, декомпозировать и сопоставлять запрос с правилами, специфичными для конкретного приложения (такими как сопоставление путей, сопоставление схемы, сопоставление хоста и т.д.). Таким образом, запрос сервера также может обеспечивать обмен сообщениями между несколькими потребителями запроса.

### 1.6 Загруженные файлы

`ServerRequestInterface` определяет метод для получения дерева загружаемых файлов в нормализованной структуре, где каждый лист является экземпляром `UploadedFileInterface`.

У `$_FILES` суперглобального есть некоторые хорошо известные проблемы при работе с массивами входных файлов. В качестве примера, если у вас есть форма, которая отправляет массив файлов — например, входное имя "files", submitting `files[0]` и `files[1]` — PHP будет представлять это как:

```php
array(
    'files' => array(
        'name' => array(
            0 => 'file0.txt',
            1 => 'file1.html',
        ),
        'type' => array(
            0 => 'text/plain',
            1 => 'text/html',
        ),
        /* И так далее. */
    ),
)
```

вместо ожидаемого:

```php
array(
    'files' => array(
        0 => array(
            'name' => 'file0.txt',
            'type' => 'text/plain',
            /* И так далее. */
        ),
        1 => array(
            'name' => 'file1.html',
            'type' => 'text/html',
            /* И так далее. */
        ),
    ),
)
```

В результате потребители должны знать детали реализации этого языка, и написать код для сбора данных для данной загрузки.

Кроме того, существуют сценарии, в которых `$_FILES` не заполняется при загрузке файла:

- Когда HTTP - метод не используется `POST`.
- При модульном тестировании.
- При работе в среде, отличной от SAPI, такой как [ReactPHP](https://reactphp.org/).

В таких случаях данные необходимо будет заполнять по-другому. В качестве примеров:

- Процесс может проанализировать тело сообщения, чтобы обнаружить загруженные файлы. В таких случаях реализация может выбрать _не_ записывать загружаемые файлы в файловую систему, а вместо этого обернуть их в поток, чтобы уменьшить нагрузку на память, ввод-вывод и хранилище.
- В сценариях модульного тестирования разработчикам необходимо иметь возможность заглушки и / или имитировать метаданные загрузки файла для проверки различных сценариев.

`getUploadedFiles()` обеспечивает нормализованную структуру для потребителей. ожидается, что реализации будут:

- Агрегируйте всю информацию для загрузки данного файла и используйте ее для заполнения `Psr\Http\Message\UploadedFileInterface` экземпляра.
- Воссоздайте представленную древовидную структуру, где каждый лист является соответствующим `Psr\Http\Message\UploadedFileInterface` экземпляром для данного местоположения в дереве.

Древовидная структура, на которую ссылаются, должна имитировать структуру именования, в которой были отправлены файлы.

В простейшем примере это может быть один именованный элемент формы, представленный как:

```html
<input type="file" name="avatar" />
```

В этом случае структура в `$_FILES` будет выглядеть следующим образом:

```php
array(
    'avatar' => array(
        'tmp_name' => 'phpUxcOty',
        'name' => 'my-avatar.png',
        'size' => 90996,
        'type' => 'image/png',
        'error' => 0,
    ),
)
```

Нормализованная форма, возвращаемая `getUploadedFiles()`, будет иметь вид:

```php
array(
    'avatar' => /* UploadedFileInterface instance */
)
```

В случае ввода с использованием обозначения массива для имени:

```html
<input type="file" name="my-form[details][avatar]" />
```

`$_FILES` в конечном итоге выглядит следующим образом:

```php
array (
    'my-form' => array (
        'name' => array (
            'details' => array (
                'avatar' => 'my-avatar.png',
            ),
        ),
        'type' => array (
            'details' => array (
                'avatar' => 'image/png',
            ),
        ),
        'tmp_name' => array (
            'details' => array (
                'avatar' => 'phpmFLrzD',
            ),
        ),
        'error' => array (
            'details' => array (
                'avatar' => 0,
            ),
        ),
        'size' => array (
            'details' => array (
                'avatar' => 90996,
            ),
        ),
    ),
)
```

И соответствующее дерево, возвращаемое `getUploadedFiles()`, должно быть:

```php
array(
    'my-form' => array(
        'details' => array(
            'avatar' => /* UploadedFileInterface instance */
        ),
    ),
)
```

В некоторых случаях вы можете указать массив файлов:

```html
Upload an avatar: <input type="file" name="my-form[details][avatars][]" />
Upload an avatar: <input type="file" name="my-form[details][avatars][]" />
```

(В качестве примера, элементы управления JavaScript могут создавать дополнительные входные данные для загрузки файлов, чтобы разрешить загрузку нескольких файлов одновременно.)

В таком случае реализация спецификации должна агрегировать всю информацию, относящуюся к файлу, по заданному индексу. Причина в том, что `$_FILES` в таких случаях отклоняется от своей обычной структуры.:

```php
array (
    'my-form' => array (
        'name' => array (
            'details' => array (
                'avatars' => array (
                    0 => 'my-avatar.png',
                    1 => 'my-avatar2.png',
                    2 => 'my-avatar3.png',
                ),
            ),
        ),
        'type' => array (
            'details' => array (
                'avatars' => array (
                    0 => 'image/png',
                    1 => 'image/png',
                    2 => 'image/png',
                ),
            ),
        ),
        'tmp_name' => array (
            'details' => array (
                'avatars' => array (
                    0 => 'phpmFLrzD',
                    1 => 'phpV2pBil',
                    2 => 'php8RUG8v',
                ),
            ),
        ),
        'error' => array (
            'details' => array (
                'avatars' => array (
                    0 => 0,
                    1 => 0,
                    2 => 0,
                ),
            ),
        ),
        'size' => array (
            'details' => array (
                'avatars' => array (
                    0 => 90996,
                    1 => 90996,
                    3 => 90996,
                ),
            ),
        ),
    ),
)
```

Приведенный выше `$_FILES` массив будет соответствовать следующей структуре, поскольку возвращается `getUploadedFiles()`:

```php
array(
    'my-form' => array(
        'details' => array(
            'avatars' => array(
                0 => /* UploadedFileInterface instance */,
                1 => /* UploadedFileInterface instance */,
                2 => /* UploadedFileInterface instance */,
            ),
        ),
    ),
)
```

Потребители будут получать доступ к индексу `1` вложенного массива с помощью:

```php
$request->getUploadedFiles()['my-form']['details']['avatars'][1];
```

Ведь загруженные файлы данных, является производной (производная от `$_FILES` или тело запроса), мутатор метода `withUploadedFiles()`, также присутствует в интерфейс, делегация позволяя нормализации на другой процесс.

В случае с исходными примерами использование выглядит следующим образом:

```php
$file0 = $request->getUploadedFiles()['files'][0];
$file1 = $request->getUploadedFiles()['files'][1];

printf(
    "Received the files %s and %s",
    $file0->getClientFilename(),
    $file1->getClientFilename()
);

// "Получены файлы file0.txt и file1.html"
```

В этом предложении также признается, что реализации могут работать в средах, отличных от SAPI . Таким образом, `UploadedFileInterface` предоставляет методы для обеспечения того, чтобы операции работали независимо от среды. В частности:

- `moveTo($targetPath)` предоставляется как безопасная и рекомендуемая альтернатива вызову `move_uploaded_file()` непосредственно из файла временной загрузки. Реализации определят правильную операцию для использования в зависимости от среды.
- `getStream()` вернет `StreamInterface` экземпляр. В средах, отличных от SAPI, одной из предлагаемых возможностей является разбор отдельных загружаемых файлов в `php://temp` потоки, а не непосредственно в файлы; в таких случаях загружаемый файл отсутствует. `getStream()` таким образом, гарантируется работа независимо от среды.

В качестве примера:

```php
// Переместить файл в каталог загрузки
$filename = sprintf(
    '%s.%s',
    create_uuid(),
    pathinfo($file0->getClientFilename(), PATHINFO_EXTENSION)
);
$file0->moveTo(DATA_DIR . '/' . $filename);

// Потоковая передача файла на Amazon S3.
// Предположим, что $s3 wrapper - это поток PHP, который будет записывать в S3, и что
// psr7streamwrapper - это класс, который оформит StreamInterface как PHP
// StreamWrapper.
$stream = new Psr7StreamWrapper($file1->getStream());
stream_copy_to_stream($stream, $s3wrapper);
```

## 2. Пакет

Описанные интерфейсы и классы предоставляются как часть пакета [psr/http-message](https://packagist.org/packages/psr/http-message).

## 3. Интерфейсы

### 3.1 `Psr\Http\Message\MessageInterface`

```php
<?php
namespace Psr\Http\Message;

/**
 * HTTP-сообщения состоят из запросов от клиента к серверу и ответов
 * от сервера клиенту. Этот интерфейс определяет методы, общие для
 * каждого из них.
 *
 * Сообщения считаются неизменяемыми; все методы, которые могут изменять состояние, ДОЛЖНЫ
 * быть реализованы таким образом, чтобы они сохраняли внутреннее состояние текущего
 * сообщения и возвращали экземпляр, содержащий измененное состояние.
 *
 * @see http://www.ietf.org/rfc/rfc7230.txt
 * @see http://www.ietf.org/rfc/rfc7231.txt
 */
interface MessageInterface
{
    /**
     * Извлекает версию протокола HTTP в виде строки.
     *
     * Строка ДОЛЖНА содержать только номер версии HTTP (например, "1.1", "1.0").
     *
     * @return string HTTP версия протокола.
     */
    public function getProtocolVersion();

    /**
     * Возвращает экземпляр с указанной версией протокола HTTP.
     *
     * Строка версии ДОЛЖНА содержать только номер версии HTTP (например,
     * "1.1", "1.0").
     *
     * Этот метод ДОЛЖЕН быть реализован таким образом, чтобы сохранить
     * неизменность сообщения, и должен возвращать экземпляр, имеющий
     * новую версию протокола.
     *
     * @param string $version HTTP версия протокола.
     * @return static
     */
    public function withProtocolVersion($version);

    /**
     * Retrieves all message header values.
     *
     * The keys represent the header name as it will be sent over the wire, and
     * each value is an array of strings associated with the header.
     *
     *     // Represent the headers as a string
     *     foreach ($message->getHeaders() as $name => $values) {
     *         echo $name . ': ' . implode(', ', $values);
     *     }
     *
     *     // Emit headers iteratively:
     *     foreach ($message->getHeaders() as $name => $values) {
     *         foreach ($values as $value) {
     *             header(sprintf('%s: %s', $name, $value), false);
     *         }
     *     }
     *
     * While header names are not case-sensitive, getHeaders() will preserve the
     * exact case in which headers were originally specified.
     *
     * @return string[][] Returns an associative array of the message's headers.
     *     Each key MUST be a header name, and each value MUST be an array of
     *     strings for that header.
     */
    public function getHeaders();

    /**
     * Checks if a header exists by the given case-insensitive name.
     *
     * @param string $name Case-insensitive header field name.
     * @return bool Returns true if any header names match the given header
     *     name using a case-insensitive string comparison. Returns false if
     *     no matching header name is found in the message.
     */
    public function hasHeader($name);

    /**
     * Retrieves a message header value by the given case-insensitive name.
     *
     * This method returns an array of all the header values of the given
     * case-insensitive header name.
     *
     * If the header does not appear in the message, this method MUST return an
     * empty array.
     *
     * @param string $name Case-insensitive header field name.
     * @return string[] An array of string values as provided for the given
     *    header. If the header does not appear in the message, this method MUST
     *    return an empty array.
     */
    public function getHeader($name);

    /**
     * Retrieves a comma-separated string of the values for a single header.
     *
     * This method returns all of the header values of the given
     * case-insensitive header name as a string concatenated together using
     * a comma.
     *
     * NOTE: Not all header values may be appropriately represented using
     * comma concatenation. For such headers, use getHeader() instead
     * and supply your own delimiter when concatenating.
     *
     * If the header does not appear in the message, this method MUST return
     * an empty string.
     *
     * @param string $name Case-insensitive header field name.
     * @return string A string of values as provided for the given header
     *    concatenated together using a comma. If the header does not appear in
     *    the message, this method MUST return an empty string.
     */
    public function getHeaderLine($name);

    /**
     * Return an instance with the provided value replacing the specified header.
     *
     * While header names are case-insensitive, the casing of the header will
     * be preserved by this function, and returned from getHeaders().
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * new and/or updated header and value.
     *
     * @param string $name Case-insensitive header field name.
     * @param string|string[] $value Header value(s).
     * @return static
     * @throws \InvalidArgumentException for invalid header names or values.
     */
    public function withHeader($name, $value);

    /**
     * Return an instance with the specified header appended with the given value.
     *
     * Existing values for the specified header will be maintained. The new
     * value(s) will be appended to the existing list. If the header did not
     * exist previously, it will be added.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * new header and/or value.
     *
     * @param string $name Case-insensitive header field name to add.
     * @param string|string[] $value Header value(s).
     * @return static
     * @throws \InvalidArgumentException for invalid header names.
     * @throws \InvalidArgumentException for invalid header values.
     */
    public function withAddedHeader($name, $value);

    /**
     * Return an instance without the specified header.
     *
     * Header resolution MUST be done without case-sensitivity.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that removes
     * the named header.
     *
     * @param string $name Case-insensitive header field name to remove.
     * @return static
     */
    public function withoutHeader($name);

    /**
     * Gets the body of the message.
     *
     * @return StreamInterface Returns the body as a stream.
     */
    public function getBody();

    /**
     * Return an instance with the specified message body.
     *
     * The body MUST be a StreamInterface object.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return a new instance that has the
     * new body stream.
     *
     * @param StreamInterface $body Body.
     * @return static
     * @throws \InvalidArgumentException When the body is not valid.
     */
    public function withBody(StreamInterface $body);
}
```

### 3.2 `Psr\Http\Message\RequestInterface`

```php
<?php
namespace Psr\Http\Message;

/**
 * Representation of an outgoing, client-side request.
 *
 * Per the HTTP specification, this interface includes properties for
 * each of the following:
 *
 * - Protocol version
 * - HTTP method
 * - URI
 * - Headers
 * - Message body
 *
 * During construction, implementations MUST attempt to set the Host header from
 * a provided URI if no Host header is provided.
 *
 * Requests are considered immutable; all methods that might change state MUST
 * be implemented such that they retain the internal state of the current
 * message and return an instance that contains the changed state.
 */
interface RequestInterface extends MessageInterface
{
    /**
     * Retrieves the message's request target.
     *
     * Retrieves the message's request-target either as it will appear (for
     * clients), as it appeared at request (for servers), or as it was
     * specified for the instance (see withRequestTarget()).
     *
     * In most cases, this will be the origin-form of the composed URI,
     * unless a value was provided to the concrete implementation (see
     * withRequestTarget() below).
     *
     * If no URI is available, and no request-target has been specifically
     * provided, this method MUST return the string "/".
     *
     * @return string
     */
    public function getRequestTarget();

    /**
     * Return an instance with the specific request-target.
     *
     * If the request needs a non-origin-form request-target — e.g., for
     * specifying an absolute-form, authority-form, or asterisk-form —
     * this method may be used to create an instance with the specified
     * request-target, verbatim.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * changed request target.
     *
     * @see http://tools.ietf.org/html/rfc7230#section-5.3 (for the various
     *     request-target forms allowed in request messages)
     * @param mixed $requestTarget
     * @return static
     */
    public function withRequestTarget($requestTarget);

    /**
     * Retrieves the HTTP method of the request.
     *
     * @return string Returns the request method.
     */
    public function getMethod();

    /**
     * Return an instance with the provided HTTP method.
     *
     * While HTTP method names are typically all uppercase characters, HTTP
     * method names are case-sensitive and thus implementations SHOULD NOT
     * modify the given string.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * changed request method.
     *
     * @param string $method Case-sensitive method.
     * @return static
     * @throws \InvalidArgumentException for invalid HTTP methods.
     */
    public function withMethod($method);

    /**
     * Retrieves the URI instance.
     *
     * This method MUST return a UriInterface instance.
     *
     * @see http://tools.ietf.org/html/rfc3986#section-4.3
     * @return UriInterface Returns a UriInterface instance
     *     representing the URI of the request.
     */
    public function getUri();

    /**
     * Returns an instance with the provided URI.
     *
     * This method MUST update the Host header of the returned request by
     * default if the URI contains a host component. If the URI does not
     * contain a host component, any pre-existing Host header MUST be carried
     * over to the returned request.
     *
     * You can opt-in to preserving the original state of the Host header by
     * setting `$preserveHost` to `true`. When `$preserveHost` is set to
     * `true`, this method interacts with the Host header in the following ways:
     *
     * - If the Host header is missing or empty, and the new URI contains
     *   a host component, this method MUST update the Host header in the returned
     *   request.
     * - If the Host header is missing or empty, and the new URI does not contain a
     *   host component, this method MUST NOT update the Host header in the returned
     *   request.
     * - If a Host header is present and non-empty, this method MUST NOT update
     *   the Host header in the returned request.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * new UriInterface instance.
     *
     * @see http://tools.ietf.org/html/rfc3986#section-4.3
     * @param UriInterface $uri New request URI to use.
     * @param bool $preserveHost Preserve the original state of the Host header.
     * @return static
     */
    public function withUri(UriInterface $uri, $preserveHost = false);
}
```

#### 3.2.1 `Psr\Http\Message\ServerRequestInterface`

```php
<?php
namespace Psr\Http\Message;

/**
 * Representation of an incoming, server-side HTTP request.
 *
 * Per the HTTP specification, this interface includes properties for
 * each of the following:
 *
 * - Protocol version
 * - HTTP method
 * - URI
 * - Headers
 * - Message body
 *
 * Additionally, it encapsulates all data as it has arrived at the
 * application from the CGI and/or PHP environment, including:
 *
 * - The values represented in $_SERVER.
 * - Any cookies provided (generally via $_COOKIE)
 * - Query string arguments (generally via $_GET, or as parsed via parse_str())
 * - Upload files, if any (as represented by $_FILES)
 * - Deserialized body parameters (generally from $_POST)
 *
 * $_SERVER values MUST be treated as immutable, as they represent application
 * state at the time of request; as such, no methods are provided to allow
 * modification of those values. The other values provide such methods, as they
 * can be restored from $_SERVER or the request body, and may need treatment
 * during the application (e.g., body parameters may be deserialized based on
 * content type).
 *
 * Additionally, this interface recognizes the utility of introspecting a
 * request to derive and match additional parameters (e.g., via URI path
 * matching, decrypting cookie values, deserializing non-form-encoded body
 * content, matching authorization headers to users, etc). These parameters
 * are stored in an "attributes" property.
 *
 * Requests are considered immutable; all methods that might change state MUST
 * be implemented such that they retain the internal state of the current
 * message and return an instance that contains the changed state.
 */
interface ServerRequestInterface extends RequestInterface
{
    /**
     * Retrieve server parameters.
     *
     * Retrieves data related to the incoming request environment,
     * typically derived from PHP's $_SERVER superglobal. The data IS NOT
     * REQUIRED to originate from $_SERVER.
     *
     * @return array
     */
    public function getServerParams();

    /**
     * Retrieve cookies.
     *
     * Retrieves cookies sent by the client to the server.
     *
     * The data MUST be compatible with the structure of the $_COOKIE
     * superglobal.
     *
     * @return array
     */
    public function getCookieParams();

    /**
     * Return an instance with the specified cookies.
     *
     * The data IS NOT REQUIRED to come from the $_COOKIE superglobal, but MUST
     * be compatible with the structure of $_COOKIE. Typically, this data will
     * be injected at instantiation.
     *
     * This method MUST NOT update the related Cookie header of the request
     * instance, nor related values in the server params.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated cookie values.
     *
     * @param array $cookies Array of key/value pairs representing cookies.
     * @return static
     */
    public function withCookieParams(array $cookies);

    /**
     * Retrieve query string arguments.
     *
     * Retrieves the deserialized query string arguments, if any.
     *
     * Note: the query params might not be in sync with the URI or server
     * params. If you need to ensure you are only getting the original
     * values, you may need to parse the query string from `getUri()->getQuery()`
     * or from the `QUERY_STRING` server param.
     *
     * @return array
     */
    public function getQueryParams();

    /**
     * Return an instance with the specified query string arguments.
     *
     * These values SHOULD remain immutable over the course of the incoming
     * request. They MAY be injected during instantiation, such as from PHP's
     * $_GET superglobal, or MAY be derived from some other value such as the
     * URI. In cases where the arguments are parsed from the URI, the data
     * MUST be compatible with what PHP's parse_str() would return for
     * purposes of how duplicate query parameters are handled, and how nested
     * sets are handled.
     *
     * Setting query string arguments MUST NOT change the URI stored by the
     * request, nor the values in the server params.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated query string arguments.
     *
     * @param array $query Array of query string arguments, typically from
     *     $_GET.
     * @return static
     */
    public function withQueryParams(array $query);

    /**
     * Retrieve normalized file upload data.
     *
     * This method returns upload metadata in a normalized tree, with each leaf
     * an instance of Psr\Http\Message\UploadedFileInterface.
     *
     * These values MAY be prepared from $_FILES or the message body during
     * instantiation, or MAY be injected via withUploadedFiles().
     *
     * @return array An array tree of UploadedFileInterface instances; an empty
     *     array MUST be returned if no data is present.
     */
    public function getUploadedFiles();

    /**
     * Create a new instance with the specified uploaded files.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated body parameters.
     *
     * @param array $uploadedFiles An array tree of UploadedFileInterface instances.
     * @return static
     * @throws \InvalidArgumentException if an invalid structure is provided.
     */
    public function withUploadedFiles(array $uploadedFiles);

    /**
     * Retrieve any parameters provided in the request body.
     *
     * If the request Content-Type is either application/x-www-form-urlencoded
     * or multipart/form-data, and the request method is POST, this method MUST
     * return the contents of $_POST.
     *
     * Otherwise, this method may return any results of deserializing
     * the request body content; as parsing returns structured content, the
     * potential types MUST be arrays or objects only. A null value indicates
     * the absence of body content.
     *
     * @return null|array|object The deserialized body parameters, if any.
     *     These will typically be an array or object.
     */
    public function getParsedBody();

    /**
     * Return an instance with the specified body parameters.
     *
     * These MAY be injected during instantiation.
     *
     * If the request Content-Type is either application/x-www-form-urlencoded
     * or multipart/form-data, and the request method is POST, use this method
     * ONLY to inject the contents of $_POST.
     *
     * The data IS NOT REQUIRED to come from $_POST, but MUST be the results of
     * deserializing the request body content. Deserialization/parsing returns
     * structured data, and, as such, this method ONLY accepts arrays or objects,
     * or a null value if nothing was available to parse.
     *
     * As an example, if content negotiation determines that the request data
     * is a JSON payload, this method could be used to create a request
     * instance with the deserialized parameters.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated body parameters.
     *
     * @param null|array|object $data The deserialized body data. This will
     *     typically be in an array or object.
     * @return static
     * @throws \InvalidArgumentException if an unsupported argument type is
     *     provided.
     */
    public function withParsedBody($data);

    /**
     * Retrieve attributes derived from the request.
     *
     * The request "attributes" may be used to allow injection of any
     * parameters derived from the request: e.g., the results of path
     * match operations; the results of decrypting cookies; the results of
     * deserializing non-form-encoded message bodies; etc. Attributes
     * will be application and request specific, and CAN be mutable.
     *
     * @return mixed[] Attributes derived from the request.
     */
    public function getAttributes();

    /**
     * Retrieve a single derived request attribute.
     *
     * Retrieves a single derived request attribute as described in
     * getAttributes(). If the attribute has not been previously set, returns
     * the default value as provided.
     *
     * This method obviates the need for a hasAttribute() method, as it allows
     * specifying a default value to return if the attribute is not found.
     *
     * @see getAttributes()
     * @param string $name The attribute name.
     * @param mixed $default Default value to return if the attribute does not exist.
     * @return mixed
     */
    public function getAttribute($name, $default = null);

    /**
     * Return an instance with the specified derived request attribute.
     *
     * This method allows setting a single derived request attribute as
     * described in getAttributes().
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated attribute.
     *
     * @see getAttributes()
     * @param string $name The attribute name.
     * @param mixed $value The value of the attribute.
     * @return static
     */
    public function withAttribute($name, $value);

    /**
     * Return an instance that removes the specified derived request attribute.
     *
     * This method allows removing a single derived request attribute as
     * described in getAttributes().
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that removes
     * the attribute.
     *
     * @see getAttributes()
     * @param string $name The attribute name.
     * @return static
     */
    public function withoutAttribute($name);
}
```

### 3.3 `Psr\Http\Message\ResponseInterface`

```php
<?php
namespace Psr\Http\Message;

/**
 * Representation of an outgoing, server-side response.
 *
 * Per the HTTP specification, this interface includes properties for
 * each of the following:
 *
 * - Protocol version
 * - Status code and reason phrase
 * - Headers
 * - Message body
 *
 * Responses are considered immutable; all methods that might change state MUST
 * be implemented such that they retain the internal state of the current
 * message and return an instance that contains the changed state.
 */
interface ResponseInterface extends MessageInterface
{
    /**
     * Gets the response status code.
     *
     * The status code is a 3-digit integer result code of the server's attempt
     * to understand and satisfy the request.
     *
     * @return int Status code.
     */
    public function getStatusCode();

    /**
     * Return an instance with the specified status code and, optionally, reason phrase.
     *
     * If no reason phrase is specified, implementations MAY choose to default
     * to the RFC 7231 or IANA recommended reason phrase for the response's
     * status code.
     *
     * This method MUST be implemented in such a way as to retain the
     * immutability of the message, and MUST return an instance that has the
     * updated status and reason phrase.
     *
     * @see http://tools.ietf.org/html/rfc7231#section-6
     * @see http://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml
     * @param int $code The 3-digit integer result code to set.
     * @param string $reasonPhrase The reason phrase to use with the
     *     provided status code; if none is provided, implementations MAY
     *     use the defaults as suggested in the HTTP specification.
     * @return static
     * @throws \InvalidArgumentException For invalid status code arguments.
     */
    public function withStatus($code, $reasonPhrase = '');

    /**
     * Gets the response reason phrase associated with the status code.
     *
     * Because a reason phrase is not a required element in a response
     * status line, the reason phrase value MAY be empty. Implementations MAY
     * choose to return the default RFC 7231 recommended reason phrase (or those
     * listed in the IANA HTTP Status Code Registry) for the response's
     * status code.
     *
     * @see http://tools.ietf.org/html/rfc7231#section-6
     * @see http://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml
     * @return string Reason phrase; must return an empty string if none present.
     */
    public function getReasonPhrase();
}
```

### 3.4 `Psr\Http\Message\StreamInterface`

```php
<?php
namespace Psr\Http\Message;

/**
 * Describes a data stream.
 *
 * Typically, an instance will wrap a PHP stream; this interface provides
 * a wrapper around the most common operations, including serialization of
 * the entire stream to a string.
 */
interface StreamInterface
{
    /**
     * Reads all data from the stream into a string, from the beginning to end.
     *
     * This method MUST attempt to seek to the beginning of the stream before
     * reading data and read the stream until the end is reached.
     *
     * Warning: This could attempt to load a large amount of data into memory.
     *
     * This method MUST NOT raise an exception in order to conform with PHP's
     * string casting operations.
     *
     * @see http://php.net/manual/en/language.oop5.magic.php#object.tostring
     * @return string
     */
    public function __toString();

    /**
     * Closes the stream and any underlying resources.
     *
     * @return void
     */
    public function close();

    /**
     * Separates any underlying resources from the stream.
     *
     * After the stream has been detached, the stream is in an unusable state.
     *
     * @return resource|null Underlying PHP stream, if any
     */
    public function detach();

    /**
     * Get the size of the stream if known.
     *
     * @return int|null Returns the size in bytes if known, or null if unknown.
     */
    public function getSize();

    /**
     * Returns the current position of the file read/write pointer
     *
     * @return int Position of the file pointer
     * @throws \RuntimeException on error.
     */
    public function tell();

    /**
     * Returns true if the stream is at the end of the stream.
     *
     * @return bool
     */
    public function eof();

    /**
     * Returns whether or not the stream is seekable.
     *
     * @return bool
     */
    public function isSeekable();

    /**
     * Seek to a position in the stream.
     *
     * @see http://www.php.net/manual/en/function.fseek.php
     * @param int $offset Stream offset
     * @param int $whence Specifies how the cursor position will be calculated
     *     based on the seek offset. Valid values are identical to the built-in
     *     PHP $whence values for `fseek()`.  SEEK_SET: Set position equal to
     *     offset bytes SEEK_CUR: Set position to current location plus offset
     *     SEEK_END: Set position to end-of-stream plus offset.
     * @throws \RuntimeException on failure.
     */
    public function seek($offset, $whence = SEEK_SET);

    /**
     * Seek to the beginning of the stream.
     *
     * If the stream is not seekable, this method will raise an exception;
     * otherwise, it will perform a seek(0).
     *
     * @see seek()
     * @see http://www.php.net/manual/en/function.fseek.php
     * @throws \RuntimeException on failure.
     */
    public function rewind();

    /**
     * Returns whether or not the stream is writable.
     *
     * @return bool
     */
    public function isWritable();

    /**
     * Write data to the stream.
     *
     * @param string $string The string that is to be written.
     * @return int Returns the number of bytes written to the stream.
     * @throws \RuntimeException on failure.
     */
    public function write($string);

    /**
     * Returns whether or not the stream is readable.
     *
     * @return bool
     */
    public function isReadable();

    /**
     * Read data from the stream.
     *
     * @param int $length Read up to $length bytes from the object and return
     *     them. Fewer than $length bytes may be returned if underlying stream
     *     call returns fewer bytes.
     * @return string Returns the data read from the stream, or an empty string
     *     if no bytes are available.
     * @throws \RuntimeException if an error occurs.
     */
    public function read($length);

    /**
     * Returns the remaining contents in a string
     *
     * @return string
     * @throws \RuntimeException if unable to read.
     * @throws \RuntimeException if error occurs while reading.
     */
    public function getContents();

    /**
     * Get stream metadata as an associative array or retrieve a specific key.
     *
     * The keys returned are identical to the keys returned from PHP's
     * stream_get_meta_data() function.
     *
     * @see http://php.net/manual/en/function.stream-get-meta-data.php
     * @param string $key Specific metadata to retrieve.
     * @return array|mixed|null Returns an associative array if no key is
     *     provided. Returns a specific key value if a key is provided and the
     *     value is found, or null if the key is not found.
     */
    public function getMetadata($key = null);
}
```

### 3.5 `Psr\Http\Message\UriInterface`

```php
<?php
namespace Psr\Http\Message;

/**
 * Value object representing a URI.
 *
 * This interface is meant to represent URIs according to RFC 3986 and to
 * provide methods for most common operations. Additional functionality for
 * working with URIs can be provided on top of the interface or externally.
 * Its primary use is for HTTP requests, but may also be used in other
 * contexts.
 *
 * Instances of this interface are considered immutable; all methods that
 * might change state MUST be implemented such that they retain the internal
 * state of the current instance and return an instance that contains the
 * changed state.
 *
 * Typically the Host header will also be present in the request message.
 * For server-side requests, the scheme will typically be discoverable in the
 * server parameters.
 *
 * @see http://tools.ietf.org/html/rfc3986 (the URI specification)
 */
interface UriInterface
{
    /**
     * Retrieve the scheme component of the URI.
     *
     * If no scheme is present, this method MUST return an empty string.
     *
     * The value returned MUST be normalized to lowercase, per RFC 3986
     * Section 3.1.
     *
     * The trailing ":" character is not part of the scheme and MUST NOT be
     * added.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-3.1
     * @return string The URI scheme.
     */
    public function getScheme();

    /**
     * Retrieve the authority component of the URI.
     *
     * If no authority information is present, this method MUST return an empty
     * string.
     *
     * The authority syntax of the URI is:
     *
     * <pre>
     * [user-info@]host[:port]
     * </pre>
     *
     * If the port component is not set or is the standard port for the current
     * scheme, it SHOULD NOT be included.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-3.2
     * @return string The URI authority, in "[user-info@]host[:port]" format.
     */
    public function getAuthority();

    /**
     * Retrieve the user information component of the URI.
     *
     * If no user information is present, this method MUST return an empty
     * string.
     *
     * If a user is present in the URI, this will return that value;
     * additionally, if the password is also present, it will be appended to the
     * user value, with a colon (":") separating the values.
     *
     * The trailing "@" character is not part of the user information and MUST
     * NOT be added.
     *
     * @return string The URI user information, in "username[:password]" format.
     */
    public function getUserInfo();

    /**
     * Retrieve the host component of the URI.
     *
     * If no host is present, this method MUST return an empty string.
     *
     * The value returned MUST be normalized to lowercase, per RFC 3986
     * Section 3.2.2.
     *
     * @see http://tools.ietf.org/html/rfc3986#section-3.2.2
     * @return string The URI host.
     */
    public function getHost();

    /**
     * Retrieve the port component of the URI.
     *
     * If a port is present, and it is non-standard for the current scheme,
     * this method MUST return it as an integer. If the port is the standard port
     * used with the current scheme, this method SHOULD return null.
     *
     * If no port is present, and no scheme is present, this method MUST return
     * a null value.
     *
     * If no port is present, but a scheme is present, this method MAY return
     * the standard port for that scheme, but SHOULD return null.
     *
     * @return null|int The URI port.
     */
    public function getPort();

    /**
     * Retrieve the path component of the URI.
     *
     * The path can either be empty or absolute (starting with a slash) or
     * rootless (not starting with a slash). Implementations MUST support all
     * three syntaxes.
     *
     * Normally, the empty path "" and absolute path "/" are considered equal as
     * defined in RFC 7230 Section 2.7.3. But this method MUST NOT automatically
     * do this normalization because in contexts with a trimmed base path, e.g.
     * the front controller, this difference becomes significant. It's the task
     * of the user to handle both "" and "/".
     *
     * The value returned MUST be percent-encoded, but MUST NOT double-encode
     * any characters. To determine what characters to encode, please refer to
     * RFC 3986, Sections 2 and 3.3.
     *
     * As an example, if the value should include a slash ("/") not intended as
     * delimiter between path segments, that value MUST be passed in encoded
     * form (e.g., "%2F") to the instance.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-2
     * @see https://tools.ietf.org/html/rfc3986#section-3.3
     * @return string The URI path.
     */
    public function getPath();

    /**
     * Retrieve the query string of the URI.
     *
     * If no query string is present, this method MUST return an empty string.
     *
     * The leading "?" character is not part of the query and MUST NOT be
     * added.
     *
     * The value returned MUST be percent-encoded, but MUST NOT double-encode
     * any characters. To determine what characters to encode, please refer to
     * RFC 3986, Sections 2 and 3.4.
     *
     * As an example, if a value in a key/value pair of the query string should
     * include an ampersand ("&") not intended as a delimiter between values,
     * that value MUST be passed in encoded form (e.g., "%26") to the instance.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-2
     * @see https://tools.ietf.org/html/rfc3986#section-3.4
     * @return string The URI query string.
     */
    public function getQuery();

    /**
     * Retrieve the fragment component of the URI.
     *
     * If no fragment is present, this method MUST return an empty string.
     *
     * The leading "#" character is not part of the fragment and MUST NOT be
     * added.
     *
     * The value returned MUST be percent-encoded, but MUST NOT double-encode
     * any characters. To determine what characters to encode, please refer to
     * RFC 3986, Sections 2 and 3.5.
     *
     * @see https://tools.ietf.org/html/rfc3986#section-2
     * @see https://tools.ietf.org/html/rfc3986#section-3.5
     * @return string The URI fragment.
     */
    public function getFragment();

    /**
     * Return an instance with the specified scheme.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified scheme.
     *
     * Implementations MUST support the schemes "http" and "https" case
     * insensitively, and MAY accommodate other schemes if required.
     *
     * An empty scheme is equivalent to removing the scheme.
     *
     * @param string $scheme The scheme to use with the new instance.
     * @return static A new instance with the specified scheme.
     * @throws \InvalidArgumentException for invalid schemes.
     * @throws \InvalidArgumentException for unsupported schemes.
     */
    public function withScheme($scheme);

    /**
     * Return an instance with the specified user information.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified user information.
     *
     * Password is optional, but the user information MUST include the
     * user; an empty string for the user is equivalent to removing user
     * information.
     *
     * @param string $user The user name to use for authority.
     * @param null|string $password The password associated with $user.
     * @return static A new instance with the specified user information.
     */
    public function withUserInfo($user, $password = null);

    /**
     * Return an instance with the specified host.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified host.
     *
     * An empty host value is equivalent to removing the host.
     *
     * @param string $host The hostname to use with the new instance.
     * @return static A new instance with the specified host.
     * @throws \InvalidArgumentException for invalid hostnames.
     */
    public function withHost($host);

    /**
     * Return an instance with the specified port.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified port.
     *
     * Implementations MUST raise an exception for ports outside the
     * established TCP and UDP port ranges.
     *
     * A null value provided for the port is equivalent to removing the port
     * information.
     *
     * @param null|int $port The port to use with the new instance; a null value
     *     removes the port information.
     * @return static A new instance with the specified port.
     * @throws \InvalidArgumentException for invalid ports.
     */
    public function withPort($port);

    /**
     * Return an instance with the specified path.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified path.
     *
     * The path can either be empty or absolute (starting with a slash) or
     * rootless (not starting with a slash). Implementations MUST support all
     * three syntaxes.
     *
     * If an HTTP path is intended to be host-relative rather than path-relative
     * then it must begin with a slash ("/"). HTTP paths not starting with a slash
     * are assumed to be relative to some base path known to the application or
     * consumer.
     *
     * Users can provide both encoded and decoded path characters.
     * Implementations ensure the correct encoding as outlined in getPath().
     *
     * @param string $path The path to use with the new instance.
     * @return static A new instance with the specified path.
     * @throws \InvalidArgumentException for invalid paths.
     */
    public function withPath($path);

    /**
     * Return an instance with the specified query string.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified query string.
     *
     * Users can provide both encoded and decoded query characters.
     * Implementations ensure the correct encoding as outlined in getQuery().
     *
     * An empty query string value is equivalent to removing the query string.
     *
     * @param string $query The query string to use with the new instance.
     * @return static A new instance with the specified query string.
     * @throws \InvalidArgumentException for invalid query strings.
     */
    public function withQuery($query);

    /**
     * Return an instance with the specified URI fragment.
     *
     * This method MUST retain the state of the current instance, and return
     * an instance that contains the specified URI fragment.
     *
     * Users can provide both encoded and decoded fragment characters.
     * Implementations ensure the correct encoding as outlined in getFragment().
     *
     * An empty fragment value is equivalent to removing the fragment.
     *
     * @param string $fragment The fragment to use with the new instance.
     * @return static A new instance with the specified fragment.
     */
    public function withFragment($fragment);

    /**
     * Return the string representation as a URI reference.
     *
     * Depending on which components of the URI are present, the resulting
     * string is either a full URI or relative reference according to RFC 3986,
     * Section 4.1. The method concatenates the various components of the URI,
     * using the appropriate delimiters:
     *
     * - If a scheme is present, it MUST be suffixed by ":".
     * - If an authority is present, it MUST be prefixed by "//".
     * - The path can be concatenated without delimiters. But there are two
     *   cases where the path has to be adjusted to make the URI reference
     *   valid as PHP does not allow to throw an exception in __toString():
     *     - If the path is rootless and an authority is present, the path MUST
     *       be prefixed by "/".
     *     - If the path is starting with more than one "/" and no authority is
     *       present, the starting slashes MUST be reduced to one.
     * - If a query is present, it MUST be prefixed by "?".
     * - If a fragment is present, it MUST be prefixed by "#".
     *
     * @see http://tools.ietf.org/html/rfc3986#section-4.1
     * @return string
     */
    public function __toString();
}
```

### 3.6 `Psr\Http\Message\UploadedFileInterface`

```php
<?php
namespace Psr\Http\Message;

/**
 * Value object representing a file uploaded through an HTTP request.
 *
 * Instances of this interface are considered immutable; all methods that
 * might change state MUST be implemented such that they retain the internal
 * state of the current instance and return an instance that contains the
 * changed state.
 */
interface UploadedFileInterface
{
    /**
     * Retrieve a stream representing the uploaded file.
     *
     * This method MUST return a StreamInterface instance, representing the
     * uploaded file. The purpose of this method is to allow utilizing native PHP
     * stream functionality to manipulate the file upload, such as
     * stream_copy_to_stream() (though the result will need to be decorated in a
     * native PHP stream wrapper to work with such functions).
     *
     * If the moveTo() method has been called previously, this method MUST raise
     * an exception.
     *
     * @return StreamInterface Stream representation of the uploaded file.
     * @throws \RuntimeException in cases when no stream is available.
     * @throws \RuntimeException in cases when no stream can be created.
     */
    public function getStream();

    /**
     * Move the uploaded file to a new location.
     *
     * Use this method as an alternative to move_uploaded_file(). This method is
     * guaranteed to work in both SAPI and non-SAPI environments.
     * Implementations must determine which environment they are in, and use the
     * appropriate method (move_uploaded_file(), rename(), or a stream
     * operation) to perform the operation.
     *
     * $targetPath may be an absolute path, or a relative path. If it is a
     * relative path, resolution should be the same as used by PHP's rename()
     * function.
     *
     * The original file or stream MUST be removed on completion.
     *
     * If this method is called more than once, any subsequent calls MUST raise
     * an exception.
     *
     * When used in an SAPI environment where $_FILES is populated, when writing
     * files via moveTo(), is_uploaded_file() and move_uploaded_file() SHOULD be
     * used to ensure permissions and upload status are verified correctly.
     *
     * If you wish to move to a stream, use getStream(), as SAPI operations
     * cannot guarantee writing to stream destinations.
     *
     * @see http://php.net/is_uploaded_file
     * @see http://php.net/move_uploaded_file
     * @param string $targetPath Path to which to move the uploaded file.
     * @throws \InvalidArgumentException if the $targetPath specified is invalid.
     * @throws \RuntimeException on any error during the move operation.
     * @throws \RuntimeException on the second or subsequent call to the method.
     */
    public function moveTo($targetPath);

    /**
     * Retrieve the file size.
     *
     * Implementations SHOULD return the value stored in the "size" key of
     * the file in the $_FILES array if available, as PHP calculates this based
     * on the actual size transmitted.
     *
     * @return int|null The file size in bytes or null if unknown.
     */
    public function getSize();

    /**
     * Retrieve the error associated with the uploaded file.
     *
     * The return value MUST be one of PHP's UPLOAD_ERR_XXX constants.
     *
     * If the file was uploaded successfully, this method MUST return
     * UPLOAD_ERR_OK.
     *
     * Implementations SHOULD return the value stored in the "error" key of
     * the file in the $_FILES array.
     *
     * @see http://php.net/manual/en/features.file-upload.errors.php
     * @return int One of PHP's UPLOAD_ERR_XXX constants.
     */
    public function getError();

    /**
     * Retrieve the filename sent by the client.
     *
     * Do not trust the value returned by this method. A client could send
     * a malicious filename with the intention to corrupt or hack your
     * application.
     *
     * Implementations SHOULD return the value stored in the "name" key of
     * the file in the $_FILES array.
     *
     * @return string|null The filename sent by the client or null if none
     *     was provided.
     */
    public function getClientFilename();

    /**
     * Retrieve the media type sent by the client.
     *
     * Do not trust the value returned by this method. A client could send
     * a malicious media type with the intention to corrupt or hack your
     * application.
     *
     * Implementations SHOULD return the value stored in the "type" key of
     * the file in the $_FILES array.
     *
     * @return string|null The media type sent by the client or null if none
     *     was provided.
     */
    public function getClientMediaType();
}
```

Начиная с [psr / http-message версии 1.1](https://packagist.org/packages/psr/http-message#1.1.0), вышеупомянутые интерфейсы были обновлены для добавления объявлений типов аргументов. Начиная с [psr / http-message версии 2.0](https://packagist.org/packages/psr/http-message#2.0.0), вышеупомянутые интерфейсы были обновлены для добавления объявлений возвращаемого типа.
