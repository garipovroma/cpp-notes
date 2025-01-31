# Шаблоны
- [Запись лекции №1](https://www.youtube.com/watch?v=AXl4_eZ1eis)
- [Запись лекции №2](https://www.youtube.com/watch?v=DwDbH7pxzRA)
- [Практика](https://www.youtube.com/watch?v=CY7vxMSBork)
---
Часто очень хочется делать типизированный класс - например, какую-то структуру данных для разных типов. Здесь и применяются шаблоны.

В Си их не было и было два способа реализовать такое:

- Хранить внутри данные "любого типа" как `void*`, но это неудобно, потому что нельзя хранить ничего, что занимает больше памяти, чем `void*`.  Можно хранить указатели, но это не очень эффективно из-за кэш-промахов.

- Через `#define`:

```c++
#define DECLASE_MYVECTOR(vector_name, type)                       \          
struct vector_name {                                              \
	void push_back(type);                                     \
	type operator[](size_t index) const;                      \
}
```


У дефайнов несколько проблем: если ссылаться на один тип по-разному через `typedef`, то будут генериться два разных класса, которые по сути одно и то же. А ещё нельзя создавать с длинными типами типа `unsigned short`. А ещё может возникнуть дублирование.

Как это делают ~~нормальные люди~~ в C++:

```c++
template <typename T>
// template<class T> это то же самое
struct vector {
     void push_back(T const &);
     T const& operator[](size_t index) const;
}
```

Функции тоже можно типизировать:

```c++
template<typename T>
void swap(T& a, T& b) {
     T tmp = a;
     a = b;
     b = tmp;
}
```

## Специализации

Иногда для каких-то типов хочется сделать отдельную реализацию. Например, `vector<bool>`. Это называется специализацией и делается так:

```c++
template <>
struct vector<bool> {
	// ...
};
```

С точки зрения языка шаблонным параметром может быть любой тип, но, например, вектор из стандартной библиотеке может принимать не любые типы, а, например, только копируемые и присваиваемые. Поэтому, например, `vector<const bool>` сделать нельзя, так как он не является присваиваемым.

Чтобы "заифать типы" есть `std::conditional`, концептуально он работает как-то так:
```c++
template <bool Cond, typename IfTrue, typename IfFalse>
struct conditional;

template <typename IfTrue1, typename IfFalse1>
struct conditional<false, IfTrue1, IfFalse1> {
	typedef IfFalse1 type;
};
template <typename IfTrue1, typename IfFalse1>
struct conditional<true, IfTrue1, IfFalse1> {
	typedef IfTrue1 type;
};
```

Можно делать partial специализацию, например для указателей:

```c++
template <typename V>
struct vector<V*> {
	// ...
}
vector<foo*> c; // выберется эта специализация
```

Есть ли у функций специализации? Да, но нельзя делать partial и вообще чаще всего достаточно перегрузок.

```c++
template<typename T>
void foo(T*);

template<>
void foo<int>(int*);

// void foo(int*); 
```

Чем отличаются две последних функции? Например, на таком коде:

```c++
int main() {
	foo(nullptr); // не работает со специализацией
}
```

Для функций алгоритм выбора такой: 

1. Выбирается перегрузка.
2. Внутри перегрузки выбирается специализация.

Подробнее про **Template argument deduction** на [cppreference](https://en.cppreference.com/w/cpp/language/template_argument_deduction).

Поэтому когда мы подставляем `nullptr`, то из него нельзя понять, на какой тип он указывает, поэтому **deduction** фейлится и получаем ошибку. Если же есть `void foo(int*)`, то выбирается он, как единственный подходящий.

Есть ещё одно правило: *если есть нешаблонная функция, которая идеально совпадает с шаблонной, то выбирается нешаблонная*, аналогично всегда выбирается `explicit` специализация.

```c++
template <typename U, typename V>
struct mytype{};

template <typename U, typename V>
struct mytype<U*, V>{};

template <typename U, typename V>
struct mytype<U, V*>{};

// В таком случае нельзя делать так, будет ошибка ambiguous template instantiation, так как есть два равноправных кандидата
mytype<long*, double*> f;

// если сделать такую специализацию, то выберется она
// template <typename U, typename V>
// struct mytype<U*, V*>{}; 
```

Можно делать *non-type* шаблонные параметры (числа или `enum`):

```c++
template <typename T, size_t N>
struct array {
     T data[N];
}
// можно делать специализацию:
template <typename T>
struct array<T, 0> {};

array<int, 10> a;
array<int, 0> a;
```

А ещё есть такое:

```c++
template <template <typename T> class C>
struct array {
};

template <typename T> 
struct cc {
     
};

array<cc> a;
```

Шаблонные параметры могут иметь дефолтные значения, они работают как и у функций:

```c++
struct default_comparer {};
templte<typename T, typename C = default_comparer>
struct set {};

set<int> a; // юзает дефолтный C
```

## Немного про парсер C++

```c++
struct foo {
	static float a;
};
(foo::a)-b; // может быть бинарное вычитание или каст -b к типу а, чтобы это различать, компилятору нужно резолвить foo
int a(b); // может быть конструктор или определение функции
a < b && c > d; // может быть шаблон a<b&&c> d, а может быть буловое выражение
```

Как компилятор различает, например, в случае `(foo::a)-b`? Особенно если это всё не в одной функции. Здесь работает такое правило: *по умолчанию считаем, что в скобках не тип*.

Пример:

```c++
struct foo {
	static float a;
};
struct bar {
	typedef float a;
};

template<typename T>
void f() {
	int b;
	(T::a)-b; // если хотим тип, то нужно писать (typename T::a)
}

int main {
	f<foo>();
	f<bar>(); // не компилируется
}
```

Если хотим, чтобы это читалось как каст, то нужно писать `(typename T::a)-b`. Кстати, в старых компиляторах *Microsoft* это не было нужно, так как они откладывали токены и парсили только в момент, когда подставляют функцию.

Аналогично для шаблонов по умолчанию считается не шаблоном, иначе нужно писать так:

```c++
T::a < b && c > d; // по умолчанию не шаблон
typename T::template a < b && c > d; // это шаблон
```



## Как это устроено внутри? 

*На лекции очень много [godbolt.org](godbolt.org), поэтому посмотрите [запись](https://www.youtube.com/watch?v=DwDbH7pxzRA) или сами покомпилируйте.*

Начнём с шаблонных функций:

```c++
template <typename T>
void swap(T& a, T& b) {
	T tmp = a;
	a = b;
	b = tmp;
}
auto p = &swap<int>; 
auto q = &swap<char>; 
```

Для каждого типа код функции генерируется отдельно. При этом, например, чтобы сделать `sizeof(swap(a, b))`, компилятору не обязательно подставлять тело функции.

### Немного про разные единицы трансляции:

```c++
// swap.h:
template<typename T>
int swap(T& a, T& b);

// swap.cpp:
template <typename T>
void swap(T& a, T& b) {
	T tmp = a;
	a = b;
	b = tmp;
}

// main.cpp:
#include "swap.h"
int main(){
	int a, b;
	swap(a, b);
}
```

Такое не скомпилируется. Почему? Каждая единица трансляции транслируется отдельно, а потом всё линкуется. Инстанцирование шаблонов (подстановка) происходит до линковки. По этой причине в `swap.cpp` мы не можем сгенерить `swap<int>`, потому что не знаем, что он будет использоваться, а в `main.cpp` не может сгенерить, потому что нет её тела.

Можно определить тело шаблонной функции прямо в `swap.h` и инклудить в разные файлы. Казалось бы, получим ошибку из-за нескольких определений, но нет. Шаблонные функции помечаются компилятором как `inline` и не выдаёт ошибку, считая, что они все одинаковые.

Так же нельзя использовать incomplete типы как параметр. А указатели на них можно.

В стандарте прописано, что инстанцирование происходит только когда необходимо. При этом компилятор может делать это в конце единицы трансляции.

```c++
template <typename T>
struct foo {
	T* a;
	void f();
	void g(){
		T a;
	}
};
int main() {
	foo<void> a; // так скомпилируется
	a.g(); // а так нет, ошибка из-за void a
}
```

В примере выше видно, что если в коде нет вызова функции g, то всё компилируется, так как она не инстанцируется.

С классами работает аналогично: полное тело класса не подставляется, если не требуется.

Например:

```c++
template <typename T>
struct foo {
	T a;
};
int main() {
	foo<void>* a; // так скомпилируется
	a->a; // а так нет, опять ошибка из-за void a
}
```

Почему это важно понимать? Ну например, мы не можем использовать `std::unique_ptr` на incomplete типе. Ну как не можем. Можем, но получим ошибку, если в коде есть вызов деструктора или другие обращение к объекту.
```c++
// mytype.h
#include <memory>
struct object;
struct mytype {
	std::unique_ptr<object> obj;
};
// mytype.cpp
struct object {
	object(int, int, int){};
};
mytype::mytype() : obj(new object(1, 2, 3)){}
// main.cpp
#include "mytype.h"
int main(){
	mytype a;
	return 0;
}
```
Без `main.cpp` компилируется, так как у `a` не вызывался деструктор, поэтому он не инстанцировался. С `main.cpp` компилятор генерирует деструктор, который вызывает деструкторы всех членов класса, а там `unigue_ptr<object>`, у которого при компиляции будет инстанцироваться деструктор, а `object` там `incomplete type`, поэтому получим ошибку компиляции, так как в `unique_ptr` есть специальная проверка, что если удаляется `incomplete type`, то это ошибка. 

Как решить проблему? Сделать явное определение деструктора `mytype` в `mytype.cpp`, где у нас определён `object`.


Ещё пример:

```c++
template <typename T>
struct base {
	typename T::mytype a;
};
template <typename T>
struct derived : base<derived<T>> {
	derived* a; // внутри класса можно ссылаться без шаблонного параметра, будет использоваться текущее инстанцирование
	typedef T mytype;
};
derived<int> a; // не скомпилируется
```

Почему это не скомпилируется? Посмотрим на пример попроще.

```c++
template <typename T>
struct base {
	typename T::mytype a;
};
struct derived : base<derived> {
	typedef int mytype;
};
```

Тоже не скомпилируется с ошибкой про incomplete type `derived`. Почему? Ну потому что `derived` является incomplete типом, когда инстанцируется `base<derived>`.

В предыдущем примере тот же самый эффект: так как `derived` шаблонный, то он не инстанцируется сразу, но когда мы инстанцируем `derived`, то он создаётся как incomplete (complete он станет после подстановки базовых классов), происходит подстановка base и получаем ошибку.

*Может быть интересно прочитать про идиому [CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)*

## Явное инстанцирование шаблонов

Мы не хотим, чтобы одни и те же лишние инстанцирования были в разных единицах трансляции. Чтобы этого избежать, можно делать так:

```c++
template <typename T>
void foo(T) {
}
template void foo<int>(int); // генерирует тело функции в этом месте
template void foo<float>(float);
template void foo<double>(double);
```

## Подавление инстанцирования
Подавление явного инстанцирования, если знаем, что функции уже где-то инстанцированы и мы не хотим лишних:
```c++
extern template void foo<int>(int); 
extern template void foo<float>(float);
```

"Выдаём тело наружу и говорим, что уже проинстанцировано, `main` не будет пытаться инстанцировать функцию, так как увидит `extern` и будет работать соответствующе."



Последний пример:

```c++
struct arg {
	struct type {};
};
template <typename T>
struct base {
	int a;
	int b;
};
template <typename T> 
struct derived : base<T> {
	void f() {
		typename T::type a; // depended имя - зависит от шаблонного параметра
		a + 1; // не будет ошибки компиляции, так как откладывается до инстанциации
		arg::type b; // не depended
		b + 1; // сразу ошибка компиляции
		x = 5; // является ли depended?
	}
}

int main() {
	// derived<arg> d;
	// d.f(); // из-за этого происходят ошибки компиляции, если в derived что-то не так, так как происходит инстанцирование 
	return 0;
}
```

Depended имена компилятор откладывает до инстанцирования.

Является ли `x` в `derived` depended? С одной стороны, здесь не участвуют шаблонные параметры, но с другой стороны у нас может быть такая специализация `base`,  в которой нет его? А ещё этот `x` может быть глобальной переменной. Тут вопрос, на что мы компилятор должен ссылаться?

По стандарту определяется следующим образом. Компилятор ищет `x`, игнорируя базовые классы, так как иначе компилятор не мог бы откидывать любые неизвестные переменные. Если хотим ссылаться на `x` из базы, то нужно писать явно `base<T>::x` или `this->x`. Тогда он будет depended: если писать `base<T>::x`, то очевидно, почему, если `this->x`, то понятно, что `this` зависит от шаблонного параметра. При этом компилятор всё равно откладывает это до инстанцирования, даже на таком не выдает ошибку:

```c++
// внутри derived
		void* x;
		void f() {
			this->x = 5;
		}
```

Компилятор имеет право выдавать ошибку, если при подстановке любого типа будет ошибка инстанцирования.

На эту тему можно почитать про [two-phase name lookup](http://blog.llvm.org/2009/12/dreaded-two-phase-name-lookup.html).
