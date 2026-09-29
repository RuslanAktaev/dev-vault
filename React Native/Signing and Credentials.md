---
tags: [react-native]
related: ["[[Deep Links]]"]
---
# Signing and Credentials

Сертификаты и ключи, без которых приложение нельзя собрать и опубликовать: Android, iOS и то, как этим управляет **EAS** (Expo Application Services).

## Android

Публиковать можно и как `.apk` напрямую, но обычно — через Google Play (нужен аккаунт в Play Console).

| Сущность | Что это | Кто создаёт и хранит | Где смотреть | Срок |
|---|---|---|---|---|
| **Upload key** | подписывает AAB перед отправкой в Play | разработчик или EAS (`eas credentials`), на приложение | EAS / локальный keystore | бессрочно, при утере сбрасывается через форму Google |
| **App signing key** | ключ, которым **Google на своей стороне** пересобирает и подписывает финальный APK для пользователей | целиком Google: генерируется сам при подключении Play App Signing | только отпечатки SHA-1/SHA-256: Play Console → Release → Setup → App integrity | бессрочно |
| **Service Account Key** (`.json`) | доступ к Google Play Developer API | создаётся в Google Cloud Console, права выдаются в Play Console → Users and permissions | EAS (уровень аккаунта) | бессрочно |

**Правило:** ключей подписи на Android всего два — upload и app signing, причём второй разработчик никогда не видит. Service account — это не про подпись билда, а про API-доступ к консоли, и нужен **только для автоматизации** (`eas submit`, fastlane, CI). При ручной загрузке AAB через UI Play Console он не нужен вообще.

### eas build ≠ eas submit
- `eas build` — только **собирает** бинарник (APK/AAB) и кладёт его на серверы Expo. В Play не публикует.
- `eas submit` — отдельная команда, заливает уже собранный билд в Play Console или App Store Connect.

```bash
eas build  -p android --profile=development   # для локального тестирования, обычно internal distribution
eas build  -p android --profile=production    # AAB для публикации
eas submit -p android --profile=production    # загрузка в Play Console
```

Профили (`development` / `preview` / `production`) описываются в `eas.json`.

### Google Play Developer API
REST API (`androidpublisher/v3`) для программного управления консолью: сабмит билдов, смена трека, метаданные, in-app products, отзывы, отчёты. Работает через модель **Edits** — транзакция: `edits.insert` → загрузка → `edits.tracks.update` → `edits.commit`; до коммита ничего не публикуется. Аутентификация — OAuth2 через service account.

## iOS

Нужен платный Apple Developer Program ($99/год). Релиз возможен **только** через App Store Connect.

| Сущность | Что это | Где живёт | Срок |
|---|---|---|---|
| **Bundle ID** | уникальное имя приложения (`com.company.app`, `com.company.app.dev`) | проект | бессрочно |
| **App ID** | Bundle ID + список разрешённых **capabilities** | проект | бессрочно |
| **Development Certificate** | подпись для запуска на реальном устройстве из Xcode | аккаунт/команда, общий | 1 год |
| **Distribution Certificate** | подпись всего публикуемого: и Ad Hoc, и App Store | аккаунт/команда, общий на все приложения | 1 год |
| **Development Provisioning Profile** | App ID + development cert + список UDID устройств разработчиков | проект | ~1 год |
| **Ad Hoc Provisioning Profile** | App ID + distribution cert + список UDID тестовых устройств | проект | ~1 год |
| **App Store Provisioning Profile** | App ID + distribution cert, **без** списка устройств | проект | ~1 год, automatic signing продлевает сам |
| **Apple Push Key / APN Key** (`.p8`) | ключ для пушей через APNs | аккаунт/команда, работает для всех приложений | бессрочно |
| **App Store Connect API Key** (`.p8`) | программный доступ к App Store Connect без интерактивного логина и 2FA (Issuer ID + Key ID + файл, скачивается один раз) | аккаунт/команда | бессрочно, отзывается вручную |

**Правило:** типов сертификатов всего два — Development (дебаг на устройстве) и Distribution (всё публикуемое). Provisioning profile, наоборот, **не один**: под один App ID одновременно живут development, ad hoc и store-профили, и это нормально. Каждый профиль жёстко привязан к связке App ID + тип назначения + соответствующий тип сертификата и не шарится между Bundle ID.

ASC API Key покрывает весь API App Store Connect (не только загрузку билда: TestFlight-группы, метаданные, отчёты), а его права зависят от роли, с которой он создан (Admin / App Manager / Developer).

