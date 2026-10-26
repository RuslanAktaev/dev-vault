---
tags: [react-native]
related: ["[[Expo Router]]", "[[Routing]]", "[[Signing and Credentials]]"]
---
# Deep Links

Диплинк — URL снаружи приложения (ссылка, пуш, QR, OAuth-редирект), который роутер сопоставляет с экраном. В Expo Router — с файлами в `app/`, как при `router.navigate`.

## Шпаргалка: что настраивает RN/Web-разработчик

| Где | Что | Зачем |
|---|---|---|
| `app.json` → `scheme` | `"myapp"` | custom scheme `myapp://…`: OAuth, пуши, dev |
| `app.json` → `ios.associatedDomains` | `["applinks:example.com"]` | universal links, сторона приложения |
| `app.json` → `android.intentFilters` | `VIEW` + `https` + `host` + `autoVerify: true` | App Links, сторона приложения |
| сайт → `/.well-known/apple-app-site-association` | `appIDs` (Team ID + bundle ID) + `components` (пути) | universal links, сторона сайта |
| сайт → `/.well-known/assetlinks.json` | `package_name` + `sha256_cert_fingerprints` | App Links, сторона сайта |

- Значения для файлов на сайте: `ios.bundleIdentifier` + Team ID; `android.package` + SHA-256 **app signing key** из Play Console (не upload key).
- Поля `app.json` — нативный конфиг: после изменения нужна **новая сборка**, `eas update` не привезёт.
- Файлы на сайте: HTTPS, без редиректов, `application/json`, на **каждом** хосте (`www.` — отдельный).
- Роутер: `anchor` (кнопка «назад» у экрана из ссылки), `+not-found`, `+native-intent.tsx` (переписать чужие URL / allowlist), валидация параметров.
- Entitlements, provisioning profile, проверку файлов системой делают EAS и ОС — это нужно для отладки, а не для настройки.

## Custom scheme или https-ссылка

| | Custom scheme | Universal Links (iOS) / App Links (Android) |
|---|---|---|
| Вид | `myapp://post/42` | `https://example.com/post/42` |
| Что нужно | `scheme` в `app.json` | домен + файл на нём + `associatedDomains` (iOS) / `intentFilters` с `autoVerify` (Android) |
| Приложение не установлено | ошибка или ничего | открывается сайт или лендинг «Скачай приложение» |
| Кликабельна в мессенджерах и почте | часто нет | да, с превью |
| Можно перехватить | да: схема не уникальна, реестра нет | нет: владение подтверждает домен |
| Для чего | OAuth-редиректы (с PKCE), пуши, dev | ссылки для людей: шаринг, письма, реклама, QR |

- **Схема — не bundle ID.** `scheme` — префикс URL, `bundleIdentifier` — ID приложения в системе и сторе. Без `scheme` Expo берёт bundle ID как схему. Схему может занять любое приложение: на iOS не определено, какое откроется, на Android — диалог выбора.
- **Это нативный конфиг.** `scheme`, `associatedDomains`, `intentFilters` при сборке попадают в `Info.plist`, `.entitlements`, `AndroidManifest.xml`. Система читает их при установке → **через `eas update` не доедет, нужна новая сборка**. Экраны и `+native-intent.tsx` — JS, их OTA привозит.
- При `runtimeVersion` с политикой `fingerprint` EAS заметит изменение нативного конфига и не отправит несовместимый OTA-апдейт на старые сборки.

## Universal links по шагам

### На пальцах
Сайт `pizza.ru`, приложение «Пицца». Тебе прислали `https://pizza.ru/order/42`, и она должна открыться в приложении, а не в браузере.

- **Куда открыть ссылку, решает телефон** (iOS), а не браузер и не приложение.
- **Сам телефон не знает**, что `pizza.ru` — это «Пицца». Кто-то должен ему сказать.
- **Верить только приложению нельзя.** Иначе приложение «Мошенник» заявило бы «я открываю ссылки `sberbank.ru`» и получало бы ссылки банка — со сбросом пароля, входом, оплатой.
- **Решать должен владелец сайта.** Спросить его можно одним способом: прочитать файл, который он положил на свой сайт по фиксированному адресу `/.well-known/apple-app-site-association`. Там написано: «мои ссылки открывает приложение „Пицца“». Мошенник положить файл на `sberbank.ru` не может — нет доступа к сайту.
- **Поэтому телефон и идёт на сайт — за подтверждением от владельца.** Не на все сайты, а только на те, что приложение назвало при установке.

