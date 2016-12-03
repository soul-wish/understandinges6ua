# Множини та мапи

Більшу частину своєї історії JavaScript мав лише один тип колекцій — тип `Array` (хоча дехто може стверджувати, що всі об’єкти–немасиви є просто колекціями пар “ключ–значення”, їх планували використовувати зовсім іншим чином). Масиви використовуються в JavaScript точно так само, як і масиви в інших мовах, проте недоліком масивів, порівняно з іншими колекціями, була в тому, що вони часто використовувались в якості черг (queues) та стеків (stacks). Оскільки масиви мають лише числову індексацію, розробники використовували об’єкти–немасиви тоді, коли були необхідні нечислові індекси. Такий підхід породив власні реалізації множин (sets) та мап (maps) з допомогою об’єктів–немасивів.

*Множина (set)* — це список значень, що не може містити повторень. На відміну від масивів, у множині зазвичай вам не потрібно отримувати доступ до окремих елементів. Замість цього, набагато частіше є потреба просто перевірити чи значення присутнє в множині, чи ні. *Мапа (map)* — це колекція ключів, що відповідають певним значенням. Таким чином, кожен елемент у мапі зберігає дві частинки даних, а значення отримуються через певний ключ, з якого потрібно читати. Мапи часто використовують в якості кешу, для збереження даних, як потрібно згодом швидко отримати. Формально ECMAScript 5 не має множин та мап, проте розробники обходять це обмеження використовуючи об’єкти–немасиви.

ECMAScript 6 додає множини та мапи у JavaScript і у цьому розділі про все, що вам потрібно знати про ці два типи колекцій.

Для початку, я розповім про обхідні шляхи, які розробники використовували для реалізації множин та мап до ECMAScript 6, і чому ці реалізації мали проблеми. Після цього, я покажу, як множини та мапи працюють у ECMAScript 6.

## Множини та мапи у ECMAScript 5

У ECMAScript 5 розробники імітували множини та мапи з використанням властивостей об’єктів, ось так:

```js
let set = Object.create(null);

set.foo = true;

// перевірка на існування
if (set.foo) {

    // щось виконуємо
}
```

Змінна `set`, у цьому прикладі, є об’єктом з прототипом `null`, для певності у тому, що об’єкт не має жодних успадкованих властивостей. Використання властивостей об’єктів в якості унікальних значення для перевірки є звичною практикою у ECMAScript 5. Якщо властивість додається у об’єкт `set`, їй встановлюється значення `true`, тому умовні оператори (як от оператор `if` у цьому випадку) можуть легко перевірити, чи це значення вже є у множині.

Єдина дійсна відмінність між об’єктом, що використовується як множина та об’єктом, що використовується як мапа у значенні, яке зберігається. До прикладу, цей приклад використовує об’єкт як мапу:

```js
let map = Object.create(null);

map.foo = "bar";

// отримання значення
let value = map.foo;

console.log(value);         // "bar"
```

Цей код зберігає значення `"bar"` під ключем `foo`. На відміну від множин, мапи частіше використовуються для отримання інформації, ніж для перевірки існування ключів.

## Проблеми з обхідними реалізаціями

Використання об’єктів у якості множин та мап працює добре для простих випадків, проте такий підхід може стати більш складним, як тільки ви заглибитесь у обмеження об’єктних властивостей. Наприклад, оскільки всі властивості об’єктів повинні бути рядками, ви маєте бути певні, що два ключі не будуть одним і тим же рядком. Розгляньте наступне:

```js
let map = Object.create(null);

map[5] = "foo";

console.log(map["5"]);      // "foo"
```

У цьому прикладі рядкове значення `"foo"` присвоюється числовому ключу `5`. Внутрішньо, це числове значення конвертується у рядок, тому `map["5"]` та `map[5]` насправді посилатимуться на одну і ту ж властивість. Така внутрішня поведінка може спричинити помилку, якщо ви захочете використовувати і рядкові, і числові ключі водночас. Інша проблема виникає при використанні об’єктів у якості ключів:

```js
let map = Object.create(null),
    key1 = {},
    key2 = {};

map[key1] = "foo";

console.log(map[key2]);     // "foo"
```

Тут `map[key2]` та `map[key1]` посилаються на те саме значення. Об’єкти `key1` та `key2` конвертуються у рядки, тому що властивості об’єктів мають бути рядками. Оскільки `"[object Object]"` є рядковим представленням за замовчуванням для об’єктів, як `key1`, так і `key2` конвертуються у цей рядок. Це може спричинити помилки через свою неочевидність, тому що логічно припустити, що різні об’єкти–ключі мали б бути різними.

Перетворення у рядкове представлення за замовчуванням ускладнює використання об’єктів у якості ключів. (Така ж проблема виникає при спробі використовувати об’єкт у якості множини.)

