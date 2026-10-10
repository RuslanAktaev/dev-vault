---
tags: [web]
related: ["[[OAuth 2.0]]", "[[PKCE]]", "[[Сессии и JWT]]"]
---
# OIDC

**Определение.** OIDC (OpenID Connect) — надстройка над OAuth 2.0, которая отвечает на вопрос «кто этот пользователь». Сам OAuth выдаёт `access_token` для API и ничего не говорит о личности. OIDC добавляет стандартный способ её узнать. «Войти через Google» — это OIDC.

## Что добавляет OIDC

| Что | Зачем |
|---|---|
| scope `openid` | Включает OIDC. Без него провайдер вернёт обычный OAuth-ответ |
| `id_token` | JWT с данными о том, кто вошёл. Приходит вместе с `access_token` |
| `nonce` | Случайная строка: приложение отправляет её на `/authorize` и сверяет с `id_token` |
| scopes `profile`, `email` | Какие данные о пользователе включить |
| `/userinfo` | Эндпоинт, который по `access_token` отдаёт данные о пользователе |
| Discovery `/.well-known/openid-configuration` | Адреса эндпоинтов и ключей провайдера в одном JSON |
| JWKS (`jwks_uri`) | Публичные ключи для проверки подписи `id_token` |

## Поток
Это тот же Authorization Code + PKCE. В запрос добавляются `scope=openid profile email` и `nonce`, а в ответе `/token`, кроме `access_token`, приходит `id_token`.

```
GET /authorize?response_type=code&client_id=...&redirect_uri=...
    &scope=openid%20profile%20email&state=...&nonce=...
    &code_challenge=...&code_challenge_method=S256
```

```json
// ответ /token
{ "access_token": "...", "id_token": "eyJ...", "token_type": "Bearer", "expires_in": 3600 }

// payload id_token после раскодирования
{ "iss": "https://accounts.google.com", "sub": "248289761001", "aud": "my-client-id",
  "exp": 1790000000, "iat": 1789996400, "nonce": "...", "email": "ivan@gmail.com", "name": "Ivan" }
```

## id_token и access_token

| | `id_token` | `access_token` |
|---|---|---|
| Для кого | Для твоего приложения | Для API |
| Что говорит | Кто вошёл и когда | Что разрешено |
| Что с ним делают | Проверяют и берут из него пользователя | Шлют в API в `Authorization: Bearer` |

`id_token` не отправляют в API вместо `access_token`.

## Как проверять id_token
Проверки те же, что для любого JWT:
- подпись — по ключам из JWKS;
- `iss` равен адресу провайдера;
- `aud` равен твоему `client_id`;
- `exp` не истёк;
- `nonce` совпадает с отправленным.

Обычно всё это делает библиотека: Auth.js, `openid-client`, `jose`.

## Кто пользователь
`sub` — стабильный идентификатор пользователя у этого провайдера. В своей БД пользователя связывают по паре (`iss`, `sub`), а **не по email**: email может смениться и не всегда подтверждён (`email_verified`).

## Что дальше
Проверив `id_token`, приложение обычно создаёт свою обычную сессию (куку с session id). Токены провайдера после этого не нужны, если только приложению не требуется доступ к его API.

## Частые ошибки понимания
- **Слать `id_token` в API.** Для API есть `access_token`. `id_token` предназначен приложению.
- **Связывать аккаунт по email.** Связывают по (`iss`, `sub`).
- **Не проверять `aud` и `nonce`.** Тогда приложение примет токен, выданный другому клиенту, или старый повторно подсунутый токен.
- **«Без scope `openid` тоже OIDC».** Без него провайдер вернёт чистый OAuth, и `id_token` не придёт.

## Связи
- [[OAuth 2.0]] — OIDC построен поверх OAuth: роли, flow и регистрация клиента те же.
- [[PKCE]] — поток OIDC использует PKCE. `nonce` и PKCE защищают от разных атак.
- [[Сессии и JWT]] — `id_token` — это JWT, правила проверки общие. После входа создают свою серверную сессию.
