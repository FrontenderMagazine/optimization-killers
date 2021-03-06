# Убийцы оптимизации

## Введение

В этой статье я расскажу о том, как избегать написания кода, который будет работать
значительно хуже, чем вы думаете. В частности, покажу антипаттерны, которые не позволяют V8 
(также относится к Node.js, Opera и Chromium) оптимизировать ваши функции.

### Предыстория о V8

В движке V8 нет интерпретаторов, но существует два типа компиляторов: общий и
оптимизирующий. Это приводит к тому, что JavaScript всегда компилируется и запускается
точно так же, как и машинный код. Это означает, что код работает быстро? Нет. Сама по себе
компиляция не ускоряет работу кода. Оно позволяет избежать накладных расходов,
вызванных работой интерпретатора, но неоптимизированный код будет работать медленно.

Например, в обычном компиляторе выражение `a + b` превратится во что-то вроде:

    mov eax, a
    mov ebx, b
    call RuntimeAdd

Другими словами, этот код будет каждый раз вызывать функцию. С другой стороны, если `a`
и `b` всегда будут целыми, то скомпилированный код будет другим:

    mov eax, a
    mov ebx, b
    add eax, ebx

И новая версия кода будет выполняться в разы быстрее, чем вызов сложной JavaScript-
функции.

В общем, после работы общего компилятора вы получаете сырой машинный код, затем 
результат передаётся оптимизирующему компилятору. Код на выходе оптимизирующего
компилятора с легкостью может быть, скажем, в 100 раз быстрее, чем после
первоначальной компиляции. Но здесь есть подводный камень: вы не можете писать какой
угодно JavaScript-код и быть уверены, что он будет оптимизирован. Существует множество
паттернов, некоторые даже основополагающие, при использовании которых в JS
оптимизирующий компилятор будет отказываться его оптимизировать.

Важно заметить, что эти паттерны обладают катастрофическим эффектом для всей функции, в
которой содержатся. Код оптимизируется только по одной функции в каждый момент
времени без учета того, что делает остальной код (если только этот код заинлайнен в
текущую оптимизируемую функцию).

Эта статья покрывает большинство антипаттернов, вызывающих «деоптимизационный ад».
Со временем антипаттерны могут меняться, и предложенные пути решения могут стать
необязательными, когда оптимизирующий компилятор станет умнее и научится распознавать
антипаттерны.


## Темы