По времени:
1. **Установка «Пиццы».** Телефон видит в приложении запись «хочу ссылки `pizza.ru`».
2. **Сразу же, один раз**, телефон скачивает файл с `pizza.ru` (через сервер Apple). Там «да, это „Пицца“» → телефон запоминает: «`pizza.ru` → „Пицца“».
3. **Нажатие на ссылку позже.** Телефон никуда не ходит: смотрит запись и открывает «Пиццу».

У «Мошенника» шаг 2 проваливается: в файле на `sberbank.ru` его нет → записи нет → ссылки банка открываются в браузере.

Аналогия: ты приходишь на почту за посылкой Иванова. Слов «я от Иванова» мало, нужна доверенность, подписанная самим Ивановым. Приложение — ты, сайт — Иванов, файл на сайте — доверенность.

### Сторона 1 — приложение
```json
"ios": { "associatedDomains": ["applinks:example.com"] }
```
- Формат `<сервис>:<домен>`. `applinks` — **фиксированное ключевое слово** Apple (сервис «открывать ссылки»), пишется ровно так. Пример здесь только домен, он указывается **без `https://`**. Другие сервисы: `webcredentials` (автозаполнение паролей), `activitycontinuation` (Handoff), `appclips` (App Clips).
- При сборке запись попадает в **entitlements** и вшивается в подпись приложения твоим сертификатом (см. ниже). Подделать эту сторону нельзя: изменить запись можно только пересобрав и переподписав приложение под твоим аккаунтом.

### Сторона 2 — сайт
Сайт отвечает приложению **обычным JSON-файлом по заранее известному адресу**:

```
https://example.com/.well-known/apple-app-site-association
```

Почему это доказывает владение: положить файл по этому адресу на `example.com` по HTTPS может **только тот, кто управляет сайтом** (и у кого валидный TLS-сертификат домена). Сам адрес фиксирован Apple — iOS не ищет файл, а просто идёт по этому URL. `/.well-known/` — стандартная папка для служебных файлов сайта (там же лежат, например, файлы проверки для SSL).

Содержимое файла — ответ «какие приложения и какие пути»:
```json
{
  "applinks": {                       // ключевое слово: раздел про universal links
    "details": [{
      "appIDs": ["ABCDE12345.com.example.app"],   // КАКОЕ приложение
      "components": [                             // КАКИЕ пути ему отдавать
        { "/": "/admin/*", "exclude": true },     // /admin — не отдавать, пусть в Safari
        { "/": "/post/*" }                        // /post/... — в приложение
      ]
    }]
  }
}
```

- **`appIDs`** — точный адрес приложения: `<Team ID>.<bundle ID>`.
  - `com.example.app` — bundle ID, его может взять себе кто угодно.
  - `ABCDE12345` — Team ID, ID твоего аккаунта разработчика (Apple Developer → Membership). Его не подделать: приложение с этим Team ID может подписать только твой аккаунт.
  - Вместе они однозначно указывают на **твоё** приложение, а не на клон с тем же bundle ID.
- **`components`** — какие пути открывать в приложении. Проверяются сверху вниз, первое совпадение побеждает, поэтому `exclude` ставят выше. `*` не переходит через `/`. Всё, что не совпало, остаётся в Safari.
- Можно перечислить несколько приложений (например, prod и staging) и разные пути для каждого.
- Ключ верхнего уровня `applinks` — то же ключевое слово, что в `associatedDomains`. В файле могут быть и другие сервисы (`webcredentials` и т.д.).

**Требования к хостингу** (иначе iOS молча игнорирует файл):
- только HTTPS с валидным сертификатом;
- **без редиректов** (даже `example.com` → `www.example.com`);
- ответ `200` и `Content-Type: application/json`, файл без расширения `.json`.

**Почему без расширения.** Имя задала Apple, iOS запрашивает ровно этот URL: `…/apple-app-site-association.json` — другой адрес, его не найдут. Исторически (iOS 8–9) файл был не JSON, а подписанный сертификатом сайта PKCS#7-контейнер, отсюда и имя без `.json`; подпись потом отменили, имя осталось. Следствие: сервер определяет `Content-Type` по расширению и отдаст этот файл как `octet-stream` / `text/plain`, поэтому `application/json` для этого пути прописывают в конфиге сервера вручную (nginx, Vercel, Netlify, S3). У Android файл `assetlinks.json` — с расширением, проблемы нет.

