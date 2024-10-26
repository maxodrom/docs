# Лучшие практики безопасности

Ниже мы рассмотрим общие принципы безопасности и опишем, как избежать угроз при разработке приложений с использованием Yii. Большинство из этих принципов не являются уникальными только для Yii, но применимы к разработке веб-сайтов или программного обеспечения в целом, так что вы также найдете ссылки для дальнейшего чтения об общих идеях, лежащих в их основе.

## Основные принципы

Независимо от того, какое приложение разрабатывается, существуют два основных принципа обеспечения безопасности:

1. Фильтрация ввода.
2. Экранирование вывода.

### Фильтрация ввода

Фильтрация ввода означает, что входные данные никогда не должны считаться безопасными и вы всегда должны проверять, являются ли полученные данные допустимыми. 
Например, если мы знаем, что сортировка может быть осуществлена только по трём полям `title`, `created_at` и `status`, и поле может передаваться через ввод пользователем, лучше проверить значение там, где мы его получили.
С точки зрения чистого PHP, это будет выглядеть следующим образом:

```php
$sortBy = $_GET['sort'];
if (!in_array($sortBy, ['title', 'created_at', 'status'])) {
	throw new \InvalidArgumentException('Invalid sort value.');
}
```

В Yii, вы, скорее всего, будете использовать [валидацию форм](../input/validation.md), чтобы делать такие проверки.

Дополнительная информация по теме:

- <https://owasp.org/www-community/vulnerabilities/Improper_Data_Validation>
- <https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html>

### Экранирование вывода

Экранирование вывода означает, что в зависимости от контекста, в котором вы используете данные, вам следует добавить к ними специальные символы, чтобы экранировать их значение.
В контексте HTML вы должны экранировать `<`, `>` и похожие специальные символы.
В контексте JavaScript или SQL это будет другой набор символов.
Так как ручное экранирование чревато ошибками, Yii предоставляет различные утилиты для экранирования в различных контекстах.

Дополнительная информация по теме:

- <https://owasp.org/www-community/attacks/Command_Injection>
- <https://owasp.org/www-community/attacks/Code_Injection>
- <https://owasp.org/www-community/attacks/xss/>

## Как избежать SQL-инъекций

SQL-инъекции происходят, когда текст запроса формируется склеиванием неэкранированных строк, как показано ниже:

```php
$username = $_GET['username'];
$sql = "SELECT * FROM user WHERE username = '$username'";
```

Вместо того, чтобы подставлять корректное имя пользователя, злоумышленник может передать в ваше приложение что-то вроде `'; DROP TABLE user; --`. В результате SQL будет следующий:

```sql
SELECT * FROM user WHERE username = ''; DROP TABLE user; --'
```

Это валидный запрос, который сначала будет искать пользователей с пустым именем, а затем удалит таблицу user. Скорее всего будет сломано приложение и будут потеряны данные (вы ведь делаете регулярное резервное копирование?).

Убедитесь, что либо вы напрямую используете подготовленные PDO запросы, либо это делает выбранная вами библиотека. 
В случае подготовленных запросов невозможно манипулированть запросом, как было продемонстрировано выше.

Если вы используете данные для указания имен столбцов или таблиц, лучше всего разрешить только предопределенный набор значений:

```php
function actionList($orderBy = null)
{
    if (!in_array($orderBy, ['name', 'status'])) {
        throw new \InvalidArgumentException('Only name and status are allowed to order by.');
    }
    
    // ...
}
```

Дополнительная информация по теме:

- <https://owasp.org/www-community/attacks/SQL_Injection>

## Как избежать XSS

XSS или кросс-сайтинговый скриптинг становится возможен, когда неэкранированный выходной HTML попадает в браузер. 
Например, если пользователь должен ввести своё имя, но вместо `Alexander` он вводит `<script>alert('Hello!');</script>` то все страницы, которые его выводят без экранирования, будут выполнять JavaScript `alert('Hello!');`, и в результате будет выводиться окно сообщения в браузере.
В зависимости от сайта, вместо невинных скриптов с выводом всплывающего hello, злоумышленниками могут быть отправлены скрипты, похищающие личные данные пользователей сайта, либо выполняющие операции от их имени (например, банковские операции).

В Yii избежать XSS легко. Существует два варианта:

1. Вы хотите вывести данные в виде обычного текста.
2. Вы хотите вывести данные в виде HTML.

Если вам нужно вывести простой текст, то экранировать лучше следующим образом:

```php
<?= \Yiisoft\Html\Html::encode($username) ?>
```

