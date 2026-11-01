---
tags: [moc, react-native]
related: []
---
# React Native MOC

- [[Expo Router]] — файловые конвенции, layouts и дерево состояния, push / navigate / dismissTo, вложенные стеки и dismissAll, что вызывает mount, Local vs Global params, режимы `presentation`, Protected, статические redirects, anchor, тесты.
- [[Deep Links]] — custom scheme vs universal / App Links, AASA и entitlements, проверка и отладка, `+native-intent` allowlist, безопасность.
- [[New Architecture]] — замена Bridge: JSI, TurboModules (lazy + Codegen), Fabric (C++ shadow tree); формулировка и вопросы для собеса.
- [[JSI]] — C++ API встраивания движка, что RN регистрирует при старте, цепочка JS → C++ → Swift/Kotlin.
- [[TurboModules и Codegen]] — стандарт нативных модулей, спек и граница JS ↔ натив, как работает Codegen и что он генерирует.
- [[Hermes]] — движок по умолчанию: AOT-байткод, бандл и OTA, Hermes V1.
- [[Fabric]] — рендерер: хост-операции, иммутабельное Shadow Tree, зачем diff, Yoga до mount, от `setState` до экрана, кастомная нативная вьюха.
- [[Виртуализированные списки]] — FlatList, FlashList (recycling, `getItemType`, v2), LegendList, подгрузка, что когда выбирать.
- [[Routing]] — React Navigation (stack / tabs / вложенность), жизненный цикл экранов в стеке.
- [[Signing and Credentials]] — сертификаты и ключи Android / iOS, разделение eas build и eas submit, что где лежит в EAS.