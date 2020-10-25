# ChatApiWA-JavaGuide
Данный репозиторий хранит в себе пример по работе с API сайта chat-api.com на Java

# Пример создания WA бота с использованием API chat-api.com на Java
Для того, чтобы наш бот мог получать общаться с серверами chat-api.com необходимо, чтобы бот мог принять и обработать входящие данные. Для этого требуется поднять собственный сервер - Weebhook и указать его адрес в личном кабинете.

# Немного о том, для чего нужнен Webhook
Webhook решает проблему с задержкой на отклик входящих сообщений. Без него, нашему боту пришлось бы постоянно спрашивать у серверов chat-api.com об входящих данных. Делать переодические фиксированные во времени запросы к серверам. Тем самым имея некоторую задержку в отклике и нагруженность на сервера. 

Но, если мы укажем адрес Webhook сервера, то данная необходимость перестанет быть актуальной. Тогда сервера chat-api.com сами нам будут присылать уведомления о входящих изменениях, как только они появятся. А задача Webhook сервера их принять и правильно обработать, реализовывая логику бота.

# Какой функционал реализуем?
+ При отправке команды "chatid" бот должен отправить сообщение с текущим ID чата.
+ При отправке команды "file [format]" бот должен отправить заранее подготовленные файлы (pdf, jpg, doc, mp3).
+ При отправке команды "ogg", бот должен отправить файл с расширением "ogg" (Голосовое сообщение).
+ При отправке команды "geo", бот должен отправить гео-координаты (локацию).
+ При отправке команды "group" бот должен создать группу с собой и пользователем.
+ Реагировать приветственным сообщением с описанием команд на все входящие не "командные" сообщения (как аналог "меню")

