---
tags: [react-native]
related: ["[[Routing]]", "[[App Router и Server Components]]"]
---
# Expo Router

Файловый роутер **поверх React Navigation**: папка `app/` (или `src/app/`) превращается в дерево навигаторов, и у каждого экрана появляется URL. Всё следует из трёх идей:
- **Файл = экран = URL.** `app/explore.tsx` — это экран `/explore`. Конфига маршрутов нет, его заменяет файловая система.
- **`_layout.tsx` = навигатор.** Оборачивает соседние файлы и подпапки. Тип (`Stack`, `Tabs`, `NativeTabs`, `Slot`) задаёт поведение.
- **Навигация = смена URL.** `Link` и `router` принимают путь. Роутер сам находит, в каком навигаторе лежит экран, и решает, как его показать.

> [!note] SDK 57
> React Navigation встроен прямо в `expo-router`: темы (`ThemeProvider`), `useRoute`, `StackRouter` импортируются из `expo-router`, пакеты `@react-navigation/*` напрямую не нужны.

## Файловые конвенции

| Файл в `app/` | URL | Роль |
|---|---|---|
| `about.tsx` | `/about` | статический маршрут |
| `index.tsx` | `/` или `/папка` | маршрут по умолчанию для папки |
| `user/[id].tsx` | `/user/42` | динамический сегмент → параметр `id` |
| `docs/[...slug].tsx` | `/docs/a/b/c` | catch-all, `slug = ['a','b','c']` |
| `(tabs)/home.tsx` | `/home` | группа: в URL не попадает, нужна для общего layout |
| `(a,b)/feed.tsx` | `/feed` в обеих группах | массив групп: один файл в двух вкладках |
| `_layout.tsx` | — | навигатор для соседей |
| `+not-found.tsx` | любой несовпавший | 404 |
| `+html.tsx` | — | HTML-обёртка, только web |
| `+native-intent.tsx` | — | переписывает входящие deep links, не совпавшие с маршрутами (пуши, сторонние SDK) |
| `+middleware.ts` | — | код до рендера маршрута (web, серверный вывод) |
| `api/hello+api.ts` | `/api/hello` | API-роут на сервере, не экран |
| `screen.ios.tsx` / `.web.tsx` | как у `screen` | платформенная версия, базовый `screen.tsx` обязателен |
| `_sitemap` | `/_sitemap` | генерируется сам: список всех маршрутов |

- **Приоритет совпадений:** статический > `[param]` > `[...rest]`. Поэтому `user/me.tsx` и `user/[id].tsx` живут рядом.
- Catch-all: тип `useLocalSearchParams<{ slug: string[] }>()`, путь обратно — `slug.join('/')`. Чтобы голый `/docs` открывался, нужен `docs/index.tsx`.
- **Всё, что не экран, держи вне `app/`**: любой `.tsx` там становится маршрутом. Компоненты и хуки — в `components/`, `hooks/`.

## Layouts и навигаторы

