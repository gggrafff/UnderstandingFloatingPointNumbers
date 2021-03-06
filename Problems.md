# Возможные проблемы при работе с числами с плавающей точкой

Числа с плавающей точкой обладают рядом особенностей, что иногда усложняет работу. Общее правило: чтобы избежать ошибок при работе с числами с плавающей точкой, не используйте числа с плавающей точкой. Очень часто в практических задачах работу с действительными числами можно заменить работой с целыми числами, пользуйтесь этим. Например, деньги можно считать как целое количество копеек, а некоторые геометрические формулы могут быть преобразованы так, чтобы все операции были целочисленные (чтобы сравнить длины векторов, не обязательно брать квадратный корень). Но часто работа с числами с плавающей точкой необходима, поэтому далее опишем тонкости и наиболее частые ошибки при работе с плавающей точкой.

<!-- vim-markdown-toc GFM -->

* [Конечная точность представления чисел](#Конечная-точность-представления-чисел)
	* [Накопленная погрешность](#Накопленная-погрешность)
		* [Пример](#Пример)
	* [Сравнение чисел](#Сравнение-чисел)
		* [Пример](#Пример-1)
		* [Оценка абсолютной величины разницы чисел](#Оценка-абсолютной-величины-разницы-чисел)
		* [Оценка абсолютной разницы чисел с привязкой к величине чисел](#Оценка-абсолютной-разницы-чисел-с-привязкой-к-величине-чисел)
		* [Нахождение разницы как целых чисел](#Нахождение-разницы-как-целых-чисел)
* [Переменная абсолютная точность](#Переменная-абсолютная-точность)
	* [Неассоциативность арифметических операций](#Неассоциативность-арифметических-операций)
	* [Потеря точности при возрастании числа](#Потеря-точности-при-возрастании-числа)
* [Другие особенности](#Другие-особенности)
	* [Отношение порядка](#Отношение-порядка)
* [Задачи](#Задачи)

<!-- vim-markdown-toc -->

## Конечная точность представления чисел

Не все десятичные числа имеют двоичное представление с  плавающей запятой. Например, число «0.2» будет представлено как  «0.200000003» при использовании одинарной точности. 

### Накопленная погрешность

Погрешность записи чисел сказывается и на результате арифметических операций над этими числами. Например, «0.2 + 0.2 ≈ 0.4».  Абсолютная ошибка в отдельном случае не высока, но если сохранить такое непредставимое число и использовать его в цикле, можем получить накопленную погрешность. Старайтесь этого избегать.

#### Пример

Найдём 30-й член геометрической прогрессии с множителем q = 1.1 и первым членом прогрессии b0 = 1.0.

Ответ, полученный на калькуляторе: 17.4494022689. 

Будем использовать для арифметических операций и записи результата числа двойной точности (64-бита), а множитель q запишем числом одинарной точности. Это позволяет получить достаточно точный результат вычислений и оценить только влияние погрешности представления множителя q.

```c++
#include <iostream>
#include <iomanip>

template<typename FloatingType>
double geometric_progression(double b0, FloatingType q, uint32_t n) {
    double result = b0;
    for (uint32_t i = 0; i < n; ++i) {
        result *= q;
    }
    return result;
}

int main()
{
    std::cout << std::fixed << std::setprecision(12);
    
    std::cout << geometric_progression(1.0, 1.1f, 30) << std::endl;
    std::cout << geometric_progression(1.0, 1.1, 30) << std::endl;
    
    return 0;
}
```

### Сравнение чисел

Погрешность записи чисел часто вызывает трудности с их сравнением.

#### Пример

```c++
#include <iostream>

int main()
{
    // Пример 1
    // 0.2 не имеет точного представления
    // Мы сравниваем число 0.2 одинарной точности с числом 0.2 двойной точности. 
    float a = 0.2;
    std::cout << (a == 0.2) << std::endl;
    
    // Пример 2
    // Проследите, как записываются эти числа и почему такой результат?
    std::cout << (0.2 == 0.6 / 3.0) << std::endl;
    
    return 0;
}
```

Так как же лучше сравнивать числа с плавающей точкой? Подробнее об этом читайте в [Comparing floating point numbers, Bruce Dowson](https://randomascii.wordpress.com/2012/02/25/comparing-floating-point-numbers-2012-edition/), а здесь кратко опишем основные способы. 

#### Оценка абсолютной величины разницы чисел

Этот способ работает лучше, чем в примере выше. Нам нужно задать интересующую нас точность и для сравнения на равенство двух чисел проверить, что их разность меньше, чем значение этой точности:

```c++
#include <iostream>

const float PRECISION = 0.0001;

template<typename FloatingType1, typename FloatingType2>
bool is_equal(FloatingType1 num1, FloatingType2 num2) {
    return std::abs(num1 - num2) < PRECISION;
}

int main()
{
    // Пример 1
    float a = 0.2;
    std::cout << is_equal(a, 0.2) << std::endl;
    
    // Пример 2
    std::cout << is_equal(0.2, 0.6 / 3.0) << std::endl;
    
    return 0;
}
```

Но этот способ не идеален и подходит, если все числа примерно одного порядка. 

Недостаток подхода в том, что погрешность представления числа увеличивается с ростом самого этого числа. Следовательно значение PRECISION в нашем примере тоже должно как-то зависеть от значения сравниваемых чисел.

#### Оценка абсолютной разницы чисел с привязкой к величине чисел

Предыдущий способ можно было бы модицицировать так:

```c++
bool is_equal(float a, float b, float max_rel_diff = PRECISION) {
    // Calculate the difference.
    float diff = fabs(a - b);
    a = fabs(a);
    b = fabs(b);
    // Find the largest
    float largest = (b > a) ? b : a;
 
    if (diff <= largest * max_rel_diff)
        return true;
    return false;
}
```

Это неплохо работает, но выбор max_rel_diff не самый очевидный.

#### Нахождение разницы как целых чисел

Одной из особенностей описанного представления чисел с плавающей точкой является то, что все неNaN значения корректно упорядочены, если рассматривать их как знаковые целые числа. Брюс Доусон приводит следующий код для сравнения чисел:

```c++
bool AlmostEqualUlpsAndAbs(float A, float B,
            float maxDiff, int maxUlpsDiff)
{
    // Check if the numbers are really close -- needed
    // when comparing numbers near zero.
    float absDiff = fabs(A - B);
    if (absDiff <= maxDiff)
        return true;
 
    Float_t uA(A);
    Float_t uB(B);
 
    // Different signs means they do not match.
    if (uA.Negative() != uB.Negative())
        return false;
 
    // Find the difference in ULPs.
    int ulpsDiff = abs(uA.i - uB.i);
    if (ulpsDiff <= maxUlpsDiff)
        return true;
 
    return false;
}
```

В этой программе maxUlps (от Units-In-Last-Place) – это максимальное  количество чисел с плавающей запятой, которое может лежать между  проверяемым и ожидаемым значением. Другой смысл этой переменной – это  количество двоичных разрядов (начиная с младшего), которое в сравниваемых числах  разрешается упустить. 

## Переменная абсолютная точность

Число с плавающей запятой имеет фиксированную относительную точность и изменяющуюся абсолютную. То есть, точность записи числа зависит от его величины. 

### Неассоциативность арифметических операций

Зависимость точности записи от величины числа приводит к тому, что арифметические операции теряют ассоциативность (важно, в каком порядке выполнять операции).

В арифметике с плавающей запятой правило (a * b) * c = a * (b * c) нарушается для любых арифметических операций. Например:

(10^20 + 1) - 10^20 = 0 

(10^20 - 10^20) + 1 = 1

Допустим у нас есть программа суммирования чисел:

```С++
double s = 0.0;
for (int i=0; i<n; i++) s = s + t[i];
```


Некоторые компиляторы по умолчанию могут переписать код для  использования нескольких АЛУ одновременно (будем считать, что n делится  на 2):

```С++
double sa[2], s; 
sa[0]=sa[1]=0.0;
for (int i=0; i<n/2; i++) {
    sa[0]=sa[0]+t[i*2+0];
    sa[1]=sa[1]+t[i*2+1];
}
S=sa[0]+sa[1];
```

Так как операции суммирования не ассоциативны, эти две программы могут выдать различный результат. 

Чтобы избежать ошибок, связанных с этим поведением, старайтесь группировать арифметические операции так, чтобы операнды в каждой операции были примерно одного порядка. 

### Потеря точности при возрастании числа

Часто в последовательности арифметических операций промежуточные результаты вычислений на порядки отличаются от чисел на входе и выходе этой последовательности операций. Складывается ситуация, что промежуточные результаты вычислений записываются с иной точностью. Это может приводить к проблемам. 
Например, при поиске среднего арифметического значения мы сначала суммируем числа, затем делим сумму на количество чисел. При большом количестве чисел их сумма будет значительно больше, чем они сами, а следовательно будет записана с меньшей точностью. 

Пример кода, который демонстрирует это поведение:

```c++
#include <iostream>
#include <vector>

template<typename FloatingType>
float average(std::vector<float> nums) {
    FloatingType sum = 0.0f;
    for (auto num: nums) {
        sum += num;
    }
    return sum / static_cast<FloatingType>(nums.size());
}

int main()
{
    std::vector<float> nums;
    for (size_t i = 0; i < 1'000'000; ++i) {
        nums.push_back(static_cast<float>(i % 101));
    }
    
    std::cout << "avg floats: " << average<float>(nums) << std::endl;
    std::cout << "avg doubles: " << average<double>(nums) << std::endl;
    
    return 0;
}
```

Есть такое эвристическое правило: если вы складываете *10^N* значений, то теряете *N* десятичных знаков точности. Так что при сложении тысячи (*10^3*) чисел теряются три десятичных знака точности. Если складывать миллион (*10^6*) чисел, то теряются шесть десятичных знаков (а у `float` их всего семь). Есть простое решение - выполнять вычисления с числами двойной точности. Кроме того, существуют [численно стабильные способы сложения большого количества значений](https://en.wikipedia.org/wiki/Kahan_summation_algorithm). 

## Другие особенности

### Отношение порядка

Не стоит забывать, что числа с плавающей точкой - это не только привычные действительные числа, но и специальные числа ноль (со знаком), бесконечность (со знаком), неопределённость.

Это накладывает некоторые ограничения на сортируемость чисел с плавающей точкой. Отношение порядка задано не на всём множестве допустимых значений (NaN не сортируем). 

Допустим из двух значений нам нужно выбрать минимальное. В Си это можно сделать одним из следующих способов:

1. x < y? x: y
2. x <= y? x: y
3. x > y? y: x
4. x >= y? y: x


Часто компилятор считает их эквивалентными и всегда использует первый вариант, так как он выполняется за одну инструкцию процессора. Но если мы учтем ±0 и NaN, эти операции никак не эквивалентны:

| x    | y    | x < y? x: y | x <= y? x: y | x > y? y: x | x >= y? y: x |
| ---- | ---- | ----------- | ------------ | ----------- | ------------ |
| +0   | -0   | -0          | +0           | +0          | -0           |
| NaN  | 1    | 1           | 1            | NaN         | NaN          |

## Задачи

https://codeforces.com/problemset/problem/691/C

https://acm.timus.ru/problem.aspx?space=1&num=1020

https://acm.timus.ru/problem.aspx?space=1&num=1192

