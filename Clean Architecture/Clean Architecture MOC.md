---
tags: [moc, clean-architecture]
related: []
---
# Clean Architecture MOC

Принципы от уровня классов до уровня системы. Порядок чтения — сверху вниз.

## Принципы кода
- [[SOLID]] — пять принципов для модулей и классов. SRP про акторов, DIP про направление зависимостей.

## Компоненты
- [[Component Cohesion]] — что класть в один компонент: REP, CCP, CRP и напряжение между ними.
- [[Component Coupling]] — как компоненты зависят друг от друга: ADP, SDP, SAP, метрики I / A / D, Zone of Pain.

## Архитектура
- [[CA Layers]] — Entities → Use Cases → Interface Adapters → Frameworks & Drivers и Dependency Rule.
- [[Boundaries]] — decoupling modes, Humble Object, partial boundaries, HAL / OSAL / PAL.
- [[Services Architecture]] — почему сервисы — не архитектура: decoupling fallacy, kitty problem.
- [[Packaging Strategies]] — by layer / by feature / ports and adapters / by component, антипаттерн Périphérique.

## Практика
- [[Nx монорепа]] — как всё это ложится на feature / ui / data-access / util в Nx.