| Навигатор | Импорт (SDK 57) | Поведение |
|---|---|---|
| `Stack` | `expo-router` | нативный стек: карточки, свайп назад, заголовок, модалки |
| `NativeTabs` | `expo-router/unstable-native-tabs` | системный таб-бар iOS/Android |
| `Tabs` | `expo-router/js-tabs` | JS-табы, полностью кастомизируемые (импорт из `expo-router` — deprecated) |
| JS Stack | `expo-router/js-stack` | стек на JS для кастомных анимаций |
| `Drawer` | `expo-router/drawer` | боковое меню |
| `Slot` | `expo-router` | только верхний экран, без UI |

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="modal" options={{ presentation: 'modal' }} />
    </Stack>
  );
}
```

- **`Stack.Screen` не создаёт маршрут**, его создаёт файл. `Stack.Screen` только настраивает экран и задаёт порядок; неописанные файлы доступны с опциями по умолчанию.
- `name` — путь относительно layout без расширения: `"(tabs)"`, `"post/[id]"`, `"modal"`.
- **Вложенная папка со своим `_layout` — один экран родителя.** Для корневого Stack вся `(tabs)` — одна карточка, поэтому ей скрывают заголовок.
- Опции можно задать изнутри экрана: `<Stack.Screen options={{ title: post.title }} />` → динамический заголовок.
- **Каждый навигатор ведёт историю только своих детей.** После main → profile → edit: корневой стек = `[main, profile]`, стек profile = `[index, edit]`. «Назад» снимает с самого глубокого стека, потом с родителя. Стек неактивной вкладки сохраняется.
- Layout не размонтируется при переходах внутри него → провайдеры (тема, авторизация, query client) кладут в корневой `_layout`.
- Заголовок у `Stack`, JS `Tabs` и `Drawer` включён по умолчанию (покажет имя файла). У `NativeTabs` и `Slot` его нет. Вложенный навигатор с заголовком в родителе с заголовком = две полосы.

### Slot или Stack
Оба на одном `StackRouter`: push, back, dismiss одинаковы. Разница в том, что смонтировано.

| | Slot | Stack |
|---|---|---|
| Смонтировано | только верхний экран | все экраны стека |
| После `back()` | предыдущий экран монтируется заново: стейт, скролл, ввод сброшены, `useEffect(…, [])` срабатывает снова | тот же экран, стейт на месте |
| Заголовок, анимация, свайп | нет | есть |

`Slot` берут, когда UI навигатора не нужен: корневой layout с провайдерами, веб со своей шапкой, свой навигатор. Не путать с `TabSlot` из `expo-router/ui` (контент активной вкладки у headless-табов).

### Дерево состояния
Состояние навигации одно на приложение, но это **дерево, повторяющее папки**:
- каждый `_layout` → `{ type: 'stack' | 'tab' | 'drawer', index, routes }`;
- каждый открытый экран → элемент `routes: { name, params? }`;
- у папки со своим layout элемент получает поле `state`, и дальше рекурсивно.

```ts
// Tabs [feed, profile], в feed свой Stack; открыл /feed/7, потом вкладку profile
{ type: 'stack', index: 0, routes: [
  { name: '(tabs)', state: {
      type: 'tab', index: 1, routes: [
        { name: 'feed', state: {
            type: 'stack', index: 1, routes: [
              { name: 'index' },
              { name: '[id]', params: { id: '7' } },
            ] } },
        { name: 'profile' },
      ] } },
] }
```

- В `routes` стека — только открытые экраны, во вкладках — все вкладки сразу. Всё, что можно открыть, — `routeNames` или `/_sitemap`.
- Дерево целиком — `useRootNavigationState()`, ветка — `useNavigation().getState()`, плоский путь — `usePathname()` / `useSegments()`.
- Один layout — один навигатор. Drawer + вкладки = вложенные папки.
- **`router` один на приложение.** Результат вызова зависит от того, что открыто, а не от места вызова в коде (исключения — `useNavigation()` и относительные пути).

### Схемы вложенности
- **Stack → Tabs.** Детальные экраны и модалки открываются поверх таб-бара и скрывают его. Самая частая.
- **Tabs → Stack в каждой вкладке.** Детальные внутри вкладки, таб-бар виден, вкладка помнит глубину (как Instagram). Пример: `(tabs)/(home)/_layout.tsx` со своим Stack.
- **Комбинация.** Корневой Stack для модалок и логина → Tabs → Stack в каждой вкладке.

## Навигация: Link и router

Декларативно — `<Link href>` (по умолчанию). Императивно — `router.*` после события (сохранение формы, логин, ответ сервера).

```tsx
import { Link, router } from 'expo-router';

<Link href="/about">О приложении</Link>

<Link href={{ pathname: '/post/[id]', params: { id: '42' } }} asChild>
  <Pressable><Text>Пост 42</Text></Pressable>
</Link>