# Поднимаем собственный сервер на Java с помощью фреймворка Spring Boot
Для упрощения создания заготовки нашего spring boot приложения будем использовать [spring initializr](https://start.spring.io/):
Выставляем необходимые настройки, выбираем нужную версию Java и жмём Generate
![](https://ia.wampi.ru/2020/10/25/2020-10-25_18-33-25.png)

Далее необходимо скачать сгенерированный проект и открыть в вашей IDE. Я буду использовать [Intellij IDEA](https://www.jetbrains.com/idea/)
Создадим папку "controller", в которой будем хранить наш контроллер по обработке входящих данных на наш сервер и необходимые классы для работы бота.
![](https://ia.wampi.ru/2020/10/25/2020-10-25_18-58-29.png)

Сhat-api.com присылает Json данные, которые нам необходимо будет распарсить и обработать.
Для того, чтобы узнать, что именно нам приходит в json, воспользуемся страницей [тестирование](https://app.chat-api.com/testing) в личном кабинете и перейдем во вкладку "Симуляция Weebhoka"
![](https://ia.wampi.ru/2020/10/25/2020-10-25_19-03-09.png)
JSON body - Данные, которые будут приходить к нам на сервер (Их структура). Для того, чтобы удобно с ним работать из Java мы будем десериализовывать его в объект и обращаться с данными, как со свойствами объекта.
Для нашего удобства воспользуемся [сервисом](http://www.jsonschema2pojo.org/) по автоматической генерации Json to Java class. Копируем JSON body
![](https://ia.wampi.ru/2020/10/25/2020-10-25_19-10-35.png)
Жмем Preview и копируем автоматически сгенерированные классы Java в наш проект. Для этого создадим файл jsonserializables.java и вставляем код в файл. 
![](https://ia.wampi.ru/2020/10/25/2020-10-25_19-15-38.png)

После того, как наш класс, описывающий JSON тело создан. Приступаем к реализации самого контроллера, который будет обрабатывать входящие данные. Для этого создадим новый файл "MessageController"
![](https://ia.wampi.ru/2020/10/25/2020-10-25_19-19-36.png)
И опишем в нём класс MessageController
```java
@RestController
@RequestMapping("webhook")
public class MessageController {
    @PostMapping
    public String AnswerWebhook(@RequestBody RequestWebhook hook) throws IOException {
        return  "ok";
    }
```
Аннотация **@RequestMapping("webhook")** отвечает за адрес, по которому будет обрабатываться наш контроллер. Например: "localhost:8080/webhook"

Аннотация **@PostMapping** означает, что функция AnswerWebhook будет обрабатывать Post запросы.

В тоже время, параметр **RequestWebhook** в параметрах функции - это наш десериализованный JSON класс, десериализацию которого берет на себя Spring Boot и мы получаем сразу готовый объект и можем с ним работать внутри самой функции. В ней мы опишем логику работы бота.

# Реализуем логику контроллера
```java
@RestController
@RequestMapping("webhook")
public class MessageController {
    @PostMapping
    public String AnswerWebhook(@RequestBody RequestWebhook hook) throws IOException {
        for (var message : hook.getMessages()) {
            if (message.getFromMe())
                continue;
            String option = message.getBody().split(" ")[0].toLowerCase();
            switch (option)
            {
                case "chatid":
                    ApiWA.sendChatId(message.getChatId());
                    break;
                case "file":
                    var texts = message.getBody().split(" ");
                    if (texts.length > 1)
                        ApiWA.sendFile(message.getChatId(), texts[1]);
                    break;
                case "ogg":
                    ApiWA.sendOgg(message.getChatId());
                    break;
                case "geo":
                    ApiWA.sendGeo(message.getChatId());
                    break;
                case "group":
                    ApiWA.createGroup(message.getAuthor());
                    break;
                default:
                    ApiWA.sendDefault(message.getChatId());
                    break;
            }
        }
        return  "ok";
    }
```

В цикле проходимся по всем пришедшим сообщениям и обрабатываем их в switch.
Проверка в начале цикла нужна для того, чтобы бот не зациклился сам на себя. В случае, если сообщение пришло от самого себя - то пропускаем его и не обрабатываем.
Далее записываем в перменную **option** пришедшую команду. Для этого в тело сообщения мы разбиваем методом split и приводим команду в нижний регистр, чтобы бот реагировал на команды независимо от регистра. 
В switch, в зависимости от пришедшей команды, мы вызываем необходимый метод из класса ApiWA, о реализации которого мы сейчас и поговорим.

# ApiWA
Данный класс будет реализовывать статические методы по обращению к [API](https://app.chat-api.com/docs).
Внутри класса опишем переменные, которые будут хранить наш токен, чтобы мы могли успешно пользоваться API, передавая эти данные. Их можно найти в [личном кабинете](https://app.chat-api.com/dashboard)
```java
public class ApiWA {
    private static String APIURL = "https://eu115.chat-api.com/instance123456/";
    private static String TOKEN = "1hi0xwfzaen1234";
}
```

После чего нам потребуется реализовать метод, который будет отправлять POST запрос.
```java
 public static CompletableFuture<Void> postJSON(URI uri,
                                            Map<String,String> map)
            throws IOException
    {
        ObjectMapper objectMapper = new ObjectMapper();
        String requestBody = objectMapper
                .writerWithDefaultPrettyPrinter()
                .writeValueAsString(map);

        HttpRequest request = HttpRequest.newBuilder(uri)
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(requestBody))
                .build();

        return HttpClient.newHttpClient()
                .sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApply(HttpResponse::statusCode)
                .thenAccept(System.out::println);
    }
```
Данный метод принимает в параметры ссылку URI, на которую необходимо сделать post запрос, а также словарь, который сериализуется в Json строку и передается на сервер. 

Для реализации каждого из методов смотрим [документацию](https://app.chat-api.com/docs) и повторяем запросы.

Так выглядит метод по отправке **chatid**
```java
    public static void sendChatId(String chat_id) throws IOException {
        URI uri = URI.create(APIURL + "sendMessage?token=" + TOKEN);
        Map<String, String> map = new HashMap<String, String>();
        map.put("body", "Your ID: " + chat_id);
        map.put("phone", chat_id);
        ApiWA.postJSON(uri, map);
    }
```
Формиуем ссылку, на которую нужно сделать запрос и словарь с параметрами. После чего обращаемся к методу по отправке post запроса, который мы реализовали выше.
Аналогичным способом реализованы остальные методы. Исходный код можно посмотреть на github.

# Отдельно поговорим о реализации метода по отправке файлов
```java
 public static void sendFile(String chat_id, String file_format) throws IOException {
        Map<String, String> formats= new HashMap<String, String>();
        formats.put("doc", Base64Help.getDOC());
        formats.put("jpeg", Base64Help.getJPEG());
        formats.put("pdf", Base64Help.getPDFtring());
        formats.put("mp3", Base64Help.getMP3String());

        if (formats.containsKey(file_format))
        {
            Map<String, String> map = new HashMap<String, String>();
            map.put("phone", chat_id);
            map.put("body", formats.get(file_format));
            map.put("filename", "ThisIsFile");
            map.put("caption", "ThisIsCaption");
            URI uri = URI.create(APIURL + "sendFile?token=" + TOKEN);
            ApiWA.postJSON(uri, map);
        }
        else
        {
            Map<String, String> map = new HashMap<String, String>();
            map.put("phone", chat_id);
            map.put("body", "File not found");
            URI uri = URI.create(APIURL + "sendMessage?token=" + TOKEN);
            ApiWA.postJSON(uri, map);
        }
    }
```
Данный метод хранит в себе словарь, в котором ключи - формат файлов, а значения - строка в формате Base64. Base64 строкa используется для передачи файлов. Для генерации строки можно воспользоваться [сервисом](https://app.chat-api.com/base64) на нашем сайте. Если файл нужного формата отсутствует в словаре, то мы отправляем сообщение о том, что файл не найден.

Мы описали класс Base64Help и методы по получению строки с файлом нужного формата. 
Сама строка хранится в txt файлах на сервере, а из кода мы просто считываем её из файла. Это необходимо, потому что Java не позволяет хранить такие длинные строки прямо в коде. Вы же можете генерировать строку Base64 автоматически, либо пользуясь сервисами. 

```java
public class Base64Help {
    static public String getPDFString() throws IOException {
        return new String(Files.readAllBytes(Paths.get("src/main/resources/pdf.txt")));
    }

    static  public String getMP3String() throws IOException {
        return new String(Files.readAllBytes(Paths.get("src/main/resources/mp3.txt")));
    }

    static public String getJPEG() throws IOException {
        return new String(Files.readAllBytes(Paths.get("src/main/resources/jpeg.txt")));
    }

    static public String getDOC() throws IOException {
        return new String(Files.readAllBytes(Paths.get("src/main/resources/doc.txt")));
    }

}
```
