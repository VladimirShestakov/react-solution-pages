---
title: Инверсия зависимостей
description: Прямая связь инвертируется - не модуль решает с кем работать, а вызывающая сторона передаёт все зависимости. Использование функционального подхода. На примере использования общих экземпляров.
keywords:
  - Инверсия зависимости
  - Функциональный подход
  - Прямая зависимость
  - Общий экземпляр
  - Передача функций
  - Проброс свойств
  - Изоляция
  - Лишние зависимости
created: 2025-03-02
updated: 2025-03-09
author: Vladimir Shestakov
---
# Инверсия зависимостей

Модули часто предоставляют конструктор или фабричный метод для создания экземпляра программного решения с необходимыми настройками. Перед непосредственным исполнением методов модуля необходимо создать экземпляр и знать с какими настройками его создать. Но что, если экземпляр программного решения необходим во множестве модулях проекта? Тогда в каждом модуле создавать свой экземпляр с одинаковыми настройками?

Самое простое решение — создать единственный экземпляр в отдельном модуле и экспортировать его. Другие модули смогут импортировать его и работать с одним и тем же экземпляром, не заботясь об его настройках, так как он уже будет подготовлен для работы.

```ts
const httpClient = createHttpClient({ baseUrl: '/api/v1' });

export httpClient;
```

```ts
import { httpClient } from './http-client';

const response = await httpClient.request('/data');
```

Это решение кажется простым и удобным, однако образует **жёсткую связь**. Из-за импорта общих экземпляров возникнет сложность повторного и изолированного исполнения модулей в многопользовательской среде (обработке запросов или в многооконном интерфейсе), где различные задачи выполняются одними и теми же модулями, но с разным состоянием. Для изолированного исполнения нельзя применять импорт общих экземпляров, если их состояние может меняться.  Заранее подготовить несколько общих экземпляров — неприемлемое решение, ведь оно приведёт только к увеличению жёстких связей и усложнит логику импорта нужного экземпляра. Код станет запутанным.

Для решения проблем необходимо вывернуть "прямую связь" в обратную сторону. Чтобы модуль не импортировал зависимые экземпляры, а принимал их от вызывающей стороны. Модуль можно рассматривать как функцию, в аргументы которой передаются зависимости. Обычный **функциональный подход**. Вызывающая сторона будет решать когда и с какими экземплярами исполняться модулю, и в зависимости от ситуации передавать модулю разные экземпляры (но сопоставимые по типу). Импорты не понадобятся, модуль избавится от жестких связей и его можно будет повторно использовать в разном окружении для разных задач. Передача зависимостей вместо прямого их импорта называется **инверсией зависимостей**. 

Стоит уточнить, что через аргументы нужно передавать **все зависимости**. Это касается не только данных, от которых непосредственно зависит результат функции, но и **способа вычисления**, который также влияет на результат. Использование функции само по себе не решает проблему жёстких связей, ведь остаётся возможность вызова одной функции внутри другой. Внутренняя функция скорее всего будет напрямую импортироваться, что оставит скрытую жёсткую связанность.

Чтобы полностью избежать жёсткой связанности, необходимо передавать не только данные, но и используемые функции в качестве аргументов. Однако важно не доводить этот подход до абсурда. Нужно чётко понимать, какая **обязанность** у функции. Всё, что касается её ключевой внутренней логики, допустимо импортировать или реализовывать внутри самой функции. Остальные зависимости лучше получать через аргументы.

Классический пример функционального подхода с инверсией зависимостей — сортировка элементов массива. Функция `sort` перебирает элементы и сравнивает их, но алгоритм сравнения элементов не реализуется внутри функции сортировки и не импортируется функцией `sort` самостоятельно. Вместо этого алгоритм сравнения передаётся через аргументы в виде функции обратного вызова (callback). При этом алгоритм сортировки остаётся в обязанностях функции `sort` и не передаётся в аргументах. Это позволяет гибко применять функцию `sort` для разных задач с разными типами данных, но не делегировать избыточные обязанности вызывающей стороне.

```ts
function compareNumbers(a, b) {
  return a - b;
}

[4, 2, 5, 1, 3].sort(compareNumbers)
```

Инверсия зависимостей значительно упрощает изолированное исполнение модуля и его автоматическое тестирование. Так как модуль не привязывается жёстко к конкретной реализации зависимых модулей и, соответственно, не привязан к конкретному окружению. 

Но остаётся вопрос с общими экземплярами — кто и когда их создаёт, чтобы передать их в аргументы в качестве зависимостей? Распространяется ли инверсия зависимостей на вызывающую сторону? Ведь вызывающая сторона тоже является модулем. Кто-то все равно будет импортировать используемые в проекте модули.

Исключения из правил не нужны. Прямой импорт заменяется на передачу зависимостей, если предполагается использование модуля с другими зависимостями, предполагается его расширение или за счёт инверсии модуль избавляется от лишней обязанности и тем самым становится и проще и гибче.

Общие экземпляры, например http-клиент, сервис мультиязычности или состояние приложения (хранилища), можно создать в корне  приложения. В корне приложения будет прямой импорт основных модулей и передача в исполняемые функции подготовленных экземпляров. Вызванные функции смогут передать экземпляры другим функциям (модулям). 

```tsx
import { render } from './render-library';
import { httpClient } from './http-client';
import { createStore } from './state-library';
import App from './app';

const httpClient = createHttpClient({ baseUrl: '/api/v1' });
const store = createStore({ httpClient });

render(<App store={store}/>);
```

В примере выше импорт модулей осуществлен в корневом файла проекта. В корневом файле также создаются основные экземпляры — `httpClient`, `store` и react-элемент `<App>`. Модуль `state-library` не импортирует `./http-client`, а модуль компонента `App` не импортирует `./state-library` — всё передаётся в их аргументы (свойства).

Но представьте дерево вызова всех вложенных функций приложения, которых в примере не видно... Из-за того что какой-то вложенной функции понадобится экземпляр некого модуля, его придётся подготовить в корне приложения и пробросить сначала в свойства компонента и далее через всю цепочку вызова вложенных функций! У множества вышестоящих функций появится дополнительная зависимость только ради её проброса вложенным функциям.

Проблему проброса свойств (Props Drilling) попробуем решить в следующей теме.

[Содержание](main.md)