Мапи з ключами, значення яких є хибними, мають свою власну проблему. Хибні значення зводяться до false у випадках, коли потрібне булеве значення, як от в умові оператора `if`. Таке приведення не є проблемою, проте вам слід бути обережними при використанні таких значень. Наприклад, подивіться на цей код:

```js
let map = Object.create(null);

map.count = 1;

// перевірка існування "count", чи ненульового значення?
if (map.count) {
    // ...
}
```

Цей приклад має деяку невизначеність стосовно того, як слід використовувати `map.count`. Оператор `if` призначений для перевірки існування `map.count`, чи перевірки того, що ця властивість має ненульове значення? Код всередині оператора `if` буде виконуватись тому, що значення 1 є істинним значенням. Однак, якщо б `map.count` дорівнювало 0, або якщо б `map.count` не існувало, код всередині оператора `if` не виконався би.

Ці проблеми важко помітити та полагодити у великих додатках і це одна з основних причин, чому ECMAScript 6 вводить у мову множини та мапи.

I> JavaScript має оператор `in`, що повертає `true`, якщо властивість існує в об’єкті без читання його значення з об’єкту. Однак, оператор `in` також шукає за прототипом об’єкта, що робить його небезпечним для використання, якщо прототипом об’єкта не є `null`. Хоча навіть так багато розробників продовжують використовувати помилковий код з попереднього прикладу частіше ніж оператор `in`.

## Множини (Sets) у ECMAScript 6

ECMAScript 6 додає тип `Set`, що впорядковує список значень без повторень. Множини дають швидкий доступ до даних, які вони містять, додаючи більш ефективний спосіб оперувати дискретними значеннями.

### Створення множин та додавання елементів

Sets створюються через `new Set()`, а метод `add()` додає у множину елементи. Ви можете подивитись, скільки значень містить множина через властивость `size`:

```js
let set = new Set();
set.add(5);
set.add("5");

console.log(set.size);    // 2
```

Множини не приводять значення для визначення того, чи вони однакові. Це означає, що множина може містити як і число `5`, так і рядок `"5"` як два різних елементи. (Внутрішньо, для порівняння двох значень на ідентичність  використовується метод `Object.is()`, який ми обговорювали у Главі 4) Ви також можете додати кілька об’єктів у множину, і вони будуть окремими елементами:

```js
let set = new Set(),
    key1 = {},
    key2 = {};

set.add(key1);
set.add(key2);

console.log(set.size);    // 2
```

Оскільки `key1` та `key2` не конвертуються у рядки, вони вважаються двома унікальними елементами множини. (Пам’ятайте, якщо б `key1` та `key2` конвертувались у рядки, вони були би рівні `"[Object object]"`.)

Якщо метод `add()` викликається з одним і тим самим значенням, всі виклики після першого будуть проігноровані:

```js
let set = new Set();
set.add(5);
set.add("5");
set.add(5);     // повторення - воно ігнорується

console.log(set.size);    // 2
```

Ви можете ініціалізувати множину через масив, і конструктор `Set` перевірить, щоб використовувались лише унікальні значення. Наприклад:

```js
let set = new Set([1, 2, 3, 4, 5, 5, 5, 5]);
console.log(set.size);    // 5
```

У цьому прикладі, масив зі значеннями, що повторюються використовується для ініціалізації множини. Число `5` з’являється у множині лише один раз, хоча у масивів воно зустрічається чотири рази. Такий функціонал робить простішим перетворення існуючого коду, або JSON–структур у множини.

I> Конструктор `Set` насправді приймає в якості аргументу ітерабельний об’єкт. Масиви працюють тому, що вони є ітерабельними об’єктами за замовчуваннями, як і множини та мапи. Конструктор `Set` використовує ітератор, щоб отримати значення з аргументу. (Про ітерабельні об’єкти та ітератори піде мова у Главі 8.)

Ви можете перевірити, чи значення є у множині через метод `has()`, ось так:

```js
let set = new Set();
set.add(5);
set.add("5");

console.log(set.has(5));    // true
console.log(set.has(6));    // false
```

Тут `set.has(6)` поверне false, оскільки множина не має такого значення.

### Видалення значень

Також можливо видаляти значення з множини. Ви можете видалити одне значення, використовуючи метод `delete()`, або видалити всі значення з множини викликом методу `clear()`. Ось як вони працюють:

```js
let set = new Set();
set.add(5);
set.add("5");

console.log(set.has(5));    // true

set.delete(5);

console.log(set.has(5));    // false
console.log(set.size);      // 1

set.clear();

console.log(set.has("5"));  // false
console.log(set.size);      // 0
```

Після виклику `delete()`, видаляється `5`; після виклику методу `clear()`, множина`set` стає порожньою.