1. [Инструменты](#1-tooling)
2. [Неподдерживаемый синтаксис](#2-unsupported-syntax)
3. [Обработка `arguments`](#3-managing-arguments)
4. [Switch-case](#4-switch-case)
5. [For-in](#5-for-in)


## 1. Инструменты

Вы должны уметь использовать Node.js с несколькими включёнными флагами V8, чтобы
понять, как антипаттерны влияют на оптимизацию. В общих чертах, всё выглядит так: вы
создаёте функцию, содержащую паттерн, вызываете её всеми возможными способами, затем
вызываете внутренние методы V8 для оптимизации и анализируете результат.

_test.js:_

    // Функция с антипаттерном для исследования (выражение `with`)
    function containsWith() {
        return 3;
        with({}) {}
    }
     
    function printStatus(fn) {
        switch(%GetOptimizationStatus(fn)) {
            case 1: console.log("Функция оптимизирована"); break;
            case 2: console.log("Функция не оптимизирована"); break;
            case 3: console.log("Функция всегда оптимизируема"); break;
            case 4: console.log("Функция никогда не оптимизируема"); break;
            case 6: console.log("Функция возможно деоптимизирована"); break;
        }
    }
     
    // вызовите функцию
    containsWith();
     
    %OptimizeFunctionOnNextCall(containsWith);
    // Следующий вызов
    containsWith();
     
    // Проверка
    printStatus(containsWith);

Запустите файл:

    $ node --trace_opt --trace_deopt --allow-natives-syntax test.js
    Функция не оптимизирована

Чтобы проверить правильность работы, закомментируйте выражение `with` и вызовите
скрипт ещё раз:

    $ node --trace_opt --trace_deopt --allow-natives-syntax test.js
    [optimizing 000003FFCBF74231 <JS Function containsWith (SharedFunctionInfo 00000000FE1389E1)> - took 0.345, 0.042, 0.010 ms]
    Функция оптимизирована

Важно использовать этот инструмент для подтверждения того, что обходные решения
работают и необходимы.


## 2. Неподдерживаемый синтаксис

Некоторые конструкции не поддерживаются оптимизирующим компилятором, что приводит к
тому, что функция, в которой используется такой синтаксис, не может быть оптимизирована.

**Очень важно отметить**, что даже если конструкция никогда не будет выполняться, она
одним только фактом своего существования выключает оптимизацию для всей функции.

К примеру, этот хак не спасёт положения:

    if (DEVELOPMENT) {
        debugger;
    }

Код выше не позволит оптимизировать функцию, в которой находится, даже если выражение `debugger;` никогда
не будет выполняться.

На данный момент не оптимизируются:

- Генераторы,
- Функции, содержащие выражение `for-of`,
- Функции, содержащие выражение `try-catch`,
- Функции, содержащие выражение `try-finally`,
- Функции, содержащие присваивание с помощью `let`,
- Функции, содержащие присваивание с помощью `const`,
- Функции, содержащие литералы объекта с использованием `__proto__`, `get` или `set` конструкций.

Никогда не оптимизируются:

- Функции, содержащие выражение `debugger`,
- Функции, непосредственно вызывающие `eval()`,
- Функции, содержащие выражение `with`.

Чтобы было ясно насчёт последних утверждений: функция недоступна для оптимизации,
если сделать что-нибудь подобное:

    function containsObjectLiteralWithProto() {
        return {__proto__: 3};
    }
     
    function containsObjectLiteralWithGetter() {
        return {
            get prop() {
                return 3;
            }
        };
    }
     
    function containsObjectLiteralWithSetter() {
        return {
            set prop(val) {
                this.val = val;
            }
        };
    }

Прямое использование `eval` и `with` заслуживает отдельного внимания, так как их
использование влечёт за собой применение динамической области видимости относительно
всего, к чему они относятся, поэтому есть возможность при оптимизации сломать
множество других функций, так как невозможно наверняка знать, какая переменная
за что отвечает.

**Обходные решения**

Некоторые из упомянутых выше выражений сложно избегать в конечном коде, например,
конструкции `try-finally` и `try-catch`. Чтобы снизить влияние этих конструкций 
на производительность, необходимо изолировать их в минимальной функции, чтобы не
затронуть основной код:

    var errorObject = {value: null};
    function tryCatch(fn, ctx, args) {
        try {
            return fn.apply(ctx, args);
        } catch(e) {
            errorObject.value = e;
            return errorObject;
        }
    }
     
    var result = tryCatch(mightThrow, void 0, [1,2,3]);
    // Однозначно выясним, произошло ли исключение
    if (result === errorObject) {
        var error = errorObject.value;
    } else {
        // результат и есть возвращённое значение
    }

## 3. Обработка `arguments`

Существует несколько способов использовать `arguments` так, чтобы не допустить
оптимизацию функции. Каждый должен быть предельно осторожен с `arguments`.

#### 3.1. Переопределение передаваемого параметра вместе с упоминанием `arguments` в теле функции.

    function defaultArgsReassign(a, b) {
        if (arguments.length < 2) b = 5;
    }

**Обходное решение:** сохранить параметр в новую переменную:

    function reAssignParam(a, b_) {
        var b = b_;
        // b, в отличие от b_, может быть безопасно переприсвоено
        if (arguments.length < 2) b = 5;
    }

Если это единственная причина использования `arguments`, то этот фрагмент кода 
часто можно заменить проверкой на `undefined`:

    function reAssignParam(a, b) {
        if (b === void 0) b = 5;
    }

Но имейте в виду, если есть вероятность, что `arguments` будут добавлены и использованы в функции 
позднее, то в процессе очень легко забыть добавить переприсваивание.


#### 3.2. Утечка arguments:

    function leaksArguments1() {
        return arguments;
    }
     
    function leaksArguments2() {
        var args = [].slice.call(arguments);
    }
     
    function leaksArguments3() {
        var a = arguments;
        return function() {
            return a;
        };
    }

Объект `arguments` не должен никуда передаваться или «утекать» как либо ещё.

**Обходное решение:** для безопасной передачи `arguments` создайте новый массив:

    function doesntLeakArguments() {
        // .length просто ещё одно число, поэтому
        // весь объект `arguments` не утекает
        var args = new Array(arguments.length);
        for(var i = 0; i < args.length; ++i) {
            // i всегда валидный индекс для объекта `arguments`
            args[i] = arguments[i];
        }
        return args;
    }

Это решение слишком большое и громоздкое, и стоит несколько раз подумать, стоит ли игра
свеч. Как вы видите, оптимизация всегда требует большого количества кода, а значит 
снижается читаемость.

С другой стороны, если у вас есть этап сборки, желаемого эффекта можно достичь с помощью
макроса, что не потребует использования карт кода (source map) и позволит исходному коду оставаться
валидным javascript-кодом:

    function doesntLeakArguments() {
        INLINE_SLICE(args, arguments);
        return args;
    }

Эта техника используется в bluebird, и в результате на этапе сборки код разворачивается
в следующий:

    function doesntLeakArguments() {
        var $_len = arguments.length;var args = new Array($_len); for(var $_i = 0; $_i < $_len; ++$_i) {args[$_i] = arguments[$_i];}
        return args;
    }

#### 3.3. Присваивание аргументам:

Это может случиться при небрежном написании кода:

    function assignToArguments() {
        arguments = 3;
        return arguments;
    }

**Обходное решение**: нет нужды в таком идиотском коде. В любом случае, такой код в
строгом режиме будет вызывать исключение.

#### Как же безопасно использовать `arguments`?

Используйте только

- `arguments.length`,
- `arguments[i]` **при этом `i` должен быть валидным целым индексом для `arguments`**,
- Никогда не используйте `arguments` напрямую без `.length` или `[i]`.
(ТОЛЬКО `x.apply(y, arguments)` МОЖНО использовать, ничего из оставшегося нельзя, 
например нельзя использовать `.slice`. `Function#apply` — особенный случай).

Стоит заметить, что пока вы пользуетесь вышеупомянутыми безопасными способами, 
все страшилки про то, что любое упоминание `arguments` приводит к выделению 
дополнительной памяти остаются лишь страшилками.


## 4. Switch-case

На данный момент выражение `switch-case` может включать до 128 веток `case` и
оставаться оптимизируемой. Любая функция содержащая больше веток `switch` 
оптимизирована не будет.

    function over128Cases(c) {
        switch(c) {
            case 1: break;
            case 2: break;
            case 3: break;
            …
            case 128: break;
            case 129: break;
        }
    }

Поэтому старайтесь учитывать количество веток `case` и держать его на уровне 128 или
менее, используя массивы функций или `if-else`.

## 5. For-in

Выражения `for-in` может помешать оптимизации в нескольких случаях.

Все они дают повод думать о том, что «For-In медленный».

#### 5.1. Переменная key не локальная:

    function nonLocalKey1() {
        var obj = {}
        for(var key in obj);
        return function() {
            return key;
        };
    }
     
    var key;
    function nonLocalKey2() {
        var obj = {}
        for(key in obj);
    }

Итак, переменная key не может быть взята из соседних пространств имён (как верхних,
так и нижних). Необходима только локальная переменная.


#### 5.2. Итерируемый объект не является «простым перечисляемым»

##### 5.2.1. Объекты в режиме «хэш-таблицы» («нормализованные объекты» или режим «словаря» — объекты, которые имеют хэш-таблицу в своём основании) не являются простыми перечисляемыми.

    function hashTableIteration() {
        var hashTable = {"-": 3};
        for(var key in hashTable);
    }

Объект переходит в режим хэш-таблицы, к примеру, при добавлении большого количества
свойств динамически (вне конструктора), удаления свойств с помощью `delete`,
использовании невалидных идентификаторов для свойств объекта и так далее. Другими
словами, если вы используете объект как хэш-таблицу, он превращается в хэш-таблицу.
Использование такого объекта в `for-in` — худшее, что вы можете сделать. Вы можете
проверить, находится ли объект в режиме хэш-таблицы, запустив
`console.log(%HasFastProperties(obj))` с включённым в Node.JS
флагом `--allow-natives-syntax`.

---------------------------------------

##### 5.2.2. Объект имеет перечисляемые свойства в цепочке прототипов

    Object.prototype.fn = function() {};

Это действие добавит перечисляемое свойство в цепочку прототипов всех объектов (за
исключением созданных с помощью `Object.create(null)`). Любая функция, содержащая
выражение `for-in` после этого станет неоптимизируемой (если только не итерируется по
`Object.create(null)` объектам)

Вы можете создавать неперечисляемые свойства с помощью `Object.defineProperty` (не
рекомендуется делать в реальном времени, но отлично подходит для определения
статических штук наподобие свойств прототипа).

---------------------------------------

##### 5.2.3. Объект содержит перечисляемый индекс массива

Возможность для свойства быть индексом массива описана в [спецификации ecmascript][1]

> Имя свойства P (в форме строкового значения) является *индексом массива* (array
> index) только в том случае, если ToString(ToUint32(P)) равно P, а ToUint32(P)
> не равно 2<sup>32</sup>−1. Если имя свойства представляет собой индекс массива, такое свойство
> также называется *элементом* (element).

Обычно так обращаются к элементам массива, но таким же образом можно обращаться к
свойствам и атрибутам объектов: `normalObj[0] = value;`

    function iteratesOverArray() {
        var arr = [1, 2, 3];
        for (var index in arr) {
            //…
        }
    }

Поэтому итерирование по массиву с помощью `for-in` не только медленнее, чем обычный
цикл, но и не позволяет оптимизировать функцию, в которой он находится.

---------------------------------------

Итак, если вы будете использовать в `for-in`-цикле объект, не являющийся простым
перечисляемым, это скажется на всей содержащей его функции.

**Обходное решение:** Всегда используйте `Object.keys` и итерируйтесь по полученному
массиву с помощью обычного цикла. Если всё же вам нужны все свойства из всей цепочки
прототипов, создайте изолированную функцию-помощник:

    function inheritedKeys(obj) {
        var ret = [];
        for(var key in obj) {
            ret.push(key);
        }
        return ret;
    }


[1]: http://www.ecma-international.org/ecma-262/5.1/#sec-15.4




