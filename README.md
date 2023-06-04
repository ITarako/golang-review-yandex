# yandex

## Сборка мусора и планировщик - секреты

TODO

## Указатели

`*` - разыменование указателя (пример: a := \*b), но для арументов функции - тип указателя
`&` - взятие адреса (пример: a := &b - положить в переменную указатель на значение)

- в Gо нет ссылок, только указатели (ссылки есть в c/c++).
- у переменой есть значение, адрес ссылки это адрес значения, а указатель на значение имеет свой собственный адрес (​это очень грубое объяснение).

## interface{}

- алиас any
- x.(MyType) - это "утверждение типа" (а не "приведение типа", как например: float64 к int)

## Ссылочные типы данных

Ссылочные типы данных в GoLang - это типы данных, которые хранятся в системной куче (heap) и передаются по ссылке, а не по значению. Это означает, что при передаче ссылочного типа данных в функцию, функция работает с оригинальным объектом, а не с его копией. Некоторые из ссылочных типов данных в GoLang:

- Срезы (slices) - это динамические массивы, которые представляют собой ссылку на последовательность элементов определенного типа.
- Карты (maps) - это ассоциативные массивы, которые представляют собой ссылку на набор пар ключ-значение.
- Каналы (channels) - это механизм для обмена данными между горутинами (goroutines) в многопоточной программе.
- Указатели (pointers) - это переменные, которые хранят адрес в памяти другой переменной.
- ?? Структуры (structs) - это пользовательские типы данных, которые могут содержать поля разных типов.
- Интерфейсы (interfaces) - это типы данных, которые определяют набор методов, которые должны быть реализованы для типа данных, чтобы он удовлетворял интерфейсу.
- Функции (functions) - это типы данных, которые могут быть переданы в качестве аргументов другим функциям или возвращены из функций.

Все эти типы данных являются ссылочными в GoLang и передаются по ссылке, а не по значению.

Поправка:

В GoLang тип данных `struct` является составным типом данных, который объединяет несколько полей разных типов данных в один объект. `struct` не является ссылочным типом данных, а является значимым типом данных, то есть при передаче `struct` в функцию или присваивании его переменной происходит копирование значений его полей. Однако, при передаче `struct` в функцию в качестве аргумента, происходит передача его копии, что может быть неэффективно для больших `struct`. В таких случаях можно использовать указатели на `struct`.

## Передача слайса как аргумента функции

Длина и вместимость передаются по значению, но массив значений передается по ссылке. Вследствие этого получается неявное поведение: добавленные элементы не сохранятся в исходный слайс, но изменение существующих останется.

```go
package main

import "fmt"

func main() {
	cap := 4 // если 3, то ответы разные; если 4 - одинаковые
	var a = make([]int, 0, cap)
	a = append(a, 111, 222, 333)

	fmt.Printf("%#v\n", getArray(a))
	fmt.Printf("%#v\n", a)
}

func remove(slice []int, s int) []int {
	return append(slice[:s], slice[s+1:]...)
}

func getArray(a []int) []int {
	a = append(a, 444)
	a = remove(a, 0)
	return a
}
```

Общий формат среза: a[начало:конец:шаг]. Если начало не указано, то по умолчанию начало считается 0. Если конец не указан, то по умолчанию конец считается длиной массива. Если шаг не указан, то по умолчанию шаг считается равным 1.

