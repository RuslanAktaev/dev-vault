---
tags: [nextjs]
related: ["[[SSR-SSG-ISR]]", "[[Rehydration]]", "[[Virtual DOM]]", "[[Routing]]"]
---
# App Router и Server Components

## App Router
Файловый роутинг в `app/`, построенный на React Server Components.
- `page.tsx` — страница, `layout.tsx` — обёртка. Layout **сохраняет состояние** при навигации между дочерними маршрутами.
- Спец-файлы: `loading.tsx` (Suspense-граница), `error.tsx` (error boundary, client), `not-found.tsx`, `route.ts` (API-хендлер).
- Сегменты: `[slug]`, `[...slug]`, `(group)`, параллельные (`@slot`) и перехватывающие (`(.)`) маршруты.
- Streaming: HTML и RSC payload отдаются частями по мере готовности Suspense-границ.

## Server Components (RSC)
Компоненты **по умолчанию серверные**:
- рендерятся только на сервере, их код **не попадает в клиентский бандл**;
- могут быть `async` и получать данные напрямую (БД, секреты);
- без state, эффектов и обработчиков событий.

Результат — **RSC payload**: сериализованное дерево. Клиентский React вставляет его и reconcile-ит, не теряя состояние client components.

## Client Components
`'use client'` — **граница**: сам модуль и всё, что он импортирует, попадают в клиентский бандл.
- Client components **тоже рендерятся на сервере в HTML** (SSR) и затем гидрируются.
- Props через границу server → client должны быть сериализуемыми: без функций, кроме Server Actions.
- Server component можно передать в client component как `children` или prop, но **нельзя импортировать** его внутри client-модуля.

**Server Actions** (`'use server'`) — функции, которые вызываются с клиента и выполняются на сервере. Используются для мутаций и форм.

## Частые ошибки понимания
- `'use client'` = «рендерится только в браузере». Нет: это граница бандла, SSR всё равно есть.
- Ставить `'use client'` в каждый файл. Нужно только в точке входа клиентского поддерева. Лучше выносить интерактивность в листья дерева.
- Ожидать, что `useState` или контекст работают в server component.
- Передавать через границу функции, классы, `Date` без учёта сериализации.

## Связи
- [[SSR-SSG-ISR]] — стратегия рендеринга маршрута в App Router выводится из данных в серверных компонентах.
- [[Rehydration]] — гидрируются только client components.
- [[Virtual DOM]] — RSC payload — дерево элементов, которое клиент встраивает в своё дерево через reconciliation.
- [[Routing]] — Expo Router заимствует файловую модель App Router.
