## Контейнер DI

Контейнер DI управляет зависимостями. Именно контейнер разбирается во всех разновидностях инъекций и в особенностях токенов. В контейнер добавляются инъекции всех программных решений приложения. Контейнер запоминает инъекции и исполняет их по мере необходимости. Контейнер предоставляет доступ ко всем решениям приложения, беря на себя обязанность создания и инициализации решения (через исполнение инъекции). Контейнер запоминает подготовленное решение, чтобы не создавать и не инициализировать его повторно при повторном запросе решения из контейнера. Контейнер реализует паттерн "Одиночка".
### Добавление инъекции

`set(inject)`

В метод set передаётся инъекция одного из вида `InjectFactory`, `InjectClass`, `InjectValue`. Или массив инъекций.

```ts
set<Type, ExtType extends Type, Deps>(inject: InjectFactory<Type, ExtType, Deps>): this;  
set<Type, ExtType extends Type, Deps>(inject: InjectClass<Type, ExtType, Deps>): this;  
set<Type, ExtType extends Type>(inject: InjectValue<Type, ExtType>): this;  
set(inject: InjectArray): this;
```

Можно передать заранее подготовленную инъекцию или описать её непосредственно в методе.

```ts
container.set({
  token: FIRST,
  depends: { logger: LOGS, config: FIRST_CFG },
  constructor: First
})
```

Чтобы не вызывать метод `set()` для каждой внедряемой инъекции, можно передать все инъекции в массиве. Но инъекции должны быть заранее подготовлены (иначе не будут корректно выводиться типы зависимостей)! 

```ts
container.set([configs, renderService, routerService, modalsService, httpClient, i18nService, logService])
```

Если инъекция не подготовлена, но её нужно передать в массив, то можно воспользоваться утилитой `injectFactory()`, `injectClass()` или `injectValue()`. То есть нужно подготовить инъекцию, но сделать это можно и на месте внедрения.

```ts
container.set([
  configs, 
  renderService,
  //... 
  injectClass({
    token: FIRST,
    depends: { logger: LOGS, config: FIRST_CFG },
    constructor: First
  })
])
```

Весь массив инъекций также можно подготовить заранее.  Например, в модуле приложения можно определить инъекцию-массив со всеми инъекциями на сервисы, компоненты, апи и ресурсы модуля.

```ts
// Инъекция модуля со всеми его составными частями (тоже инъекциями)
export const catalogFeature = [  
  articlesApi,  
  categoriesApi,  
  injectTranslations,  
  articlesStore,  
  categoriesStore,  
];
```

В контейнер тогда будет внедряться одна инъекция на один модуль. И получается, что в метод set можно передать массив массивов инъекций любой вложенности. 

```ts
// Подключение модуля catalogFeature с массивом инъеккций в приложение
container.set([configs, renderService, catalogFeature])
```
### Выбор решения по токену

`get(token)`

Любое решение можно выбрать из контейнера по токену методом `get(token)`. Если выбор решения осуществляется впервые, то оно будет создано, инициализировано и возвращено. Если осуществляется повторная выборка решения, то  возвращается ранее созданное созданное. 

```ts
async get<Type>(token: TokenInterface<Type>): Promise<Type>
```

Обратите внимание, что метод `get(token)` асинхронный - возвращается промис. Возможна ситуация, когда повторная выборка решения осуществляется при ещё не выполненной первой выборки. Но контейнер не будет все равно повторно создавать и инициализировать решение - вернется ожидание (promise) от первой выборки. Все получат в конечном итоге один и тот же экземпляр решения.
### Выбор решений по карте токенов

`getMapped`

```ts
async getMapped<Deps extends Record<string, TokenInterface>>(depends: Deps): Promise<TypesFromTokens<Deps>>
```

Выбор множества сервисов по указанной карте токенов.  Сервисы будут возвращены под теми же ключами, под которыми указаны токены в depends.
### Синхронный выбор решения для Suspense

`getWithSuspense`

```ts
getWithSuspense<Type>(token: TokenInterface<Type>): Type
```

