---
tags: [react-native]
related: ["[[App Router и Server Components]]", "[[Signing and Credentials]]"]
---
# Routing

Навигация в RN строится не на URL, как в вебе, а на **стеке экранов** и дереве навигаторов.

## React Navigation
- Навигаторы: **stack** (`native-stack` использует нативные примитивы UINavigationController / Fragment), **tabs**, **drawer**. Навигаторы вкладываются друг в друга.
- Состояние навигации — дерево: `{ routes, index }` на каждом уровне.
- Params передаются между экранами и должны быть **сериализуемыми** (нужно для deep links и восстановления состояния).
- Deep linking — конфиг `linking` сопоставляет URL с экранами.
- Типизация: `ParamList` для каждого навигатора.

## Expo Router
Файловый роутинг **поверх** React Navigation.
- `app/` — структура папок = маршруты. `_layout.tsx` задаёт навигатор уровня.
- `(group)` — группы без сегмента в URL, `[id]` — динамические сегменты, `+not-found`, `+html`.
- Каждый экран автоматически получает URL → deep links и universal links без отдельного конфига, typed routes.
- Редиректы и защита маршрутов через layout (auth guard).

## Частые ошибки понимания
- **Экраны в стеке не размонтируются**, когда поверх открывают новый. `useEffect` не перезапустится при возврате. Для этого есть `useFocusEffect` / `useIsFocused`.
- Передавать в params функции или большие объекты. Лучше id + загрузка данных на экране.
- Навигация во вложенный навигатор: `navigate('Parent', { screen: 'Child', params })`, а не напрямую.
- Ожидать веб-семантики истории. Stack — это push/pop, а не history API.

## Связи
- [[App Router и Server Components]] — Expo Router перенял файловую модель Next.js: `app/`, layouts, группы, динамические сегменты.
- [[Signing and Credentials]] — deep links и universal links на iOS работают только при capability Associated Domains в App ID, а значит через перевыпуск provisioning profile.