С Expo Router файл кладут в `public/.well-known/`, он деплоится вместе с веб-версией. Если сайт отдельный — его выкладывают туда те, кто делает сайт.

### Как iPhone их сводит
1. **Установка / обновление приложения.** iOS читает entitlements и видит `applinks:example.com` — «приложение претендует на этот домен».
2. **Проверка сайта.** iOS запрашивает файл, но не с сайта напрямую, а **через CDN Apple** (`app-site-association.cdn-apple.com/a/v1/example.com`): Apple сама периодически скачивает и кэширует файлы сайтов. Поэтому исправленный файл доходит до телефонов с задержкой.
3. **Сверка.** Если в `appIDs` есть Team ID + bundle ID этого приложения — связь подтверждена. iOS запоминает: «`example.com/post/*` → это приложение, `/admin/*` → нет».
4. **Нажатие на ссылку.** iOS сверяет URL с сохранёнными правилами (сеть уже не нужна) и отдаёт ссылку приложению. Приложение получает URL, Expo Router открывает `post/[id].tsx`.
5. **Нет приложения** или путь не совпал — ссылка открывается в Safari как обычная. Поэтому по тому же адресу на сайте должна быть нормальная страница.

### Что значит «отдаёт URL приложению»
Приложение не «слушает» ссылки и не лезет в Safari. Работает наоборот: **iOS сама запускает приложение и передаёт ему URL как входной параметр** — так же, как при тапе по иконке, только с данными.

1. iOS решает, что ссылка принадлежит приложению (шаг 4 выше).
2. Если приложение **закрыто** — iOS его запускает (холодный старт). Если свёрнуто — выводит на передний план (тёплый старт).
3. iOS вызывает у нативной части приложения (`AppDelegate`) системный метод и передаёт в него URL:
   - universal link приходит как `NSUserActivity` типа «просмотр веба» с полем `webpageURL` → метод `application(_:continue:restorationHandler:)`;
   - custom scheme (`myapp://…`) приходит в `application(_:open:options:)`.
4. В React Native эти вызовы ловит нативный модуль `Linking` и пробрасывает URL в JS: на холодном старте — через `Linking.getInitialURL()`, на тёплом — событием `url`. В Expo `AppDelegate` уже настроен, руками ничего писать не нужно.
5. Expo Router берёт этот URL, разбирает путь `/post/42` и делает навигацию, как `router.navigate('/post/42')`. На холодном старте строит стек из URL (тут важен `anchor`).

**Никаких разрешений у пользователя не спрашивают.** Это не runtime-разрешение вроде камеры или геолокации: приложение ничего не читает с устройства, оно просто получает один URL, который пользователь сам нажал. Отказаться пользователь может только поведением: долгое нажатие → «Открыть в Safari», и iOS это запомнит.

На Android то же самое: система запускает `Activity`, у которой в манифесте есть подходящий `intent-filter`, и кладёт URL в `Intent` (`ACTION_VIEW`, поле `data`). Дальше тот же `Linking` → Expo Router.

### Android App Links
Та же идея из двух сторон, но другие файлы, и **пути фильтрует приложение, а не сайт**.

**Сторона 1 — приложение: `intentFilters` в `app.json`.**
```json
"android": {
  "package": "com.example.app",
  "intentFilters": [
    {
      "action": "VIEW",
      "autoVerify": true,
      "data": [
        { "scheme": "https", "host": "example.com", "pathPrefix": "/post" },
        { "scheme": "https", "host": "www.example.com", "pathPrefix": "/post" }
      ],
      "category": ["BROWSABLE", "DEFAULT"]
    }
  ]
}
```
- `action: "VIEW"` — «открыть/показать URL». Этот intent система шлёт при нажатии на ссылку.
- `category`: `BROWSABLE` — ссылку можно открыть из браузера и других приложений; `DEFAULT` — фильтр отвечает на обычные неявные intent'ы. Нужны оба.
- `data` — какие URL ловить: `scheme`, `host` и путь одним из способов: `path` (точно), `pathPrefix` (начинается с), `pathPattern` (простой шаблон с `.*`). Без пути — весь домен.
- `autoVerify: true` — «проверь, что домен мой» (через `assetlinks.json`). Без него это просто фильтр ссылок, и при нажатии Android откроет браузер или спросит пользователя.