Выбор сервиса с логикой ожидания для `<Suspense>` (с приостановкой через выброс исключения)  
Если сервис ещё не создан, то запоминается обещание (promise) его создания  
Если ожидание ещё не завершено, то в исключение кидается обещание создания сервиса  
Если ожидание выполнено, то возвращается результат обещания - экземпляр сервиса
### Синхронный выбор множества решений по карте токенов для Suspense

`getMappedWithSuspense`

```ts
getMappedWithSuspense<Deps extends Record<string, TokenInterface>>
```

Выбор множества сервисов по карте токенов с логикой ожиданием для `<Suspense>`
Сервисы будут возвращены под теми же свойствами, под которыми указаны их токены в depends.  
Исключение кидается пока не будут выполнены асинхронные выборки всех сервисов.  
В исключение кидается последние невыполненное обещание, чтобы попытаться все сервисы выбрать за раз.
### Хуки на выборки решений по токену

`useSolution`

```ts
useSolution<Type>(token: Token<Type>): Type
```

Хук для выборки сервиса из DI контейнера по токену.  
В приложении должен использовать компонент `<Suspense>` так как хук выкидывает исключения на ожидания.  Если сервис не выбран, то компонент прекратит свою работу.  
 
```ts
function MyCompoentn () {
  const modals = useSolution(MODALS);
  const open = () => modals.open(CONFIRM_MODAL)
  return (
    <div onClick={openConfirm}>Отправить</div>
  )
}
```

 `useSolutionMap`

```ts
useSolutionMap<Deps extends Record<string, Token>>
```
 
 Хук для выбора множества сервисов из DI по указанной карте токенов (`Map`). Сервисы будут возвращены под теми же ключами, под которыми указаны токены в depends. * В приложении должен использовать компонент `<Suspense>` так как хук выкидывает исключения на ожидания. 
 
 ```ts  
const { i18n, store } = useSolutionMap({i18n: I18N_TOKEN, store: STORE_TOKEN})
 ```

`useSolutionPending`

```ts
useSolutionPending<Type>(token: Token<Type>): {  
  service: Type | undefined;  
  isSuccess: boolean;  
  isWaiting: boolean;  
  isError: boolean;  
}
```

Хук для выборки сервиса из DI контейнера по токену с признаками ожидания/успеха/ошибки.  
Для хука не нужен Suspense, хук не выкидывает исключения.  

Если сервис ещё не готов (асинхронно создаётся), то не будет возвращен,  но будут возвращен признак `isWaiting = true`.  
При готовности сервиса (или ошибки его подготовки) хук заставит компонент перерендериться и  
при следующем выполнении вернет сервис с признаком `isSuccess = true`.  
Если будет ошибка подготовки сервиса, то он не вернется, но будет признак `isError = true`

```ts  
const i18n = useSolutionPending(I18N_TOKEN);

console.log(i18n.instance);
console.log(i18n.isSuccess); 
сonsole.log(i18n.isWaiting); 
console.log(i18n.isError);

if (i18n.instance) {  
  // Действия если сервис получен  
}  
```
### Сброс решения (удаление экземпляра)

```ts
async deleteValue<Type>(token: TokenInterface<Type>): Promise<void>
```

Удаление ранее созданного экземпляра сервиса. 
### Токен на контейнер

`CONTAINER`

Иногда нужна зависимость на сам контейнер, для этого существует токен на контейнер. Фактически контейнер сам в себя добавляет инъекцию на себя. 

```ts
const container = useSolution(CONTAINER);
```
### События контейнера

`events`

Контейнер отправляет события на создание экземпляра и его удаление. На события можно подписаться.

```ts
container.events.on('onCreate', <Type extends object>({ token, value }: { token: Token<Type>; value: Type }) => {
  // обработка события
}
```

```ts
container.events.on('onDelete', ({ token }: { token: Token }) => {
  // обработка события
}
```

Для работы с событиями используется класс `Events` из react-solution

← [Инъекция](dependency_management/injection.md) | .. →

