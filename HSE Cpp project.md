---
share: true
---
В данном файле (по ссылке) будет конспект семинаров с советами по написанию проекта.
При вопросах, предложениях или ошибках писать в тг [**@CinSGo**](https://t.me/CinSGo) 

### Важные пункты
1) Всё в разных файликах. ВАЖНО: надо сделать такую архитектуру, где при изменении одного файлика, другие затрагиваться не будут  
2) Лучше реализовывать классами  
3) Плюс балл за юнитесты (тесты на все критичные моменты)  
   Для тестов пишется отдельный CMakeLists  

### PIPELINE (шаги программы)

1) Прочитать и распарсить аргументы из CLI  
2) Прочитать BMP и преобразовать в удобный формат (Сохраняем в какую-то структуру)  
3) Создать из распарсенных аргументов последовательность фильтров  
4) Применить последовательность к изображению  
5) Записать изображение в BMP и сохранить  

### CLI, прочитать и распарсить аргументы

Parser надо делать независимым от реализации фильтров (т.е. плохо если в парсере написано: читаем аргументы фильтров и сразу создаём их)

**ПЛОХО:**
``` cpp
if (есть аргумент) {}
if (аргумент == "crop") {...}
// ...
if (аргумент == "gaussian") {...}
// ...
```
т.к. нарушается неделимость

**НАДО:**
Создать унифицированную структуру, типо

CLI Parser (struct):
- `FilterDescriptor;`
- `string name;`
- `vector<string>`
- `args`
тогда будет 
```cpp
if (есть аргумент) {
	создать FilterDiscription{
		name = имя фильтра;
		args = аргументы фильтра;
	}
}
```
тогда на **Выходе из парсера**:
- либо `Error`
- либо `Vector<FilterDiscription>`

### ТЕСТЫ

```cpp
#include <cstdint>
``` 
НО могут быть проблемы с множественным инклюдом (наследием, например)

**СТАРЫЙ ФИКС:**
```cpp
#ifndef NAME

#include <cstdint>
#include ...

#endif
``` 

**НОВЫЙ ФИКС:**
```cpp
#progma once
```
!!!НУЖНА В .h ФАЙЛАХ В **САМОЙ ПЕРВОЙ СТРОКЕ**!!!

### PARSER
- если всё **ОК**, выдаём: `vector<FileDescriptor>`
- если **НЕТ**, то выдаёт ошибку, а **КОД ПАДАТЬ НЕ ДОЛЖЕН** (по условию проекта)
Фиксяться ошибки: 

- Первый способ лучше (в нем можно передать больше инфы), но сложнее
- Второй легче, но хуже
##### СПОСОБ 1: исключения (это в `main(int argc, char *arg[])`):
```cpp
try {
	// запустить пайплайн (че там надо тебе)
} catch (std::exception& e) {
	std::cout << e.what(); 
	// метод what говорит что произошло, по ошибка (типо what the freak)
} catch (...) {
	// какой-то вывод
}
```

Если передали что-то не то или ещё где пользователь накосячил, то мы можем сами вывести что не то:  
**ЛУЧШЕ НОВЫЙ ФАЙЛ ДЛЯ ВСЕХ СВОИХ ЭКСЕПШИОН**
```cpp
class AppException : public std::exception {
	public:
		consy char *what() const noexception override {
		return "ОШИБКА!!!"
		} // если хотим выводить, то надо переопределить what
};

// !!!В ТАКОМ СЛУЧАЕ ЛУЧШЕ СОЗДАТЬ ОДИН БАЗОВЫЙ КЛАСС И ОТ НЕГО НАСЛЕДОВАТЬСЯ!!!

class InvalidFilterNameException : public AppException2 {
	public:
		InvalidFilterNameException(const std::string& arg_name):
			AppException2("invalid filter name: " + arg_name) {};
};

class AppException2 : public std::runtime_error {
	public:
		AppException2(): std::runetime_error("ОШИБКА В IMAGE_PROCESSOR!!!!") 
		//ошибка как пример
};

int main(int argc, char *arg[]) {
	try {
	// запустить пайплайн (че там надо тебе)
	} catch (AppException& e) {
		// !!!!СВОИ ЭКСЕПШИОНЫ!!!!
	} catch (std::exception& e) {
		std::cout << e.what(); 
		// метод what говорит что произошло, по ошибка (типо what the fuck)
	} catch (...) {
		// какой-то вывод
	}
}
```

##### СПОСОБ 2:
```cpp
#include <unordered_map>

enum class ERROR_CODE {
	OK,
	INVALID_FILTER_NAME,
	INVALID_ARGUMENT_NUMBER
};

std::unordered_map<ERROR_CODE, std::string> error_code_to_message = {
	{ERROR_CODE::OK, ""}
	{ERROR_CODE::INVALID_FILTER_NAME, "invalid filter name"}
	{ERROR_CODE::INVALID_ARGUMENT_NUMBER, "invalid argument number"}
};

ERROR_CODE parse() {
	// ...
}
```