`npx expo prebuild` превращает это в `android/app/src/main/AndroidManifest.xml`, внутрь главной activity:
```xml
<activity android:name=".MainActivity" ...>
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <data android:scheme="https" android:host="example.com" android:pathPrefix="/post"/>
    <data android:scheme="https" android:host="www.example.com" android:pathPrefix="/post"/>
  </intent-filter>
  <!-- Custom scheme из "scheme": "myapp" Expo добавляет сам, без autoVerify: -->
  <intent-filter>
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <data android:scheme="myapp"/>
  </intent-filter>
</activity>
```

**Сторона 2 — сайт: `https://example.com/.well-known/assetlinks.json`.** Для **каждого** хоста из фильтра (`example.com` и `www.example.com` — это два разных хоста, файл нужен на обоих):
```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.app",
    "sha256_cert_fingerprints": [
      "14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"
    ]
  }
}]
```
- `relation: delegate_permission/common.handle_all_urls` — фиксированная строка: «разрешаю приложению открывать мои ссылки».
- `package_name` — package из `app.json` (`android.package`).
- `sha256_cert_fingerprints` — SHA-256 **сертификата, которым подписан APK на телефоне**. Это аналог Team ID: package name может занять кто угодно, а подписать APK твоим ключом — только ты. Можно указать несколько (ключ Google Play, ключ для внутренних сборок).
- **Путей в файле нет** — сайт делегирует весь домен. Какие пути открывать, решают `pathPrefix` / `path` в `intentFilters`. На iOS наоборот: пути задаются в AASA (`components`).
- Хостинг: HTTPS, без редиректов, `Content-Type: application/json`, файл с расширением `.json`.

**Где взять SHA-256:**
- Сборки из Google Play: Play Console → Test and release → App integrity → App signing → сертификат **App signing key** (там же готовый сниппет `assetlinks.json`). Не путать с upload key.
- Ключ, которым подписывает EAS (внутренние APK, ad hoc): `eas credentials` → Android → Keystore.
- Свой keystore: `keytool -list -v -keystore my.keystore -alias my-alias`.

**Как Android проверяет:**
1. При установке (и обновлении) система видит фильтр с `autoVerify` и сама скачивает `assetlinks.json` с каждого хоста — напрямую, без CDN-посредника.
2. Сверяет `package_name` и SHA-256 с сертификатом установленного APK.
3. Совпало → домен «verified»: нажатие на `https://example.com/post/42` сразу открывает приложение, без диалога выбора. Приложение получает `Intent` с `ACTION_VIEW` и URL в `data` → `Linking` → Expo Router.
4. Не совпало → на Android 12+ ссылка просто откроется в браузере.

**Проверить на устройстве (Android 12+):**
```bash
adb shell pm get-app-links com.example.app                    # статус по каждому хосту: verified / none / …
adb shell pm verify-app-links --re-verify com.example.app     # перепроверить после правки файла
adb shell am start -a android.intent.action.VIEW -d "https://example.com/post/42"
```
Проверить сам файл глазами Google: `https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://example.com&relation=delegate_permission/common.handle_all_urls`.

### Entitlements
**Entitlement — это не разрешение пользователя, а подписанная декларация «что этому приложению позволено делать в системе».** Пользователь её не видит и ничего не подтверждает.

| | Runtime-разрешение | Entitlement |
|---|---|---|
| Примеры | камера, гео, фото, микрофон | universal links, пуши, Sign in with Apple, App Groups |
| Кто разрешает | пользователь, диалогом во время работы | Apple, при подписи сборки |
| Где объявляется | `Info.plist` (`NSCameraUsageDescription`) | `.entitlements` + capability в App ID |
| Можно отозвать | да, в Настройках | нет, зашито в сборку |

**Зачем entitlements.** По сути это не «разрешение», а **заявка приложения на домены**. Она решает две задачи:
1. **Говорит iOS, какие сайты проверять.** iOS не обходит все сайты в поисках упоминания приложения: при установке она берёт список доменов из entitlements и скачивает AASA только с них. Нет записи — iOS даже не пойдёт на сайт, и все ссылки уйдут в Safari, как бы правильно ни лежал файл.
2. **Делает заявку достоверной** (ниже).

