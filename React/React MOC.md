---
tags: [moc, react]
related: []
---
# React MOC

- [[Virtual DOM]] — идея: render → reconciliation → commit, `key`, почему VDOM не «быстрее DOM».
- [[Fiber и реконсиляция]] — элемент, файбер, реконсилер, рендерер; что хранит файбер, два дерева, ререндер, bailout.
- [[setState и хуки]] — что делает `setState`, когда выполнится render, батчинг, хуки в `memoizedState`.
- [[RTK Query]] — серверное состояние: кэш по аргументам, подписки, теги и инвалидация, optimistic updates.
- [[Rehydration]] — два смысла: гидрация SSR-разметки (и mismatch) и восстановление состояния стора.