Кожен з них дає нам простий механізм для операцій над унікальними впорядкованими значеннями. Однак, що якщо нам потрібно додати елементи у множину, а тоді виконати якусь операцію над кожним з них? Саме для такого випадку нам потрібен метод `forEach()`.

### Метод forEach() для множин

Якщо ви часто працюєте з масивами, скоріш за все ви вже знайомі з методом `forEach()`. ECMAScript 5 додав `forEach()` до масивів, щоб простіше виконувати операції над кожними елементом масиву без використання циклу `for`. Метод став популярним серед розробників, і тому такий самий метод доступний для множин та працює таким же чином.

Метод `forEach()` приймає функцію зворотнього виклику, що приймає три аргументи:

1. значення наступної позиції у множині;
1. те саме значення, що і у першому аргументі;
1. множину, з якої читаються значення.

Дивна відмінність між версією `forEach()` для множини та версією для масиву полягає в тому, що перший та другий елемент функції зворотнього виклику є однаковими. Це може виглядати як помилка, проте є хороше пояснення такої поведінки.

Інші об’єкти, що мають метод `forEach()` (масиви та мапи) передають по три аргументи у свої функції зворотнього виклику. Першим два аргументи для множин та для мап є значенням та ключем (числовим індексом у масивах).

Однак, множини не мають ключів. Люди, що розробляли стандарт ECMAScript 6 могли зробити функцію зворотнього виклику в `forEach()` для множин, що приймала б два аргументи, проте це вона відрізнялася б від двох інших. Замість цього, вони знайшли спосіб зберегти функцію зворотнього виклику однаковою для всіх, і щоб вона приймала три аргументи: кожне значення у множині розглядається як ключ і значення одночасно. Таким чином, перший та другий аргументи у `forEach()` для множин завжди однакові, щоб зберегти їхню функціональність цілісною з методами `forEach()` для масивів та мап.

Крім аргументів, все інше працює у `forEach()` для множин так само, як і для масивів. Ось трошки коду, що демонструє цей метод у дії:

```js
let set = new Set([1, 2]);

set.forEach(function(value, key, ownerSet) {
    console.log(key + " " + value);
    console.log(ownerSet === set);
});
```

Цей код ітерується по кожному елементі множини та виводить значення, що передаються у функцію зворотнього виклику методу `forEach()`. Щоразу, коли функція зворотнього виклику виконується, `key` та `value` є однакові, а `ownerSet` завжди дорівнює `set`. Цей код виводить:

```
1 1
true
2 2
true
```

Так само, як і для масивів, ви можете передати значення `this` в якості другого аргументу в `forEach()`, якщо потрібно використовувати `this` всередині функції зворотнього виклику:

```js
let set = new Set([1, 2]);

let processor = {
    output(value) {
        console.log(value);
    },
    process(dataSet) {
        dataSet.forEach(function(value) {
            this.output(value);
        }, this);
    }
};

processor.process(set);
```

У цьому прикладі, метод `processor.process()` викликає `forEach()` для множини та передає `this` в якості значення `this` всередині функції зворотнього виклику. Це необхідно, щоб `this.output()` правильно визначився у метод `processor.output()`. Функція зворотнього виклику `forEach()` використовує лише перший аргумент, `value`, тому інші упускаються. Ви також можете використовувати arrow–функції, щоб отримати той же результат без передачі другого аргументу, ось так:

```js
let set = new Set([1, 2]);

let processor = {
    output(value) {
        console.log(value);
    },
    process(dataSet) {
        dataSet.forEach((value) => this.output(value));
    }
};

processor.process(set);
```

Аrrow—функція у цьому прикладі читає `this` з функції `process()`, у якій вона міститься, тому `this.output()` має коректно визначатись у виклик `processor.output()`.

Запам’ятайте, що множини зручні для відслідковування значення, а `forEach()` дає вам можливість працювати з кожними значенням почергово, але ви не маєте безпосереднього доступу до значень за індексом так, як у випадку з масивами. Якщо це потрібно, тоді найкраще буде конвертувати множину у масив.

### Перетворення множини у масив

Легко конвертувати масив у множину, оскільки ви можете передати масив у конструктор `Set`. Також просто конвертувати множину назад у масив з використанням оператора розкладу (spread operator). Глава 3 розповідала про оператор розкладу (`...`) як спосіб розбити елементи масиву в окремі параметри функції. Ви також можете використовувати оператор розкладу для роботи з ітерабельними об’єктами, як от множинами, для перетворення їх у масиви. Наприклад:

```js
let set = new Set([1, 2, 3, 3, 3, 4, 5]),
    array = [...set];

console.log(array);             // [1,2,3,4,5]
```