Если нужно вывести HTML, вам лучше воспользоваться [HtmlPurifier](http://htmlpurifier.org/).
Обратите внимание, что обработка с помощью HtmlPurifier довольно тяжела, поэтому рассмотрите возможность использования кеширования.

Дополнительная информация по теме:

- <https://owasp.org/www-community/attacks/xss/>

## Как избежать CSRF

CSRF - это аббревиатура для межсайтинговой подмены запросов. Идея заключается в том, что многие приложения предполагают, что запросы, приходящие от браузера, отправляются самим пользователем. Это может быть неправдой.

Например, сайт `an.example.com` имеет URL `/logout`, который, используя простой GET, разлогинивает пользователя. Пока это запрос выполняется самим пользователем - всё в порядке, но в один прекрасный день злоумышленники размещают код '<img src="https://an.example.com/logout">' на форуме с большой посещаемостью. Браузер не делает никаких отличий между запросом изображения и запросом страницы, так что когда пользователь откроет страницу с таким тегом `<img>`, браузер отправит GET-запрос на указанный адрес, и пользователь будет разлогинен с `an.example.com`.
 
Вот основная идея того, как работает CSRF-атака. 
Можно сказать, что в разлогинивании пользователя нет ничего серьёзного. 
Однако, это был всего лишь пример.

С помощью этого подхода можно сделать гораздо больше опасных вещей.
Например, оплату или изменение данных.
Представьте, что существует страница `http://an.example.com/purse/transfer?to=anotherUser&amount=2000`, обращение к которой с помощью GET-запроса, приводит к перечислению 2000 единиц валюты со счета авторизованного пользователя на счет пользователя с логином `anotherUser`. 
Учитывая, что браузер для загрузки контента отправляет GET-запросы, можно подумать, что разрешение на выполнение такой операции только POST-запросом на 100% обезопасит от проблем. 
К сожалению, это не спасет вас, так как вместо тега `<img>`, злоумышленник может внедрить JavaScript код, который будет отправлять нужные POST-запросы на этот URL.

По этой причине Yii применяет дополнительные механизмы защиты от CSRF-атак.

Для того, чтоб избежать CSRF вы должны всегда:

1. Следовать спецификации HTTP. Например, GET-запрос не должен менять состояние приложения.
   Дополнительные сведения см. в [RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html)
2. Держите защиту CSRF в Yii включенной.

Yii имеет защиту от CSRF в middleware `Yiisoft\Yii\Web\Middleware\Csrf`.
Убедитесь, что он используется в вашем приложении.

Дополнительная информация по теме:

- <https://owasp.org/www-community/attacks/csrf>
- <https://owasp.org/www-community/SameSite>

## Как избежать нежелательного доступа к файлам

По умолчанию, webroot сервера указывает на каталог `public`, где лежит `index.php`. 
В случае использования виртуального хостинга, это может быть недостижимо, в конечном итоге весь код, конфиги и логи могут оказаться в webroot сервера.

Если это так, то нужно запретить доступ ко всему, кроме директории `web`. 
Если на вашем хостинге такое невозможно, рассмотрите возможность смены хостинга.

## Как избежать вывода отладочной информации и инструментов в боевом окружении

В режиме отладки, Yii отображает довольно подробные ошибки, которые полезны во время разработки. Однако, подробные ошибки удобны и для нападающего, так как могут раскрыть структуру базы данных, параметров конфигурации и части вашего кода.

Вы никогда не должны оставлять Debug панель или Gii доступной для всех в боевом окружении. Это может быть использовано для получения информации о структуре базы данных или коде, может позволить заменить файлы, генерируемые Gii автоматически.

Следует избегать включения в боевом окружении панели отладки, если только в этом нет острой необходимости. Она раскрывает всё приложение и детали конфигурации. Если вам всё-таки нужно запустить панель отладки, проверьте дважды, что доступ ограничен только вашими IP-адресами.

Дополнительная информация по теме:

- <https://owasp.org/www-project-.net/articles/Exception_Handling.md>
- <https://owasp.org/www-pdf-archive/OWASP_Top_10_2007.pdf>

## Использование безопасного подключения через TLS

Yii предоставляет функции, которые зависят от куки-файлов и/или сессий PHP. 
Они могут быть уязвимыми, если ваше соединение скомпрометировано.

Риск снижается, если приложение использует безопасное соединение через TLS (часто называемое как [SSL](https://en.wikipedia.org/wiki/Transport_Layer_Security)).

Сегодня любой желающий может бесплатно получить SSL-сертификат и автоматически обновлять его благодаря [Let's Encrypt](https://letsencrypt.org/).

## Безопасная конфигурация сервера

Цель этого раздела - выявить риски, которые необходимо учитывать при создании конфигурации сервера для обслуживания веб-сайта на основе Yii.
Помимо перечисленных здесь пунктов есть и другие параметры, связанные с безопасностью, которые необходимо учитывать, поэтому не рассматривайте этот раздел как завершенный.

### Как избежать атаки типа `Host`-header

Если веб-сервер настроен на обслуживание одного и того же сайта независимо от значения заголовка `Host`, эта информация может быть ненадежной и [может быть подделана пользователем, отправляющим HTTP-запрос](https://www.acunetix.com/vulnerabilities/web/host-header-attack). 
В таких ситуациях вам следует исправить конфигурацию вашего веб-сервера, чтобы он обслуживал сайт только для указанных имен хостов.

Дополнительные сведения о конфигурации сервера смотрите в документации вашего веб-сервера:

- Apache 2: <https://httpd.apache.org/docs/trunk/vhosts/examples.html#defaultallports>
- Nginx: <https://www.nginx.com/resources/wiki/start/topics/examples/server_blocks/>

### Настройка проверки SSL-сертификата

Существует типичное заблуждение о том, как решить проблемы с проверкой сертификата SSL, например:

```
cURL error 60: SSL certificate problem: unable to get local issuer certificate
```

или

```
stream_socket_enable_crypto(): SSL operation failed with code 1. OpenSSL Error messages: error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed
```

Многие источники ошибочно предлагают отключить проверку одноранговых соединений SSL.
Этого никогда не следует делать, поскольку это допускает атаки типа «man-in-the-middle».
Вместо этого, PHP должен быть правильно настроен:

1. Скачайте файл [https://curl.haxx.se/ca/cacert.pem](https://curl.haxx.se/ca/cacert.pem).
2. Добавьте в свой php.ini следующее:
  ```
  openssl.cafile="/path/to/cacert.pem"
  curl.cainfo="/path/to/cacert.pem".
  ```

Обратите внимание, что вам следует поддерживать файл в актуальном состоянии.

## Ссылки

- [OWASP top 10](https://owasp.org/Top10/)
- [The Basics of Web Application Security](https://martinfowler.com/articles/web-security-basics.html) Мартина Фаулера
- [Руководство по PHP: безопасность](https://www.php.net/manual/en/security.php)
- [Раздел "Информационная безопасность" на STackExchange](https://security.stackexchange.com/)
