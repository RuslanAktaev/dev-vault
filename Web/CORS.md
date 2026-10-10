---
tags: [web]
related: ["[[HTTP-заголовки]]", "[[Cookies]]", "[[Отладка веб-запросов]]"]
---
# CORS

**Определение.** CORS (Cross-Origin Resource Sharing) — способ, которым сервер сообщает браузеру: «страницам с этого origin можно читать мои ответы». CORS защищает пользователя в браузере, а не сервер, и настраивается только на стороне API.

## Same-Origin Policy
По умолчанию скрипт со страницы `a.com` не может прочитать ответ от `b.com`. При этом **сам запрос уходит**, браузер только не отдаёт результат странице. Отсюда:
- ошибка CORS значит «в ответе нет разрешающих заголовков», а не «запрос не дошёл»;
- curl, Postman и серверы CORS игнорируют, это правило браузера.

Origin — это схема + хост + порт. `app.example.com` и `api.example.com` — разные origin, хотя и один site, поэтому между ними CORS нужен.

## Простые запросы и preflight
- **Простые запросы** уходят сразу. Это GET, HEAD и POST с телом `text/plain`, `application/x-www-form-urlencoded` или `multipart/form-data`. Но если в ответе нет нужных заголовков, JS его не прочитает.
- **«Непростые»** — JSON в теле, заголовок `Authorization`, методы PUT/DELETE/PATCH. Перед ними браузер сам отправляет `OPTIONS` с вопросом «а можно?» и только после разрешения шлёт настоящий запрос. Поэтому в Network видно два запроса.

### Что несёт preflight
Это обычный HTTP-запрос `OPTIONS`. В нём только то, что серверу нужно для решения. Данных пользователя там нет.

```http
OPTIONS /api/orders
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type, authorization
```

| Что | Есть ли в preflight |
|---|---|
| `Origin` | да, тот же |
| Метод основного запроса | в `Access-Control-Request-Method` |
| Заголовки основного запроса | только **имена** в `Access-Control-Request-Headers`, без значений |
| Куки, `Authorization`, тело | нет |

Решение принимает браузер по заголовкам ответа. Preflight проваливается, если:
- разрешающих заголовков нет;
- они не покрывают нужный origin, метод или заголовки;
- статус не 2xx: нет обработчика `OPTIONS` или на нём висит авторизация.

Тогда **основной запрос не уходит вообще**. В Network виден только `OPTIONS`, а в логах сервера preflight может выглядеть как успешный `204`. Поэтому обработчик CORS должен стоять **до** проверки авторизации.

## Ответ сервера

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 600
Vary: Origin
```

| Параметр | Заголовок | Что значит |
|---|---|---|
| Origin | `Access-Control-Allow-Origin` | Какие сайты могут читать ответы. С куками — только конкретный origin, не `*` |
| Методы | `Access-Control-Allow-Methods` | Какие методы разрешены |
| Заголовки запроса | `Access-Control-Allow-Headers` | Какие заголовки можно слать (`Content-Type`, `Authorization`) |
| Куки | `Access-Control-Allow-Credentials: true` | Нужен, если фронт шлёт `credentials: 'include'` |
| Кеш preflight | `Access-Control-Max-Age` | На сколько секунд браузер запомнит ответ на `OPTIONS` |
| Заголовки ответа | `Access-Control-Expose-Headers` | Какие нестандартные заголовки ответа увидит JS |

## Как настроить

**Express**, пакет `cors`. Middleware сам отвечает на `OPTIONS`. Подключать его нужно до роутов и до авторизации. Для отдельного роута: `app.options('/products/:id', cors())`.

```js
import cors from 'cors';

app.use(cors({
  origin: ['https://app.example.com'],   // строка, массив, RegExp или функция
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,                     // если фронт шлёт куки
  maxAge: 3600,                          // кеш preflight, секунды
}));
```

**Next.js** (App Router, v16) — три места:
- **Один Route Handler**: заголовки `Access-Control-*` в `new Response(..., { headers })`. Если `OPTIONS` не объявлен, Next.js отвечает на него сам, но разрешающих CORS-заголовков не ставит. Поэтому для preflight с кастомными заголовками нужно экспортировать свой `OPTIONS`.
- **Несколько роутов**: `headers()` в `next.config` с `source: '/api/:path*'`. Origin при этом статический.
- **Несколько разрешённых origin**: `proxy` (раньше назывался middleware). Берёшь `Origin` из запроса, сверяешь со списком `allowedOrigins` и подставляешь его же в `Access-Control-Allow-Origin`. На `OPTIONS` отвечаешь заголовками, не вызывая остальное приложение.

**Laravel**: ничего писать не нужно. Middleware `HandleCors` уже стоит в глобальном стеке и сам отвечает на `OPTIONS`. Настройки лежат в `config/cors.php`, файл создаётся командой `php artisan config:publish cors`. Названия параметров смотри в нём.

**Для разработки.** Прокси dev-сервера (Vite, Next rewrites) делает так, что браузер ходит на тот же origin, и CORS исчезает. В проде проще держать фронт и API под одним доменом за reverse proxy.

## Что чаще всего ломается
- **`Allow-Origin: *` вместе с куками.** При `credentials: 'include'` браузер требует конкретный origin.
- **Preflight закрыт авторизацией.** `OPTIONS` приходит без токена и без куки.
- **Сервер упал с 500, а в консоли «CORS error».** У ответа с ошибкой нет CORS-заголовков. Реальный статус смотри в Network.
- **Origin не совпадает посимвольно**: `https://app.example.com/` со слэшем или `http` вместо `https`.
- **Origin подставляется динамически без `Vary: Origin`.** Общий кеш (CDN) отдаст одному origin ответ, предназначенный другому. Пакет `cors` добавляет этот заголовок сам.
- **Прокси (nginx, CDN) сам ставит CORS-заголовки или не пропускает `OPTIONS`.** Тогда настройки приложения не действуют.
- **JS не видит нестандартный заголовок ответа.** Нужен `Access-Control-Expose-Headers`.
- **Антипаттерн: отражать любой `Origin` назад вместе с credentials.** Это фактически отключает защиту.

## Частые ошибки понимания
- **«CORS защищает сервер».** Нет. Он защищает пользователя: чужая страница не прочитает ответ, подписанный его куками. Запрос до сервера всё равно доходит.
- **«Настрою CORS на фронте».** Его настраивают только на API.
- **«CORS error — запрос не дошёл».** Простой запрос дошёл и выполнился. Не дошёл только основной запрос после проваленного preflight.
- **«Один site — значит CORS не нужен».** CORS работает по origin, а не по site.

## Связи
- [[HTTP-заголовки]] — `Origin`, `Access-Control-*`, `Vary`. От `Content-Type` и `Authorization` зависит, пойдёт ли preflight.
- [[Cookies]] — чтобы кука ушла на другой origin, нужны `credentials: 'include'` и `Allow-Credentials: true` с конкретным origin. Origin против site.
- [[Отладка веб-запросов]] — симптом «CORS error»: где смотреть реальный статус и preflight.