const onSave = async () => {
  await save();
  router.back();
};
```

Стек `[A, B, C]`, открыт C:

| Вызов | Стек после | Когда |
|---|---|---|
| `push('/B')` | `[A, B, C, B]` | всегда новая карточка |
| `navigate('/B')` | `[A, B, C, B]` | как push, но если цель — текущий экран, остаётся на нём и обновляет параметры. Так работает `Link` |
| `replace('/D')` | `[A, B, D]` | без истории: после логина, онбординга |
| `back()` | `[A, B]` | шаг назад, любой навигатор |
| `dismiss(2)` | `[A]` | закрыть N экранов ближайшего стека |
| `dismissTo('/A')` | `[A]` | закрывать до href; если его нет в стеке — заменить текущий |
| `dismissAll()` | `[A]` | к первому экрану стека (popToTop) |

- **`navigate` больше не возвращает назад** к существующему экрану (с Expo Router v4 / SDK 52, как в React Navigation 7). Старое поведение вынесено в `dismissTo`.
- **navigate vs push различаются, только когда цель = текущий экран** (то же имя и те же path-параметры). С `/post/42?tab=info`:

| Вызов | navigate | push |
|---|---|---|
| `/post/43` | новый экран | новый экран |
| `/post/42?tab=comments` | тот же экран, параметры обновлены | новый экран |
| `/post/42?tab=info` | ничего | дубль |

- Когда navigate остаётся на экране, тот не размонтируется: `useLocalSearchParams()` отдаёт новое, а `useEffect(…, [])` и `useFocusEffect` не перезапускаются → загрузку привязывай к параметру: `useEffect(() => load(tab), [tab])`.
- Параметры заменяются **целиком, без слияния**. Поменять один и сохранить остальные — `router.setParams({...})` (без перехода).
- `push` нужен, когда экраны отличаются только search-параметрами и нужна история: поиск `/search?q=…`. **Во вкладках и drawer push = navigate.**
- Проверки: `router.canGoBack()`, `router.canDismiss()`.
- У `Link` те же режимы пропами: `push`, `replace`, `dismissTo`. `prefetch` заранее рендерит цель, `withAnchor` подкладывает anchor-экран вложенного стека под цель.
- Относительные пути (`./edit`, `../`) считаются от текущего маршрута; абсолютные надёжнее.
- **Typed routes** (`experiments.typedRoutes: true`): тип `Href` генерируется из файлов, опечатка в `href` — ошибка TS. Типы генерирует dev-сервер: перед `tsc --noEmit` хотя бы раз запусти `npx expo start`.

## Параметры и хуки

Параметры — часть URL, поэтому **всегда строки** (или массивы строк). `[id]` и query приходят в один объект.

```tsx
// app/post/[id].tsx, открыт /post/42?tab=comments
const { id, tab } = useLocalSearchParams<{ id: string; tab?: string }>();
const postId = Number(id); // число — только явным преобразованием
```

**Local vs Global** (стек `/post/1 → /post/2`, смонтированы оба):

| Хук | Возвращает | Перерендер |
|---|---|---|
| `useLocalSearchParams()` | параметры **этого** экрана: нижняя карточка видит `id='1'` | только при смене своих параметров |
| `useGlobalSearchParams()` | параметры **текущего URL**: обе видят `id='2'` | при любой смене URL во всех экранах |

Правило: в экранах — `useLocalSearchParams`. Global — для редких случаев вроде аналитики в layout.

Остальные хуки:
- `usePathname()` — путь без групп и query: `/post/42`.
- `useSegments()` — с группами и шаблонами: `['(tabs)', 'post', '[id]']`. Удобно проверять, в какой группе пользователь.
- `useRouter()` — тот же `router`.
- `useNavigation()` — навигатор React Navigation: `setOptions`, подписки на события.
- `useFocusEffect(cb)` / `useIsFocused()` — реакция на фокус. Открытие модалки поверх тоже вызывает blur.

В параметры кладут **только id и простые флаги**. Данные достают по id из кэша/стора — тогда экран откроется и по deep link.

## Модалки, защита, anchor, deep links
Всё настраивается в `_layout.tsx`, а не в экранах.

**Модалки** — обычный экран стека с другим `presentation`. Чтобы перекрывала таб-бар, объявляй в **корневом** Stack, а не внутри вкладки. Закрывается `router.back()` / `router.dismiss()`.
```tsx
<Stack.Screen name="modal" options={{ presentation: 'modal' }} />
<Stack.Screen name="sheet" options={{ presentation: 'formSheet', sheetAllowedDetents: [0.5, 1] }} />
```

**Защищённые маршруты** — `Stack.Protected guard`. Пока условие ложно, экраны скрыты; переход на скрытый ведёт на anchor или первый доступный. При смене guard роутер сам чистит историю от недоступных экранов — ручные редиректы в `useEffect` не нужны. Есть `Tabs.Protected`, `Drawer.Protected`. В SDK 57 — только `guard`, `redirectTo` появился в SDK 58.
```tsx
const { isLoggedIn } = useAuth();

<Stack>
  <Stack.Protected guard={!isLoggedIn}>
    <Stack.Screen name="login" />
  </Stack.Protected>
  <Stack.Protected guard={isLoggedIn}>
    <Stack.Screen name="(tabs)" />
  </Stack.Protected>
