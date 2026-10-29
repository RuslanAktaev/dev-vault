---
tags: [moc, javascript]
related: []
---
# JavaScript MOC

Порядок чтения: типы → контекст → замыкания → this → асинхронность → Symbol и итераторы.

- [[Типы]] — 8 типов, `NaN`, `typeof null`, приведение `null`/`undefined`, bigint.
- [[Функции и execution context]] — call stack, две фазы контекста, hoisting и TDZ, виды функций.
- [[Closures]] — функция + лексическое окружение, захват переменной, stale closures.
- [[This binding]] — 4 правила привязки, `bind` против `call`, что делает `new`, стрелочные функции, `this` верхнего уровня по средам.
- [[Event Loop]] — задачи, микрозадачи, рендер, порядок выполнения, синхронный executor и `async` до `await`, отличия Node.
- [[Symbol и итераторы]] — уникальные символы и реестр, протокол итерации, генераторы, `yield*`.