Приложение в iOS сидит в песочнице и по умолчанию не может ничего «особого». Для universal links iOS нужно знать, что приложение **действительно** претендует на домен, и этому заявлению можно доверять. Если бы заявление лежало в обычном файле приложения, его мог бы вписать кто угодно. Entitlements вшиваются в подпись приложения, а iOS при установке сверяет их с provisioning profile, который подписала Apple на основе галочек App ID. Поэтому на шаге 1 iOS доверяет записи `applinks:example.com`: её не подделать, не пересобрав приложение под твоим аккаунтом. Вторую половину доверия даёт файл на сайте.

| Возможность | Entitlement | Кто добавляет в Expo |
|---|---|---|
| Universal links | `com.apple.developer.associated-domains` | `ios.associatedDomains` |
| Пуши | `aps-environment` | `expo-notifications` |
| Sign in with Apple | `com.apple.developer.applesignin` | `expo-apple-authentication` |
| Общие данные между своими приложениями | `keychain-access-groups`, `com.apple.security.application-groups` | конфиг-плагины |

Работает только **связка из трёх частей**:

| Часть | Где | Что это | Кто подписывает |
|---|---|---|---|
| **App ID** | портал Apple Developer | bundle ID + галочки capabilities: «что приложению разрешено в принципе» | никто, это запись в реестре |
| **Provisioning profile** | файл внутри сборки | «приложению с этим App ID, от этой команды, с этим сертификатом разрешены такие entitlements» — собирается из галочек App ID | **Apple** |
| **Entitlements** | `.entitlements`, вшивается в подпись приложения | что **эта сборка** запрашивает (`applinks:example.com`, пуши…) | **ты**, сертификатом разработчика |

iOS при установке и запуске проверяет: **запрошенное (entitlements) ⊆ разрешённое (profile)**. Нет — не установит или не запустит. Поэтому «дописать себе» право нельзя: запросить можно что угодно, но разрешение выдаёт Apple.