</Stack>
```

**Anchor** — экран, который всегда лежит в основании стека. Без него при открытии сразу `/post/42` стек из одного экрана и «назад» некуда. Старое имя — `initialRouteName` (deprecated).
```ts
// app/_layout.tsx
export const unstable_settings = { anchor: '(tabs)' };
```

**Deep links и редиректы:**
- Каждый экран уже доступен по ссылке: `scheme: "myapp"` → `myapp://post/42` откроет `post/[id].tsx`. Ручной `linking` не нужен.
- `<Redirect href="/login" />` — декларативный редирект при рендере. Для авторизации лучше `Protected`.
- Статические redirects/rewrites — в опциях плагина `expo-router` в `app.json`, когда старые URL должны работать после переименования файлов.
- Проверка на симуляторе: `xcrun simctl openurl booted myapp://post/42`.

## Тестирование
Три уровня: `/_sitemap`, ручные deep links, Jest через `expo-router/testing-library` (роутер в памяти, без симулятора). Нужны `jest-expo`, `jest`, `@testing-library/react-native`, в `package.json` — `"jest": { "preset": "jest-expo" }`.

```tsx
import { Stack } from 'expo-router';
import { renderRouter, screen, testRouter } from 'expo-router/testing-library';

it('push кладёт пост в стек и back возвращает', () => {
  renderRouter(
    {
      _layout: () => <Stack />,
      index: () => <Text>Home</Text>,
      'post/[id]': () => <Text>Post</Text>,
    },
    { initialUrl: '/' },
  );

  testRouter.push('/post/42');
  expect(screen).toHavePathname('/post/42');
  expect(screen).toHaveSegments(['post', '[id]']);

  testRouter.back('/');
  expect(screen.getByText('Home')).toBeVisible();
});
```

- `renderRouter(context, { initialUrl })` — виртуальная ФС (ключ — путь в `app/`) или путь к настоящей папке.
- `testRouter.push / navigate / replace / back / setParams / dismissAll / canGoBack` — со встроенной проверкой пути.
- Матчеры: `toHavePathname`, `toHavePathnameWithParams`, `toHaveSegments`, `toHaveSearchParams`, `toHaveRouterState`.
- Что покрывать: гость → `/login`, после входа → `/`; deep link `/post/42` и `canGoBack() === true` благодаря anchor; неизвестный путь → `+not-found`; query доходит до экрана.

## Частые ошибки понимания
Почти все баги роутинга — экран, положенный не в тот навигатор.

| Симптом | Причина | Исправление |
|---|---|---|
| Два заголовка друг над другом | вложенный навигатор в Stack с заголовком | `headerShown: false` на `Stack.Screen` группы |
| Модалка под таб-баром | файл внутри `(tabs)` | перенести в корень к корневому Stack |
| После deep link нет «назад» | стек из одного экрана | `unstable_settings = { anchor }` |
| Данные не обновляются при возврате | экран не размонтировался | `useFocusEffect` |
| На нижней карточке чужие данные | `useGlobalSearchParams` в экране | `useLocalSearchParams` |
| Хелпер появился в `_sitemap` | любой файл в `app/` — маршрут | вынести в `components/` / `hooks/` |
| После логина свайп возвращает на логин | переход через `push` | `Stack.Protected` или хотя бы `replace` |
| `id === 42` ложно | параметры — строки | `Number(id)` и валидация |
| Новый файл не виден в `Href` | типы генерирует dev-сервер | `npx expo start` |

- Думать, что `Stack.Screen` объявляет маршрут. Маршрут — это файл.
- Думать, что `navigate` вернёт к уже открытому экрану. Для этого `dismissTo`.
- Думать, что `Slot` — это просто Stack без заголовка. Он размонтирует предыдущие экраны.

### Самопроверка
- Какой URL у `app/(auth)/(onboarding)/step/[n].tsx`?
- Почему для корневого Stack вся `(tabs)` — один экран?
- Стек `[A, B, C]`: что после `push('/B')`, `navigate('/B')`, `dismissTo('/A')`?
- Чем `useSegments()` отличается от `usePathname()` для `(tabs)/post/[id]`?
- Куда положить модалку, чтобы она перекрывала таб-бар?
- Что сделает роутер, если пользователь на защищённом экране, а guard стал false?
- Что изменится для пользователя, если перенести `post/[id]` из корневого Stack в Stack внутри вкладки?

## Связи
- [[Routing]] — общая модель навигации в RN (стек, дерево навигаторов, React Navigation), поверх которой построен Expo Router.
- [[App Router и Server Components]] — Expo Router перенял файловую модель Next.js: `app/`, layouts, группы, динамические сегменты.
