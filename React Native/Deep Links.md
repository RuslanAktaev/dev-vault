---
tags: [react-native]
related: ["[[Expo Router]]", "[[Routing]]", "[[Signing and Credentials]]"]
---
# Deep Links

Диплинк — URL снаружи приложения (ссылка, пуш, QR, OAuth-редирект), который роутер сопоставляет с экраном. В Expo Router — с файлами в `app/`, как при `router.navigate`.

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
На нажатие ссылки сначала реагирует iOS, а не браузер. `https` по умолчанию уходит в Safari, но если домен связан с установленным приложением, iOS отдаёт ссылку приложению, браузер даже не запускается. Связь подтверждают **две стороны**.

**1. Приложение: «хочу открывать ссылки example.com».**
```json
"ios": { "associatedDomains": ["applinks:example.com"] }
```
- `applinks` — цель связи, домен **без `https://`**. Другие цели: `webcredentials` (автозаполнение паролей), `activitycontinuation` (Handoff).
- Запись попадает в **entitlements** (см. ниже). У App ID должна быть capability Associated Domains, EAS Build включает её сам.

**2. Сайт: «да, это приложение моё».** Файл `https://example.com/.well-known/apple-app-site-association`, без расширения:
```json
{
  "applinks": {
    "details": [{
      "appIDs": ["ABCDE12345.com.example.app"],
      "components": [
        { "/": "/admin/*", "exclude": true },
        { "/": "/post/*" }
      ]
    }]
  }
}
```
- `ABCDE12345` — Team ID (Apple Developer → Membership).
- `components` проверяются сверху вниз, первое совпадение побеждает → `exclude` выше. `*` не переходит через `/`.
- Хостинг: только HTTPS, **без редиректов**, `200` с `Content-Type: application/json`. С Expo Router — в `public/.well-known/`, деплоится с веб-версией.

**Как iPhone их сводит:**
1. При установке и обновлении iOS видит `applinks:example.com` в entitlements.
2. Забирает файл **через CDN Apple** (`app-site-association.cdn-apple.com/a/v1/example.com`), а не напрямую с сайта.
3. Если приложение есть в файле, запоминает: `example.com/post/*` → это приложение.
4. При нажатии отдаёт URL приложению, Expo Router открывает `post/[id].tsx`. Сеть в этот момент не нужна.
5. Нет приложения → нет связи → Safari.

**Android App Links:** `intentFilters` с `autoVerify: true` в `app.json` и файл `/.well-known/assetlinks.json` на сайте.

### Entitlements
Приложение в iOS сидит в песочнице, всё «особое» объявляется заранее в plist `.entitlements`, который Apple подписывает вместе с приложением.

| Возможность | Entitlement | Кто добавляет в Expo |
|---|---|---|
| Universal links | `com.apple.developer.associated-domains` | `ios.associatedDomains` |
| Пуши | `aps-environment` | `expo-notifications` |
| Sign in with Apple | `com.apple.developer.applesignin` | `expo-apple-authentication` |
| Общие данные между своими приложениями | `keychain-access-groups`, `com.apple.security.application-groups` | конфиг-плагины |

Работает только **связка из трёх частей**: capability у App ID, provisioning profile с этой capability и entitlements в сборке. При подписи Apple проверяет, что запрошенное разрешено, поэтому «дописать себе» право нельзя. В Expo файл генерирует `prebuild`, capability и профиль обновляет EAS Build. На Android аналога с подписью нет: всё в `AndroidManifest.xml`.

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