У цьому прикладі, з масиву, що містить повторення, створюється множина. Множина видаляє повторення, а потім вона перетворюється у новий масив з допомогою оператора розкладу. Множина, сама по собі, продовжує зберігати ті самі елементи, які вона отримала при створенні (`1`, `2`, `3`, `4` та `5`). Вони просто копіюються у новий масив.

Такий підхід корисний, якщо ви вже маєте масив, і хочете створити масив, що не містить повторень. Наприклад:

```js
function eliminateDuplicates(items) {
    return [...new Set(items)];
}

let numbers = [1, 2, 3, 3, 3, 4, 5],
    noDuplicates = eliminateDuplicates(numbers);

console.log(noDuplicates);      // [1,2,3,4,5]
```

У функції `eliminateDuplicates()`, множина тимчасово використовується для фільтрації повторень перед створенням нового масиву, що не містить повторень.

### Слабкі множини (Weak Sets)

Тип `Set` також можна ще назвати сильною множиною (strong set) через те, як вона зберігає посилання на об’єкти. Об’єкт зберігається в екземплярі `Set` і це те саме, що зберігати об’єкт у змінній. Допоки посилання до цього екземпляру `Set` існує, збирач сміття не чіпатиме цей об’єкт, щоб звільнити пам’ять. Наприклад:

```js
let set = new Set(),
    key = {};

set.add(key);
console.log(set.size);      // 1

// видаляємо початкове посилання
key = null;

console.log(set.size);      // 1

// отримуємо початкове посилання
key = [...set][0];
```