### Image

- BMP -> Image
- BMP <- Image
BMP нужен чтоб **читать из файла и записывать в файл**

Фильтры лучше применять к обобщённому (Image) 

Также лучше создать класс для Image. В нём есть `width`, `height`, `vector<vector<Pixel>>` 

```cpp
struct Pixel{
	uint_8 R;
	uint_8 G;
	uint_8 B;
}
```

### BMP

1) Чтение и запись бинарных файлов
```cpp
#pragma pack(push, 1) //ЧТОБ НЕ БЫЛО ВЫРАВНИВАНИЯ ТИПОВ ПО БИТАМ
struct BMPHeader {
	uint16_t id_field;
	uint32_t bpm_size
	//...
};
#pragma pack(pop)

int main(int argc, char *argv[]) {
	std::ifstream input("input.bmp", std::ios::binary);
	BMPHeader bmp_header;
	input.read(reinterpret_cast<char *>(&bmp_header), sizeof(BMPHeader));
	input.seekg(offset, std::ios::beg);
	input.ignore();
	//...
	std::ofstream("output.bmp", std::ios::binary);
	input.write();
	input.seekp();
}
```


### Наследование (для пункта 3 и 4)

```cpp
class Base {
public:
	Base(int value) : base_value_(value) {}
	
private:
	int base_value_;
};

class Derived : public Base {
public:
	Derived(int derived_value, int base_value)
		: derived_value_(derived_value) {}
private:
//
	int derived_value;
}

// Derived [base fields][derived fields]
```

Поля в наследниках сохраняются, НО **сначала идут унаследованные, а потом личные наследника**. Т.е. конструкторы идут от самого общего, к самому частному

#### ТИПЫ НАСЛЕДОВАНИЯ:
1) Single inheritance (единичное наследование)  
   Base -> Derived  
2) Miltilevel inheritance (множественное наследование)  
   Base -> Derived -> DDer -> ...  
3) Multiple (т.е. наследование возможно от нескольких классов)  
   Base1 + Base2 -> Derived  
   ```cpp
    class Derived : public Base1, public Base2 {
    public:
	   Derived(int derived_value, int base_value)
		   : Base1(base_value), derived_value_(derived_value) {}
    }
   ```
4) Hierinchical
   Base -> Derived1 and Derived2
5) Hybrid
   Просто смешанное всё

#### Связь с типом поля (public, protected, private)

**Сверху типы полей Base**
**Слева тип наследования**

|           | **public** | **protected** |
| --------- | ---------- | ------------- |
| **public**    | public     | protected     |
| **protected** | protacted  | protacted     |
| **private**   | private    | private       |

#### Virtual
```cpp
class Animal {
public:
	Animal(int value) {
		std::cout << "Animal constructor called with value" << value << std::endl;
	}
};

class Dog: virtual public Animal {  
// !!ВАЖНО!! virtual нужен, т.к. мы в CatDog наследуем Animal **дважды**
// virtual добавляется в родителя, НЕ в прародителя
// virtual надо добавлять ВО ВСЕХ РОДИТЕЛЕЙ, которые унаследуются от одного и того же класса
public:
	Dog(int value): Animal(value) {
		std::cout << "Dog constructor called with value" << value << std::endl;
	}
};

class Cat: virtual public Animal {
// !!ВАЖНО!! virtual нужен, т.к. мы в CatDog наследуем Animal **дважды**
// virtual добавляется в родителя, НЕ в прародителя
// virtual надо добавлять ВО ВСЕХ РОДИТЕЛЕЙ, которые унаследуются от одного и того же класса
public:
	Cat(int value): Animal(value) {
		std::cout << "Cat constructor called with value" << value << std::endl;
	}
};

class CatDog : public Cat, public Dog {
public:
	CatDog(int value) : Animal(value), Cat(value), Dog(value) {
// вызываем конструктор прародителя (т.е. Animal(value)), т.к. из-за virtual конструкторы Cat и Dog игнорируются CatDog
		std::cout << "CatDog constructor called with value" << value << std::endl;
	}	
}
```

#### Полиморфизм
— это способность объектов, имеющих одинаковый интерфейс или базовый класс, вести себя по-разному в зависимости от их типа
```cpp
template <typename T>
T add(T a, T b) {
	return a + b;
}

int add(int a, int b) {
	return a + b;
}

std::string add(std::string a, std::string b) {
	return a + b;
}

int main() {
	add(a, b);
}
```

```cpp
ТУТ ОБНОВА БУДЕТ ЧУТКА ПОЗЖЕ (ПРО SHAPE)
```