### Capabilities
**Capabilities ≠ runtime-разрешения.** Capabilities — это Push Notifications, Sign in with Apple, HealthKit, App Groups, Associated Domains, iCloud: они включаются в App ID. Доступ к гео, камере, фото, микрофону — это runtime-разрешения: описываются в `Info.plist` (`NSCameraUsageDescription` и т.п.) и спрашиваются у пользователя во время работы. С App ID и порталом Apple они никак не связаны.

Где включаются capabilities:
1. Apple Developer Portal → Certificates, Identifiers & Profiles → Identifiers → App ID → чекбоксы.
2. Xcode → Target → Signing & Capabilities → «+ Capability» (создаёт или обновляет `.entitlements`).

Capability работает только в связке из трёх частей: галочка в App ID, provisioning profile с ней и файл `.entitlements` в сборке. При подписи Apple сверяет entitlements с профилем, поэтому «дописать себе» право нельзя. Оба места должны совпадать, а после добавления новой capability provisioning profile нужно перегенерировать (automatic signing делает это сам). В Expo/EAS capabilities задаются декларативно через config plugins в `app.json`, и EAS сам синхронизирует галочки на портале через API.

### Локальные сборки vs EAS-управляемые
- `eas build --profile development` — сборка на серверах Expo, credentials живут в `eas credentials` и там же видны.
- `npx expo run:ios` — подпись делает локально Xcode: Keychain на Mac + Apple ID, залогиненный в Xcode. EAS об этом не знает и покажет «not configured», хотя сертификат и профиль реально существуют — просто не в его системе учёта.
- Expo Go своего сертификата не требует вообще: это готовое подписанное приложение, которое подгружает JS-бандл.

### EAS умеет и генерировать сертификаты, и принимать готовые
- **Автогенерация** — EAS сам ходит в Apple Developer API и создаёт сертификат и профиль от имени аккаунта (при первой настройке нужна авторизация с 2FA).
- **Загрузка своего** — `eas credentials` → «Provide my own» либо `credentials.json`. Это важно для проекта с историей публикаций: нужно переиспользовать существующий Distribution-сертификат, а не плодить новый — у Apple есть лимит на число активных distribution-сертификатов на команду.

## Где что лежит в EAS

**Уровень аккаунта** (expo.dev → Account → Credentials) — общий пул, шарится между проектами: Apple Distribution Certificates, Apple Push Keys, App Store Connect API Keys, Apple Teams, Google Service Account Keys.

**Уровень проекта** (expo.dev → проект → Credentials или `eas credentials`) — привязано к Bundle ID / package name: на Android — upload keystore; на iOS — выбранный из общего пула distribution-сертификат + provisioning profile (всегда уникален для App ID).

Если под окружения (dev / staging / prod) заведены разные Bundle ID, под каждый нужен свой App ID и свой provisioning profile. Distribution certificate, Push Key и ASC API Key остаются общими на команду.

## Частые ошибки понимания
- «App signing key — альтернативный способ публикации». Нет: это следующий шаг после upload key. Разработчик подписывает upload-ключом, Google перевыпускает подпись своим.
- Заводить service account ради ручной публикации. Он нужен только для CI и `eas submit`.
- Считать, что `eas build` публикует приложение. Он только собирает.
- Путать capabilities с runtime-разрешениями и искать чекбокс «доступ к камере» на портале Apple.
- Делать отдельный Distribution-сертификат под Ad Hoc и под App Store. Сертификат один и тот же, различаются профили.
- Создавать новый сертификат на проекте с историей публикаций вместо загрузки существующего `.p12`.
- Считать, что «EAS не видит credentials» = их нет: при `expo run:ios` подпись живёт в Keychain, мимо учёта EAS.

## Мнемоника
Сертификат — один, профилей — много. Сертификат это подпись разработчика или команды, от приложения к приложению она не меняется. Provisioning profile — отдельная бумажка под каждую комбинацию «какое приложение + для чего», их естественно много.

Якорь: истекает раз в год и скачивается файлом (`.cer` / `.p12`) → это сертификат. Содержит список устройств и привязано к одному Bundle ID → это provisioning profile.

## Связи
- [[Deep Links]] — universal links требуют entitlement и capability Associated Domains, то есть упираются в подпись и provisioning profile; App Links сверяются с SHA-256 ключа Play App Signing.