У цьому прикладі, присвоєння `key` значення `null` очищує одне посилання на об’єкт `key`, проте інше залишається всередині `set`. Ви досі можете отримати `key` шляхом конвертації множини у масив з допомогою оператора розкладу та отримання першого елемента. Такий результат підходить для більшості програм, але часом краще, щоб посилання у множині зникали, якщо зникають всі інші посилання. Для прикладу, якщо ваш JavaScript–код виконується на веб–сторінці і має слідкувати за елементами в DOM, що можуть бути видалені іншим скриптом, вам не потрібно, щоб ваш код зберігав останнє посилання на DOM–елемент. (Така ситуація називається *витоком пам'яті (memory leak)*.)

Для вирішення таких проблем ECMAScript 6 також включає *слабкі множини (weak sets)*, що можуть містити лише слабкі посилання на об’єкти і не можуть містити примітивних значень. *Слабке посилання (weak reference)* на об’єкт не зупиняє збір сміття, якщо воно є єдиним посиланням на об’єкт, що залишилось.

#### Створення слабких множин

Слабкі множини створюються з допомогою конструктора `WeakSet` і мають методи `add()`, `has()` та `delete()`. Ось приклад того, як вони використовуються:

```js
let set = new WeakSet(),
    key = {};

// додаємо об’єкт у множину
set.add(key);

console.log(set.has(key));      // true

set.delete(key);

console.log(set.has(key));      // false
```

Використання слабкої множини дуже схоже на використання звичайної. Ви можете додавати, видаляти та перевіряти, чи є посилання у слабкій множині. Також можливо ініціалізувати слабку множину зі значеннями передавши ітерабельний об’єкт у її конструктор:

```js
let key1 = {},
    key2 = {},
    set = new WeakSet([key1, key2]);

console.log(set.has(key1));     // true
console.log(set.has(key2));     // true
```

У цьому прикладі, у конструктор `WeakSet` передається масив. Оскільки цей масив містить два об’єкти, ці об’єкти додаються у слабку множину. Запам'ятайте, що якщо масив міститиме значення–необ’єкти, буде кинута помилка, тому що `WeakSet` не може містити примітивних значень.

#### Ключові відмінності між типами множин

Найбільша відмінність між слабкими та звичайними множинами це слабкі посилання, що прив’язані до значення об’єкту. Ось приклад, що демонструє цю відмінність:

```js
let set = new WeakSet(),
    key = {};

// додаємо об’єкт у множину
set.add(key);

console.log(set.has(key));      // true

// видалення останнього сильного посилання на key також видаляє його з слабкої множини
key = null;
```

Після виконання цього коду, посилання на `key` у слабкій множині більше не буде доступним. Неможливо перевірити це видалення, тому що вам би знадобилось хоча б одне посилання на цей об’єкт, яке ви мали би передати у метод `has()`. Це може зробити тестування слабких множин дещо заплутаним, проте ви можете бути певні, що посилання точно було видалено рушієм JavaScript .

Ці приклади показують, що слабкі множини мають кілька спільних характеристик зі звичайними множинами, проте є кілька ключових відмінностей. Ось вони:

1. У екземплярі `WeakSet`, методи `add()`, `has()` та `delete()` кинуть помилку, якщо їм передати необ’єкт.
1. Слабкі множини не ітерабельні, і тому не можуть використовуватись у циклі `for-of`.
1. Слабкі множини не видають ніяких ітераторів (як от методи `keys()` та `values()`), і тому немає способу програмно визначити вміст слабкої множини.
1. Слабкі множини не мають методу `forEach()`.
1. Слабкі множини не мають властивості `size`.

Здавалося б обмежена функціональність слабких множин необхідна для того, щоб правильно керувати пам’яттю. Загалом, якщо потрібно лише слідкувати за посиланнями на об’єкти, вам слід використовувати слабкі множини замість звичайних.

Множини дають вам новий спосіб керування списками значень, проте вони не дуже корисні, якщо вам потрібно асоціювати додаткову інформацію з цими значеннями. Ось для чого ECMAScript 6 також додає мапи.

## Мапи в ECMAScript 6

Тип `Map` в ECMAScript 6 є впорядкованим списком пар “ключ–значення”, в яких як ключ, так і значення можуть мати будь–який тип. Еквівалентність ключів визначається через `Object.is()`, тому ви можете мати одночасно ключ `5` та ключ `"5"`, тому що це різні типи. Це дещо відрізняється від використання властивостей об’єктів у якості ключів, оскільки властивості об’єктів завжди приводять значення у рядки.

Ви можете додавати елементи у мапи викликом методу `set()` та передачею йому ключа та значення, що асоціюється з цим ключем. Потім ви можете дістати значення передавши ключ у метод `get()`. Наприклад:

```js
let map = new Map();
map.set("title", "Understanding ES6");
map.set("year", 2016);

console.log(map.get("title"));      // "Understanding ES6"
console.log(map.get("year"));       // 2016
```

У цьому прикладі відбувається збереження двох пар “ключ–значення”. Ключ `"title"` зберігає рядок, тоді як ключ `"year"` зберігає число. Потім для отримання значень для обох ключів викликається метод `get()`. Якщо б ключа не існувало у цій мапі, тоді метод `get()` повернув би спеціальне значення `undefined` замість значення.

Ви також можете використовувати об’єкти у якості ключів, що неможливо при використанні властивостей об’єктів для створення мап обхідним шляхом. Ось приклад:

```js
let map = new Map(),
    key1 = {},
    key2 = {};

map.set(key1, 5);
map.set(key2, 42);

console.log(map.get(key1));         // 5
console.log(map.get(key2));         // 42
```

Цей код використовує об’єкти `key1` та `key2` в якості ключів у мапі для збереження різних значень. Оскільки ці ключі не приводяться у інші типи, кожен об’єкт розглядається як унікальний. Це дозволяє асоціювати з об’єктом додаткові данні без зміни самого об’єкту.

### Методи мап

Мапи мають з множинами кілька спільних методів. Це зроблено для того, щоб взаємодіяти з мапами та множинами схожим чином. Ці три методи доступні як для мап, так і для множин:

* `has(key)` - визначає, чи даний ключ існує в мапі;
* `delete(key)` - видаляє з мапи ключ та асоційоване з ним значення;
* `clear()` - видаляє всі ключі та значення з мапи.

Мапи також мають властивість `size`, що вказує, скільки пар “ключ–значення” містить мапа. Приклад нижче демонструє різні способи використання цих трьох методів та властивості `size`:

```js
let map = new Map();
map.set("name", "Nicholas");
map.set("age", 25);

console.log(map.size);          // 2

console.log(map.has("name"));   // true
console.log(map.get("name"));   // "Nicholas"

console.log(map.has("age"));    // true
console.log(map.get("age"));    // 25

map.delete("name");
console.log(map.has("name"));   // false
console.log(map.get("name"));   // undefined
console.log(map.size);          // 1

map.clear();
console.log(map.has("name"));   // false
console.log(map.get("name"));   // undefined
console.log(map.has("age"));    // false
console.log(map.get("age"));    // undefined
console.log(map.size);          // 0

```

Як і для множин, властивість `size` завжди містить кількість пар “ключ–значення” у мапі. Екземпляр `Map`, у цьому прикладі, спочатку має ключі `"name"` та `"age"`, тому `has()` повертає `true`, якщо передати йому ці ключі. Після видалення ключа `"name"` через метод `delete()`, метод `has()` повертає `false` при передачі `"name"`, а властивість `size` показує на один елемент менше. Потім метод `clear()` видаляє решту ключів, тому `has()` повертає `false` для обох ключів, а властивість `size` дорівнює 0.

Метод `clear()` є швидким способом видалити багато даних з мапи, проте є також спосіб додати багато даних у мапу за один раз.

### Ініціалізація мап

Так само як і для множин, ви можете ініціалізувати мапу з даними передавши масив у конструктор `Map`. Кожен елемент у цьому масиві повинен бути масивом, в якому перший елемент буде ключем, а другий — значенням, що відповідає цьому ключеві. Сама мапа, таким чином, є масивом цих двоелементних масивів, наприклад:

```js
let map = new Map([["name", "Nicholas"], ["age", 25]]);

console.log(map.has("name"));   // true
console.log(map.get("name"));   // "Nicholas"
console.log(map.has("age"));    // true
console.log(map.get("age"));    // 25
console.log(map.size);          // 2
```

Ключі `"name"` та `"age"` додаються у `map` через ініціалізацію у конструкторі. Масив з масивів може здаватись дещо дивним, проте це необхідно для правильного представлення ключів через те, що ключі можуть бути будь–якого типу даних. Збереження ключів у масиві — єдиний спосіб збереження, який дає певність у тому, що вони не будуть приведені у інший тип даних перед збереження у мапі.

### Метод forEach для мап

Метод `forEach()` для мап схожий на `forEach()` для множин та масивів у тому, що він приймає функцію зворотнього виклику, що приймає три аргументи:

1. значення наступної позиції у множині;
1. ключ цього значення;
1. множину, з якої читаються значення.

Ці три аргументи у функції зворотнього виклику більш точно відповідають поведінці `forEach()` для масивів, коли перший аргумент є значенням, а другий — ключем (відповідно до числової індексації у масивах). Ось приклад:

```js
let map = new Map([ ["name", "Nicholas"], ["age", 25]]);

map.forEach(function(value, key, ownerMap) {
    console.log(key + " " + value);
    console.log(ownerMap === map);
});
```

Функція зворотнього виклику у `forEach()` виводить інформацію, що у неї передається. `value` та `key` виводяться безпосередньо, а `ownerMap` порівнюється з `map`, щоб показати, що їхні значення є еквівалентними. Ось вивід:

```
name Nicholas
true
age 25
true
```

Функція зворотнього виклику, яку було передано у `forEach()`, отримує кожну пару “ключ–значення” у тому порядку, в якому вони були додані у мапу. Така поведінка дещо відрізняється від виклику `forEach()` для масивів, при якому зворотній виклик отримує кожен елемент у порядку числової індексації.

I> Ви також можете передати другий аргумент у `forEach()`, щоб задати значення `this` всередині функції зворотнього виклику. Такий виклик поводитиметься так само, як і відповідна версія виклику методу `forEach()` для множин.

### Слабкі мапи (Weak Maps)

Слабкі мапи для мап є тим самим, що і слабкі множини для множин: це спосіб збереження слабких посилань на об’єкти. У *слабких мапах (weak maps)*, кожен ключ має бути об’єктом (при спробі використання ключа–необ’єкта буде кинуто помилку), і ці посилання на об’єкти утримуються слабко, тому вони не перешкоджають збору сміття. Якщо немає посилань на ключ слабкої мапи поза цією слабкою мапою, тоді пара “ключ–значення” видаляється зі слабкої мапи.

Найбільш корисним місцем застосування слабких мап є створення об’єктів, що відповідають певному DOM–елементу на веб–сторінці. Наприклад, деяка JavaScript–бібліотека для веб–сторінок підтримує один спеціальний об’єкт для кожного DOM–елементу, на який посилається бібліотека, і це відображення зберігається внутрішньо у кеші об’єктів.

Складність цього підходу у визначенні, коли DOM–елементу більше немає на веб–сторінці, щоб бібліотека змогла видалити цей асоційований об’єкт. У протилежному випадку, бібліотека зберігатиме непотрібні посилання на DOM–елементи і цим спричинить витік пам’яті. Відслідковування DOM–елементів через слабкі мапи дозволило би бібліотеці асоціювати спеціальні об’єкти з кожним DOM–елементом і при цьому автоматично видаляти будь–які об’єкти у мапі, відповідних DOM–елементів, яких більше не існує.

I> Важливо зрозуміти, що лише ключі, а не значення, у слабких мапах є слабкими посиланнями. Об’єкти, що зберігаються як значення у слабких мапах не підбираються збирачем сміття, якщо всі інші посилання на них видалені.

#### Використання слабких мап

Тип `WeakMap` у  ECMAScript 6 є невпорядкованим списком пар “ключ–значення”, в яких ключ має бути ненульовим об’єктом, а значення може мати будь–який тип. Інтерфейс у `WeakMap` є дуже схожим до інтерфейсу `Map` у тому, що `set()` та `get()` використовуються для додавання та витягнення даних, відповідно:

```js
let map = new WeakMap(),
    element = document.querySelector(".element");

map.set(element, "Original");

let value = map.get(element);
console.log(value);             // "Original"

// видаляємо елемент
element.parentNode.removeChild(element);
element = null;

// тут слабка множина стає порожньою
```

У цьому приклад, зберігається одна пара “ключ–значення”. Ключ `element` є DOM–елементом, що використовується для збереження відповідного рядкового значення. Це значення отримується шляхом передачі DOM–елементу у метод `get()`. Потім DOM–елемент видаляється з документу, а змінній, що посилається на нього, присвоюється значення `null`, і дані також видаляються зі слабкої мапи.

Так само, як і для слабких множин, немає можливості перевірити, що слабка мапа порожня тому, що вона не має властивості `size`. Оскільки не залишається жодного посилання на ключі, ви більше не можете отримати його значення через виклик метода `get()`. Слабка мапа видаляє доступ до значення для цього ключа і тоді запускається збирач сміття, пам’ять, яку займало це значення, звільняється.

#### Ініціалізація слабких мап

Щоб ініціалізувати слабку мапу, передайте масив масивів у конструктор `WeakMap`. Точно як і при ініціалізації звичайної мапи, кожен масив всередині загального масиву має два елементи, при чому перший елемент є ненульовим об’єктом, а другий елемент є значенням (будь–якого типу даних). Наприклад:

```js
let key1 = {},
    key2 = {},
    map = new WeakMap([[key1, "Hello"], [key2, 42]]);

console.log(map.has(key1));     // true
console.log(map.get(key1));     // "Hello"
console.log(map.has(key2));     // true
console.log(map.get(key2));     // 42
```

Об’єкти `key1` та `key2` використовуються у якості ключів у слабкій мапі, а тоді метод `get()` та `has()` можуть отримати значення через них. Якщо конструктор отримає ключ необ’єкт у будь–якій парі “ключ–значення”, то кинеться помилка.

#### Методи слабких мап

Слабкі мапи мають лише два додаткові методи, доступні для взаємодії з парами “ключ значення”. Є метод `has()` для визначення, чи даний ключ існує у мапі та метод `delete()` для видалення певної пари. Метода `clear()` немає через те, що він вимагає перелічуваних ключів, що є неможливим для слабких мап. Цей приклад використовує методи `has()` та `delete()`:

```js
let map = new WeakMap(),
    element = document.querySelector(".element");

map.set(element, "Original");

console.log(map.has(element));   // true
console.log(map.get(element));   // "Original"

map.delete(element);
console.log(map.has(element));   // false
console.log(map.get(element));   // undefined
```

Тут DOM–елемент знову використовується у якості ключа слабкої мапи. Метод `has()` корисний для перевірки того, чи посилання використовується у якості ключа у слабкій мапі. Запам'ятайте, що це працює лише тоді, коли ви маєте ненульові посилання на ключ. Ключ примусово видаляється з слабкої мапи з допомогою методу `delete()`, після цього `has()` повертає `false`, а `get()` повертає `undefined`.

#### Приватні дані у об’єктах

Тоді як більшість розробників вбачають у асоціюванні даних DOM–елементами основний спосіб використання слабких мап, є багато інших можливих використань (і без сумніву, деякі з них ще не достатньо вивчені). Одним з практичних застосувань слабких мап є збереження приватних даних у екземплярах об’єктів. Всі властивості об’єктів є публічними у ECMAScript 6, і тому вам слід проявити трохи винахідливості для того, щоб зробити дані доступними для об’єктів, але недоступними для всього іншого. Розгляньте такий приклад:

```js
function Person(name) {
    this._name = name;
}

Person.prototype.getName = function() {
    return this._name;
};
```

Такий код використовує поширений запис, що починається з нижнього підкреслення і вказує на те, що значення вважається приватним і його не слід змінювати поза екземпляром об’єкта. Передбачається, що для читання `this._name` буде використовуватись `getName()`, змінювати `_name` не можна. Однак, нічого не перешкоджає комусь записати щось у властивість `_name`, тому вона може бути випадково або навмисно перезаписана.

У ECMAScript 5 можливо отримати щось дуже схоже на приватні дані через створення об’єкта за таким шаблоном:

```js
var Person = (function() {

    var privateData = {},
        privateId = 0;

    function Person(name) {
        Object.defineProperty(this, "_id", { value: privateId++ });

        privateData[this._id] = {
            name: name
        };
    }

    Person.prototype.getName = function() {
        return privateData[this._id].name;
    };

    return Person;
}());
```

Цей приклад огортає визначення `Person` у НВФВ, що містить приватні змінні: `privateData` та `privateId`. Об’єкт `privateData` зберігає приватну інформацію для кожного екземпляру, тоді як `privateId` використовується, щоб генерувати унікальний ID для кожного екземпляра. Коли викликається конструктор `Person`, створюється неперелічувана, незмінна і недоступна на запис властивість `_id`.

Потім, створюється поле у об’єкті `privateData`, що відповідає ID екземпляра об’єкта: ось де де зберігається `name`. Пізніше, у функції `getName()`, можна дістати `name` через використання `this._id` в якості ключа для `privateData`. Оскільки `privateData` недоступний поза НВФВ, справжні дані в безпеці, навіть хоча й `this._id` відкривається публічно.

Великою проблемою цього підходу є те, що дані у `privateData` ніколи не зникають, тому що немає способу дізнатись, чи екземпляр об’єкта був знищений. Об’єкт `privateData` завжди зберігатиме додаткові дані. Цю проблему можна вирішити використанням слабких мап, таки чином:

```js
let Person = (function() {

    let privateData = new WeakMap();

    function Person(name) {
        privateData.set(this, { name: name });
    }

    Person.prototype.getName = function() {
        return privateData.get(this).name;
    };

    return Person;
}());
```

Така версія прикладу з `Person` використовує для приховування даних слабку мапу замість об’єкта. Оскільки екземпляр об’єкта `Person` може сам використовуватись у якості ключа, немає проблему зберігати окремий ID. Коли конструктор `Person` викликається, у слабкій мапі створюється нова одиниця з ключем `this` та об’єктом, що містить приватну інформацію, у якості значення. У цьому випадку, це об’єкт, що містить лише поле `name`. Функція `getName()` дістає цю приватну інформацію, передавши `this` у метод `privateData.get()`, який зчитує значення, що є об’єктом, і дістає з нього властивість `name`. Така техніка зберігає приватну інформацію приватною та видаляє її, як тільки екземпляр об’єкта, що асоціюється з нею, буде видалений.

#### Використання слабких мап та їх обмеження

Вирішуючи, коли використовувати слабкі мапи, а коли звичайні, рішення має ґрунтуватись на тому, чи ви хочете використовувати лише об’єкти у якості ключів. Щоразу, коли ви збираєтесь використовувати лише об’єкти у якості ключів — слабка мапа буде найкращим рішенням. Це дозволить вам оптимізувати використання пам'яті та уникнути витоків пам'яті шляхом видалення даних, що більше недоступні.

Запам’ятайте, що слабкі мапи дають вам лише часткову видимість свого вмісту, ви не можете використовувати метод `forEach()`, властивість `size` або метод `clear()` для керування елементами. Якщо вам необхідні можливості для інспектування, тоді звичайні мапи підійдуть краще. Вам просто варто буде слідкувати за використанням пам’яті.

Звісно, якщо ви хочете використовувати необ’єктні ключі, тоді звичайні мапи будуть єдиним варіантом.

## Підсумок

ECMAScript 6 формально вводить множини та мапи в JavaScript. До цього розробники часто використовували об’єкти для імітації множин та мап, часто отримуючи проблеми, пов’язані з обмеженнями, що стосуються властивостей у об’єктах.

Множини — це невпорядковані списки унікальних значень. Значення розглядаються як унікальні, якщо вони не еквівалентні відповідно до методу `Object.is()`. Множини автоматично видаляють значення, що повторюються, тому ви можете використовувати множини для фільтрації масивів на повторення. Множини не є підкласом масивів, ви не можете випадково отримати значення множини. Замість цього, ви можете використати метод `has()` для визначення того, чи значення міститься у множині, а властивість `size` показує число значень у множині. Тип `Set` також має метод `forEach()` для обробки кожного значення у множині.

Слабкі множини — це спеціальний тим, що може містити виключно об’єкти. Об’єкти зберігаються зі слабкими посиланнями, а це означає, що якщо елемент у слабкій множині не має жодного посилання на себе, він буде прибраний збирачем сміття. Слабкі множини не можна інспектувати через складності з управлінням пам'яті, тому найкраще використовувати слабкі множини для відслідковування об’єктів, що мають бути згруповані разом.

Мапи — невпорядковані пари “ключ–значення”, у який і ключ, і значення можуть бути будь–якого типу. Як і для множин, ідентичність ключів визначається викликом методу `Object.is()`, а це означає, що ви можете мати як числовий ключ `5`, так і рядковий `"5"`, в якості двох різних ключів. Значення можна асоціювати з ключем з допомогою методу `set()`, і це значення згодом можна отримати з допомогою методу `get()`. Мапи також мають властивість `size` та метод `forEach()` для легшого доступу до елементів.

Слабкі мапи — спеціальний тип мап, що може містити лише об’єктні ключі. Як і з слабкими множинами, посилання об’єкта, що є ключем, є слабким і видаляється збирачем сміття, якщо воно залишиться єдиним посиланням на цей об’єкт. Коли ключ видаляється збирачем сміття, значення, що асоційоване з цим ключем, також видаляється зі слабкої множини. Цей аспект, що стосується управлінням пам’яттю, робить слабкі мапи прекрасними для кореляції додаткової інформації з об’єктами, чий життєвий цикл керований поза кодом, з якого вони доступні.