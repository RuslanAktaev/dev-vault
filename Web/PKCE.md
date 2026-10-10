---
tags: [web]
related: ["[[OAuth 2.0]]", "[[OIDC]]"]
---
# PKCE

**Определение.** PKCE (Proof Key for Code Exchange, читается «пикси») — расширение OAuth 2.0 (RFC 7636). Оно связывает одноразовый `code` с тем Client, который начал вход, и перехваченный `code` становится бесполезным. Это не библиотека, а подход из двух частей:
- `code_verifier` — секретная случайная строка;
- `code_challenge` — её хеш.

## Проблема
В Authorization Code flow провайдер возвращает `code` редиректом, то есть через адресную строку. У SPA и мобильных приложений нет места, где можно спрятать `client_secret`. Значит, тот, кто перехватил `code`, может сам обменять его на токены. Перехватить могут вредоносное приложение на устройстве, расширение браузера, история, логи.

## Как работает
1. Client генерирует случайную строку `code_verifier` (43–128 символов) и хранит её у себя.
2. Считает `code_challenge = BASE64URL(SHA256(code_verifier))`.
3. Отправляет на `/authorize` только `code_challenge` и `code_challenge_method=S256`. Провайдер запоминает challenge вместе с будущим `code`.
4. После логина провайдер редиректит обратно с `code`.
5. Client меняет `code` на токены на `/token` и отправляет `code_verifier`. Провайдер хеширует verifier, сравнивает с сохранённым `code_challenge` и выдаёт токены только при совпадении.

**Почему это защищает.** Атакующий с перехваченным `code` не знает `code_verifier`: по сети шёл только хеш, а по хешу строку не восстановить.

## Кто что делает

| Роль | Что делает | Что хранит |
|---|---|---|
| Client | Генерирует `code_verifier`, считает `code_challenge`, отправляет challenge на `/authorize`, позже verifier на `/token` | `code_verifier` до конца обмена |
| Authorization Server | Запоминает `code_challenge` вместе с `code`. На `/token` хеширует присланный verifier и сверяет | `code_challenge` |
| Resource Owner | Ничего, про PKCE он не знает | — |

- Провайдер в PKCE ничего не генерирует, он только запоминает и проверяет.
- «Client» — тот, кто начинает вход. Для SPA это JS в браузере, для мобильного приложения — его код, для сайта с бэкендом — сам бэкенд.
- `code_verifier` нужно сохранить между шагами 1 и 5. SPA кладёт его в `sessionStorage` или память, бэкенд — в серверную сессию.

| Что | Кто создаёт | Когда |
|---|---|---|
| `code_verifier`, `code_challenge` | Client | При нажатии «Войти», до редиректа. Заново на каждый вход |
| Запись с `code_challenge` и одноразовый `code` | Authorization Server | После логина и согласия пользователя |
| `access_token`, `id_token`, `refresh_token` | Authorization Server | При обмене `code` на токены |

## Код

**Client** (SPA, в момент нажатия «Войти»):

```js
// 1. code_verifier: 32 случайных байта в base64url
const bytes = crypto.getRandomValues(new Uint8Array(32));
const codeVerifier = base64url(bytes);                    // "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

// 2. code_challenge: SHA-256 от verifier, тоже в base64url
const digest = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(codeVerifier));
const codeChallenge = base64url(new Uint8Array(digest));  // "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

// base64url: обычный base64, где + → -, / → _, а = в конце убраны
function base64url(bytes) {
  return btoa(String.fromCharCode(...bytes))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

sessionStorage.setItem('code_verifier', codeVerifier);    // сохранить до обмена
location.href = `${authServer}/authorize?...&code_challenge=${codeChallenge}&code_challenge_method=S256`;
```

Client на Node:

```js
const codeVerifier  = crypto.randomBytes(32).toString('base64url');
const codeChallenge = crypto.createHash('sha256').update(codeVerifier).digest('base64url');
```

**Authorization Server** хранит обычную запись в БД или Redis с ключом `code`. Запись живёт около минуты и удаляется при первом обмене:

```js
// пользователь вошёл и согласился
await store.set(`authcode:${code}`, {
  clientId, userId, redirectUri, scope,
  codeChallenge, codeChallengeMethod: 'S256',   // пришли в /authorize
  expiresAt: Date.now() + 60_000,
});

// запрос на /token
const record = await store.get(`authcode:${code}`);
await store.delete(`authcode:${code}`);         // код одноразовый
if (base64url(sha256(codeVerifier)) !== record.codeChallenge) return error('invalid_grant');
```

Провайдер не «раскодирует» verifier, а хеширует его и сравнивает. Хеш необратим, поэтому из challenge verifier не восстановить.

Руками это почти не пишут: Auth.js, `openid-client`, SDK провайдеров (Auth0, Clerk) генерируют и хранят verifier сами. От разработчика нужны `client_id`, `redirect_uri` и настройка провайдера. Механику нужно знать для отладки: ошибка **`invalid_grant`** на `/token` обычно значит, что `code_verifier` не совпал или потерялся между шагами (другая вкладка или сессия, сервер перезапустился).

## Почему хешируют на клиенте
Данные идут по двум разным каналам:

| Канал | Что идёт | Кто может увидеть |
|---|---|---|
| Через браузер (редиректы, адресная строка) | `code_challenge` на `/authorize`, `code` на возврате | История, логи, `Referer`, расширения, чужие приложения |
| Прямой HTTPS-запрос Client → Authorization Server | `code_verifier` на `/token` | Практически никто: verifier в теле POST, а не в URL. В DevTools его видит только сам пользователь |

Если бы verifier ушёл на `/authorize`, он прошёл бы по небезопасному каналу вместе с `code`. Поэтому через браузер идёт только необратимый хеш, а секрет уходит одним прямым запросом в конце. В спецификации есть ещё метод `plain` (challenge равен verifier), но он слабее, и везде, где можно, используют `S256`.

**`code` — не токен.** Это одноразовая короткоживущая (около минуты) «квитанция». Сама она ничего не открывает и нужна только для обмена на токены. Через браузер она идёт как раз потому, что ценного в ней нет. Токены Client получает уже прямым запросом.

## Что PKCE даёт и чего не даёт

| Даёт | Не даёт |
|---|---|
| Перехваченный `code` нельзя обменять на токены | Защиты от XSS в самом Client: скрипт прочитает и verifier, и токены |
| Подсунутый чужой `code` не подойдёт к verifier Client | Доказательства, что Client — именно твоё приложение (эту роль играл `client_secret`) |
| Защиту без `client_secret`, которого у SPA и мобильных приложений нет | Замены проверке `redirect_uri` и `state` |

## Где нужен
- **Public clients** (SPA, мобильные приложения): по современным рекомендациям обязателен. На практике часть провайдеров отклоняет вход без PKCE, часть пока разрешает.
- **Confidential clients** (бэкенд с `client_secret`): классический вариант работает и без PKCE, потому что секрет подтверждает, что код обменивает тот самый Client. PKCE всё равно рекомендуется как дополнительная защита.
- **OAuth 2.1**: обязателен для всех клиентов.

## Частые ошибки понимания
- **«Провайдер генерирует challenge».** Нет, verifier и challenge генерирует Client. Провайдер только хранит challenge и сверяет.
- **«PKCE заменяет `state`».** Нет. `state` защищает редирект от CSRF, `nonce` защищает `id_token` от повтора, PKCE защищает обмен `code` на токены.
- **«PKCE спасает от XSS».** Нет. Скрипт на странице Client читает всё, что читает сам Client.
- **Использовать `plain`.** Тогда challenge равен verifier и идёт через браузер, и защита теряется.
- **Генерировать verifier один раз на всё приложение.** Он нужен новый на каждый вход.

## Связи
- [[OAuth 2.0]] — PKCE встраивается в шаги Authorization Code flow. Confidential client вместо него может полагаться на `client_secret`.
- [[OIDC]] — тот же flow плюс `nonce`. Не путать, что защищает каждый из механизмов.