- [Что нужно знать о слайсах в Go](https://www.youtube.com/watch?v=1vAIvqDo5LE)

## Heap

Куча (heap) - это структура данных, которая упорядочивает элементы в виде двоичного дерева, при этом каждый узел имеет значение, которое не меньше (для кучи минимумов) или не больше (для кучи максимумов) значений его потомков. В GoLang куча представлена типом `heap.Interface`, который описывает методы, необходимые для работы с кучей:

```go
// heap.go
type Interface interface {
	sort.Interface
	Push(x any) // add x as element Len()
	Pop() any   // remove and return element Len() - 1.
}
// sort.go
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int

	// Less reports whether the element with index i
	// must sort before the element with index j.
	//
	// If both Less(i, j) and Less(j, i) are false,
	// then the elements at index i and j are considered equal.
	// Sort may place equal elements in any order in the final result,
	// while Stable preserves the original input order of equal elements.
	//
	// Less must describe a transitive ordering:
	//  - if both Less(i, j) and Less(j, k) are true, then Less(i, k) must be true as well.
	//  - if both Less(i, j) and Less(j, k) are false, then Less(i, k) must be false as well.
	//
	// Note that floating-point comparison (the < operator on float32 or float64 values)
	// is not a transitive ordering when not-a-number (NaN) values are involved.
	// See Float64Slice.Less for a correct implementation for floating-point values.
	Less(i, j int) bool

	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

![](./assets/heap.png)

На картинке показано справа, как выглядит куча минимумов (i.e. "Priority Queue"), где каждый узел имеет значение, которое не меньше значений его потомков. В куче минимумов наименьший элемент всегда находится в корне дерева, что делает его полезным для решения ряда задач, таких как нахождение k наименьших элементов в списке или поиск медианы в потоке данных. Для работы с кучей можно использовать пакет `container/heap`, который предоставляет реализацию кучи минимумов и максимумов, а также методы для работы с ней, такие как `Push()`, `Pop()`, `Fix()`, `Remove()`, и другие.

Добавление нового элемента в кучу выполняется путем помещения элемента в последнюю свободную позицию в массиве и затем "всплытия" его вверх по дереву, пока не будет найдена его правильная позиция. Удаление элемента с наименьшим (наибольшим) значением выполняется путем замены его последним элементом в массиве, после чего он "просеивается" вниз по дереву, пока не будет найдена его правильная позиция.

## DFS/BFS

DFS (Depth-First Search) и BFS (Breadth-First Search) - это два алгоритма поиска в графе. DFS ищет в глубину, переходя на каждый уровень графа, пока не будет найден целевой узел или не будут исследованы все узлы. BFS ищет в ширину, посещая все узлы на одном уровне перед переходом к узлам следующего уровня. Оба алгоритма могут быть использованы для поиска кратчайшего пути в невзвешенном графе, но только BFS может быть использован для этой цели во взвешенном графе.

Взвешенный граф - это граф, в котором каждому ребру присвоено некоторое числовое значение, называемое весом. Вес может отражать, например, расстояние между двумя вершинами, стоимость перехода от одной вершины к другой и т.д. Взвешенный граф используется для решения задач, связанных с оптимизацией и нахождением кратчайшего пути между вершинами. Для поиска кратчайшего пути в взвешенном графе обычно используется алгоритм Дейкстры или алгоритм Флойда-Уоршелла.

## Dynamic Programming

Динамическое программирование — способ решения сложных задач путём разбиения их на более простые подзадачи. Он применим к задачам с оптимальной подструктурой, которые выглядят как набор перекрывающихся подзадач, сложность которых чуть меньше исходной. В этом случае время вычислений, по сравнению с «наивными» методами, можно значительно сократить.

Ключевая идея в динамическом программировании достаточно проста. Как правило, чтобы решить поставленную задачу, требуется решить отдельные части задачи (подзадачи), после чего объединить решения подзадач в одно общее решение. Часто многие из этих подзадач одинаковы. Подход динамического программирования состоит в том, чтобы решить каждую подзадачу только один раз, сократив тем самым количество вычислений. Это особенно полезно в случаях, когда число повторяющихся подзадач экспоненциально велико.

Метод динамического программирования сверху — это простое запоминание результатов решения тех подзадач, которые могут повторно встретиться в дальнейшем. Динамическое программирование снизу включает в себя переформулирование сложной задачи в виде рекурсивной последовательности более простых подзадач.

Разница между динамическим программированием и жадными алгоритмами заключается в том, что при динамическом программировании существуют перекрывающиеся подпроблемы, и эти подпроблемы решаются с помощью мемоизации. "Мемоизация" - это техника, при которой решения подпроблем используются для более быстрого решения других подпроблем.

Разница в том, что динамическое программирование требует запоминания ответа для меньших состояний, в то время как жадный алгоритм является локальным в том смысле, что вся необходимая информация находится в текущем состоянии. Конечно, существует некоторое пересечение.

## Greedy Method

Жадный алгоритм — алгоритм, заключающийся в принятии локально оптимальных решений на каждом этапе, допуская, что конечное решение также окажется оптимальным. В общем случае нельзя сказать, можно ли получить оптимальное решение с помощью жадного алгоритма применительно к конкретной задаче. Но есть две особенности, характерные для задач, которые решаются с помощью жадных алгоритмов: принцип жадного выбора и свойство оптимальности для подзадач.

### Принцип жадного выбора

Говорят, что к задаче оптимизации применим принцип жадного выбора, если последовательность локально оптимальных выборов даёт глобально оптимальное решение. В этом состоит главное отличие жадных алгоритмов от динамического программирования: во втором просчитываются сразу последствия всех вариантов.

Чтобы доказать, что жадный алгоритм даёт оптимум, нужно попытаться провести доказательство, аналогичное доказательству алгоритма задачи о выборе заявок. Сначала мы показываем, что жадный выбор на первом шаге не закрывает путь к оптимальному решению: для любого решения есть другое, согласованное с жадным выбором и не хуже первого. Потом мы показываем, что подзадача, возникшая после жадного выбора на первом шаге, аналогична исходной. По индукции будет следовать, что такая последовательность жадных выборов даёт оптимальное решение.

### Оптимальность для подзадач

Это свойство говорит о том, что оптимальное решение всей задачи содержит в себе оптимальные решения подзадач.

## Про сложность алгоритма O(n)

Сложность алгоритма это величина, которая измеряет количество инструкций процессора затраченное на выполнение алгоритма.

Если есть цикл пробегающий по всем элементам массива из n элементов, то это значит что процессору потребуется выполнить n инструкций, следовательно сложность будет O(n).

При оценке сложности всегда считается худший случай, то есть, если мы выходим из цикла при нахождении ответа раньше, то сложность всё равно будет O(n), так как в худшем случае ответ будет в последнем ​элементе массива.

Из этого следует что сложность алгоритма с двумя вложенными циклами квадратична, то есть O(n^2) и т.д.

Если алгоритм выполняется за строго ограниченное количество инструкций (например обращение к элементу массива по индексу), то такая сложность называется константной O(1).

​Есть ещё факториальная сложность O(n!) такая сложность очень плоха, и алгоритмы вообще не эффективны. Также как и сложность O(2^n). Пример такого алгоритма - рекурсивное вычисление чисел Фиббаначи (вариант без мемоизации).

Замечание: O(n^2) не равно O(2^n). Временная сложность O(n^2) означает, что время выполнения алгоритма увеличивается квадратично с увеличением размера входных данных, тогда как O(2^n) означает экспоненциальную временную сложность, при которой время выполнения алгоритма увеличивается экспоненциально с увеличением размера входных данных.

Стоит оговориться, если нас просят оценить, например, сложность добавления элемента в массив, то тут будет сложность O(1\*), так называемая амортизированная сложность. То есть добавляем элемент всегда за O(1), но когда нам нужно запросить новую память для элемента при расширении создастся новый массив и туда всё скопируется. Сложность будет O(n).

![](./assets/algo.png)

Пояснения: O – оценка для худшего случая, Ω – оценка для лучшего случая, Θ – оценка для среднего случая.

## Про O(log n)

O(log n) - это алгоритмическая сложность, которая означает, что время выполнения алгоритма увеличивается не пропорционально размеру входных данных (n), а по логарифмической шкале. То есть, при увеличении n в 10 раз, время выполнения алгоритма увеличится только на несколько единиц.

Это очень эффективная сложность, и она часто встречается в алгоритмах, которые работают с отсортированными данными, такими как бинарный поиск. В бинарном поиске мы делим отсортированный массив пополам на каждом шаге, что дает нам O(log n) времени выполнения.

Однако, не все алгоритмы могут работать с O(log n) сложностью. Например, сортировка пузырьком имеет O(n^2) сложность, что означает, что время выполнения алгоритма увеличивается пропорционально квадрату размера входных данных. Поэтому важно выбирать подходящий алгоритм в зависимости от задачи, которую необходимо решить.

## Про O(n log n)

O(n \* log(n)) означает, что время выполнения алгоритма увеличивается в логарифмической пропорции с увеличением размера входных данных. Это временная сложность, которая является более быстрой, чем квадратичная O(n^2), но медленнее, чем линейная O(n) или константная O(1). Алгоритмы с временной сложностью O(n log n) часто используются в сортировке данных, например, быстрой сортировке (quicksort) или сортировке слиянием (merge sort).

## Книжки про Алгоритмы

Для изучения оценки сложности алгоритма по "time complexity" & "space complexity" можно начать с книги "Introduction to Algorithms" авторов Кормена, Лейзерсона, Ривеста и Штайна. Это общепризнанная книга по алгоритмам, которая покрывает базовые алгоритмические концепции, включая оценку временной и пространственной сложности. Книга также содержит множество примеров и задач для практического применения.

Или же книжка "[Грокаем алгоритмы](https://en.ecomed.dgmu.ru/wp-content/uploads/2020/01/%D0%93%D1%80%D0%BE%D0%BA%D0%B0%D0%B5%D0%BC_%D0%B0%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC%D1%8B.pdf)".

Или ещё [Основы алгоритмов - интерактив от Yandex](https://academy.yandex.ru/handbook/algorithms)

Из YouTube: [ВСЯ СЛОЖНОСТЬ АЛГОРИТМОВ ЗА 11 МИНУТ](https://www.youtube.com/watch?v=cXCuXNwzdfY)

## [/article](./article/) - алгоритмические задачи

- [Как проходят алгоритмические секции на собеседованиях в Яндекс](https://habr.com/ru/companies/yandex/articles/449890/)

### Задача A. Камни и украшения

> Даны две строки строчных латинских символов: строка J и строка S. Символы, входящие в строку J, — «драгоценности», входящие в строку S — «камни». Нужно определить, какое количество символов из S одновременно являются «драгоценностями». Проще говоря, нужно проверить, какое количество символов из S входит в J.

Это очень простая разминочная задача, к которой прилагаются решения на нескольких языках программирования, чтобы участники могли освоиться с проверяющей системой.

Алгоритм достаточно простой: из строки с «драгоценностями» необходимо построить множество, затем пройтись по строке с «камнями» и каждый символ проверить на вхождение в это множество. Используйте такую реализацию множества, чтобы гарантировать линейную сложность полученного решения, несмотря на то, что входные строки очень короткие и поэтому возможно сдать даже квадратичный по сложности алгоритм.

### Задача B. Последовательно идущие единицы

> Требуется найти в бинарном векторе самую длинную последовательность единиц и вывести её длину.

Алгоритм решения следующий: пройтись по всем элементам массива; встретив единицу, нужно увеличить счётчик длины текущей последовательности, а, встретив ноль, нужно обнулить этот счётчик. В конце нужно вывести максимальное из значений, которые принимал счётчик.

Проверьте, что правильно обрабатываете ситуацию, когда массив заканчивается на искомую последовательность единиц. При аккуратной реализации такая ситуация не потребует специальной обработки.

Постарайтесь использовать лишь константный объём дополнительной памяти.

### Задача C. Удаление дубликатов

> Дан упорядоченный по неубыванию массив целых 32-разрядных чисел. Требуется удалить из него все повторения.

Правильный алгоритм последовательно обрабатывает элементы массива, сравнивая их с последним выведенным. Нужно не забыть обновлять переменную, содержащую последний выведенный элемент и, кроме того, не ошибиться при обработке последнего элемента.

При решении этой задачи также не нужно использовать дополнительную память.

### Задача D. Генерация скобочных последовательностей

> Дано целое число n. Требуется вывести все правильные скобочные последовательности длины 2 \* n, упорядоченные лексикографически (см. https://ru.wikipedia.org/wiki/Лексикографический_порядок). В задаче используются только круглые скобки.

Это пример относительно сложной алгоритмической задачи. Будем генерировать последовательность по одному символу; в каждый момент мы можем к текущей последовательности приписать либо открывающую скобку, либо закрывающую. Открывающую скобку можно дописать, если до этого было добавлено менее n открывающих скобок, а закрывающую — если в текущей последовательности количество открывающих скобок превосходит количество закрывающих. Такой алгоритм при аккуратной реализации автоматически гарантирует лексикографический порядок в ответе; работает за время, пропорциональное произведению количества элементов в ответе на n; при этом требует линейное количество дополнительной памяти.

Примером неэффективного алгоритма был бы следующий: сгенерируем все возможные скобочные последовательности, а затем выведем лишь те из них, что окажутся правильными. При этом объём ответа не позволит решить задачу быстрее, чем тот алгоритм, что приведёт выше.

### Задача E. Анаграммы

Эта достаточно простая задача — типичный пример задачи, для решения которой необходимо использовать ассоциативные массивы. При решении нужно учитывать, что символы могут повторяться, поэтому необходимо использовать не множества, а словари. Поэтому решение будет следующим: составим из каждой строки по словарю, который для каждого символа будет хранить количество его повторений; затем сравним получившиеся словари. Если они совпадают, необходимо вывести единицу, в противном случае — ноль.

Альтернативное решение: отсортируем входные строки, а затем сравним их. Это решение хуже в том, что оно работает медленнее, а также меняет входные данные. Зато такое решение не использует дополнительной памяти.

Если в процессе собеседования у вас возникло несколько вариантов решения, отличающихся своими по своим характеристикам, расскажите об этом. Всегда здорово, когда разработчик знает несколько вариантов решения задачи и может рассказать об их сильных и слабых сторонах.

### Задача F. Слияние k сортированных списков

> Даны k отсортированных в порядке неубывания массивов неотрицательных целых чисел, каждое из которых не превосходит 100. Требуется построить результат их слияния: отсортированный в порядке неубывания массив, содержащий все элементы исходных k массивов. Длина каждого массива не превосходит 10 \* k.

Для каждого массива создадим по указателю; изначально каждый указатель расположен в начале соответствующего массива. Элементы, соответствующие позициям указателей, поместим в любую структуру данных, которая поддерживает извлечение минимума — это может быть мультимножество или, например, куча. Далее будем извлекать из этой структуры минимальный элемент, помещать его в ответ, сдвигать позицию указателя в соответствующем массиве и помещать в структуру данных очередной элемент из этого массива.

В этой задаче многие испытывают сложности с форматом ввода. Обратите внимание на то, что первые элементы строк не описывают элементы массивов, они описывают длины массивов!

#### Answer #1:

> В данном коде мы создаем структуру Item, которая содержит значение элемента, номер массива, из которого был взят элемент, и индекс элемента в массиве. Затем мы создаем приоритетную очередь PriorityQueue на основе этой структуры, которая будет использоваться для извлечения минимального элемента.
>
> Далее мы создаем указатели на начало каждого массива и добавляем первые элементы каждого массива в приоритетную очередь. Затем мы начинаем извлекать минимальные элементы из очереди, добавляя их в результирующий массив и сдвигая указатель на соответствующий массив. Если указатель не достиг конца массива, мы добавляем следующий элемент из этого массива в приоритетную очередь.
>
> В итоге мы получаем отсортированный в порядке неубывания массив, содержащий все элементы исходных k массивов. В примере выше мы передаем в функцию mergeArrays три отсортированных массива и получаем отсортированный массив [0 0 3 5 6 6 7 28].

#### Answer #2:

> В этом коде мы используем те же самые структуры данных и алгоритм, что и в предыдущем примере, но более лаконично записываем код. Мы создаем массив pointers для хранения указателей на текущий элемент в каждом массиве, инициализируем его нулями и используем его для проверки достижения конца каждого массива.
>
> Также мы используем оператор append для добавления элементов в результирующий массив, вместо явного указания индекса. Это делает код более читаемым и лаконичным.
>
> В итоге мы получаем тот же отсортированный в порядке неубывания массив, содержащий все элементы исходных k массивов.

#### Answer #3:

> Этот код считывает количество массивов k, затем считывает каждый массив и его элементы, сливает массивы в один отсортированный массив и выводит его элементы. Для слияния массивов используется куча, которая поддерживает извлечение минимума и добавление элементов. Код также содержит функции для построения кучи и поддержания ее свойств.

## [/leetcode](./leetcode/)

- [LEETCODE PATTERNS](https://seanprashad.com/leetcode-patterns/)

### linked lists:

- [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/)
- [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)
- [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers/)
- [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)

### binary search:

- [704. Binary Search](https://leetcode.com/problems/binary-search/)
- [374. Guess Number Higher or Lower](https://leetcode.com/problems/guess-number-higher-or-lower/)
- [74. Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix/)
- [33. Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/)
- [153. Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/)
- [81. Search in Rotated Sorted Array II](https://leetcode.com/problems/search-in-rotated-sorted-array-ii/)

### hash table:

- [136. Single Number](https://leetcode.com/problems/single-number/) - решить за O(1) по памяти
- [1. Two Sum](https://leetcode.com/problems/two-sum/)
- [18. 4Sum](https://leetcode.com/problems/4sum/)
- [49. Group Anagrams](https://leetcode.com/problems/group-anagrams/)
- [242. Valid Anagram](https://leetcode.com/problems/valid-anagram/)
- [438. Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/)

### queue/stack:

- [20. Valid Parentheses](https://leetcode.com/problems/valid-parentheses/)

### dfs/bfs:

- [200. Number of Islands](https://leetcode.com/problems/number-of-islands/)
- [301. Remove Invalid Parentheses](https://leetcode.com/problems/remove-invalid-parentheses/)
- [637. Average of Levels in Binary Tree](https://leetcode.com/problems/average-of-levels-in-binary-tree/)

### sort:

- [56. Merge Intervals](https://leetcode.com/problems/merge-intervals/)

### heap/hash:

- [692. Top K Frequent Words](https://leetcode.com/problems/top-k-frequent-words/)
- [347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/)

### two pointers:

- [11. Container With Most Water](https://leetcode.com/problems/container-with-most-water/)
- [763. Partition Labels](https://leetcode.com/problems/partition-labels/)

### sliding window:

- [480. Sliding Window Median](https://leetcode.com/problems/sliding-window-median/)
- [239. Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/)
- [424. Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/)

### tree:

- [100. Same Tree](https://leetcode.com/problems/same-tree/)
- [101. Symmetric Tree](https://leetcode.com/problems/symmetric-tree/)
- [110. Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree/)
- [113. Path Sum II](https://leetcode.com/problems/path-sum-ii/)

### greedy problems:

- [121. Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)
- [122. Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)
- [714. Best Time to Buy and Sell Stock with Transaction Fee](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-transaction-fee/)
- [309. Best Time to Buy and Sell Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/)

---

- [coderun.yandex.ru](https://coderun.yandex.ru/)
- [practicum.yandex.ru](https://practicum.yandex.ru/profile/algorithms-interview/)