Галочка Associated Domains в App ID разрешает заявлять домены вообще, без списка. Конкретный домен есть только в entitlements, а подтверждает его файл на сайте. Что физически лежит в `.ipa`, что чем подписано и что iOS проверяет при установке — в [[Signing and Credentials#Что на выходе: из чего состоит подписанный ipa]]. В Expo файл генерирует `prebuild`, capability и профиль обновляет EAS Build. На Android аналога с подписью нет: всё в `AndroidManifest.xml`.

### Отложенный диплинк
Человек пришёл на лендинг, скачал приложение, открыл — исходная ссылка потеряна, он на главной. Решения: Branch / AppsFlyer (их URL переписывают в `+native-intent.tsx`), на Android — Install Referrer из Google Play. iOS сама этого не умеет.

## Как проверить

| Где | Команда |
|---|---|
| Expo Go (своя схема, перед путём `/--/`) | `npx uri-scheme open "exp://127.0.0.1:8081/--/explore" --ios` |
| Dev build, iOS | `npx uri-scheme open myapp://explore --ios` или `xcrun simctl openurl booted myapp://explore` |
| Dev build, Android | `adb shell am start -a android.intent.action.VIEW -d "myapp://explore"` |

- Прогоняй дважды: **холодный старт** (приложение закрыто, стек строится из URL, важен `anchor`) и **тёплый** (ссылка работает как `navigate`).
- Universal link проверяют **нажатием** в Заметках или Сообщениях. Вставка в адресную строку Safari и ссылка с того же домена приложение не открывают — это нормально.
- В разработке CDN Apple обходится: `applinks:example.com?mode=developer` + Настройки → Разработчик → Associated Domains Development.

**Отладка universal links:**
- Что видит Apple: `https://app-site-association.cdn-apple.com/a/v1/example.com`. Связь перечитывается при установке и обновлении, исправленный файл доходит не сразу.
- На Mac: `sudo swcutil dl -d example.com`.
- Долгое нажатие показывает «Открыть в „App“» → связь есть.
- Ловушка: если пользователь в приложении нажал на домен в правом верхнем углу, iOS запоминает «открывать в Safari». Вернуть — долгое нажатие → «Открыть в приложении».
- Встроенные браузеры (Instagram, Telegram) часто открывают ссылку у себя.

## Все маршруты достижимы
В Expo Router **каждый файл в `app/` открывается по ссылке**, даже без кнопки на него: хватает `scheme`. В React Navigation наоборот: linking-конфиг перечислял пути вручную, и «нет в конфиге» значило «недостижим». Если внешние ссылки не нужны — allowlist в `+native-intent.tsx` (только натив):

```tsx
// app/+native-intent.tsx
const ALLOWED = ['/', '/explore'];

export function redirectSystemPath({ path }: { path: string; initial: boolean }) {
  try {
    const { pathname } = new URL(path, 'myapp://app');
    return ALLOWED.includes(pathname) ? path : '/';
  } catch {
    return '/'; // падать здесь нельзя
  }
}
```

Для служебных шагов сценария достаточно, чтобы экран не падал при прямом заходе: нет нужных параметров → `router.replace` на начало.

## Безопасность
Диплинк — **ввод от кого угодно**: ссылку может собрать любой сайт, письмо или приложение. `Protected` и allowlist управляют только навигацией на устройстве, данные защищает только сервер.

```tsx
// ❌ ссылка сама выполняет действие: myapp://transfer?to=attacker&amount=1000
useEffect(() => { api.transfer(to, amount); }, []);

// ❌ open redirect и чужая страница в WebView: myapp://login?next=https://evil.com
router.replace(next);
<WebView source={{ uri: params.url }} />

// ✅ ссылка только открывает экран, параметры проверены
const { id } = useLocalSearchParams<{ id: string }>();
if (!/^\d+$/.test(id)) return <Redirect href="/" />;

// ✅ next — только внутренний путь
const safeNext = next?.startsWith('/') && !next.startsWith('//') ? next : '/';
router.replace(safeNext);
```

Чек-лист:
- В custom-scheme ссылках нет секретов. Magic links и сброс пароля — через universal links с одноразовым короткоживущим токеном.
- OAuth только с PKCE (в `expo-auth-session` по умолчанию): перехваченный код бесполезен.
- Ссылка сама ничего не делает: «удалить», «оплатить» — только после действия пользователя (мобильный аналог CSRF).
- Параметры валидируются; текст из URL не показывается как «официальный».
- `next` / `redirect` — только внутренние пути (`/`, но не `//`) или allowlist.
- URL из параметров не попадает в WebView, особенно с мостом в натив.
- Закрытые экраны закрыты через `Protected` или `+native-intent`, а не отсутствием кнопок.
- Файлы в `/.well-known/` защищены так же, как продакшен; на Android включён `autoVerify`.
- URL с персональными данными не уходят в аналитику и крэш-репорты.

## Частые ошибки понимания

| Симптом | Вероятная причина |
|---|---|
| В Expo Go работает, в сборке нет (или наоборот) | разные URL: `exp://…/--/path` против `myapp://path` |
| Поменяли `scheme` или домен — ничего не изменилось | нативный конфиг, OTA его не привозит |
| После логина ссылка потерялась | `Protected` или редирект увёл на логин; ссылку нужно сохранить и открыть самому |
| На экране из ссылки нет «назад» | не задан `anchor` |
| Экран открывается дважды | ручной `Linking.addEventListener` + `router.push` поверх обработки Expo Router |
| Тап по пушу при убитом приложении открывает главную | URL не достают через `getLastNotificationResponse` из `expo-notifications` |
| iOS: поправили AASA, а ссылки всё ещё в Safari | редирект или неверный `appIDs`, кэш CDN Apple, или пользователь выбрал «открывать в Safari» |
| Android: локально App Links работают, из стора нет | в `assetlinks.json` SHA-256 upload-ключа вместо ключа Play App Signing; или один из хостов фильтра не прошёл проверку |
| Ссылка из Instagram / Telegram открывается в их браузере | встроенные браузеры часто не отдают ссылку системе |
| Ссылки Branch, AppsFlyer, Firebase Dynamic Links ведут на `+not-found` | их URL не совпадают с маршрутами → переписывать в `+native-intent.tsx`; FDL закрыты с августа 2025 |

- Думать, что custom scheme принадлежит тебе. Её может зарегистрировать любое приложение.
- Думать, что `Protected` или «нет кнопки» защищает данные. Код экрана лежит в бандле, проверка на клиенте.

## Связи
- [[Expo Router]] — сопоставляет входящий URL с файлами в `app/`; `anchor`, `Protected`, `+native-intent` и статические `redirects` определяют, что откроется.
- [[Routing]] — в голом React Navigation диплинки требуют ручного `linking`-конфига, и недостижимо всё, что в нём не перечислено.
- [[Signing and Credentials]] — universal links работают только через entitlement Associated Domains, capability в App ID и перевыпущенный profile; App Links на Android сверяются с SHA-256 ключа Play App Signing.
