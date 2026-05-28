# Лабораторная работа №1

## Исследование компилятора GCC, языка ассемблера, Makefile и параллельного выполнения

---

# Цель работы

Изучить:

- процесс компиляции программ;
- генерацию ассемблерного кода компилятором GCC/G++;
- влияние оптимизации на ASM-код;
- работу Makefile;
- основы Git;
- взаимодействие процесса и операционной системы;
- основы параллельного выполнения.

---

# Задание

Необходимо:

1. Написать программу на компилируемом языке.
2. Получить ASM-листинг программы с различными флагами оптимизации.
3. Выполнить анализ ASM-кода.
4. Преобразовать программу в модульную.
5. Создать Makefile.
6. Использовать Git.
7. Реализовать параллельное выполнение.
8. Организовать синхронизацию доступа к общему ресурсу.

---

# Вариант задания

Вычисление факториала числа.

---

# Структура проекта

project/
│
├── main.cpp
├── factorial.cpp
├── factorial.h
├── Makefile
├── result.txt
├── factorial_O0.asm
├── factorial_O3.asm
├── factorial_debug.asm
└── README.md

---

# 1. Реализация программы

## factorial.h

```cpp
#ifndef FACTORIAL_H
#define FACTORIAL_H

unsigned long long factorial(int n);

#endif
```

## factorial.cpp

```cpp
#include "factorial.h"

unsigned long long factorial(int n) {

    if (n < 0)
        return 0;

    unsigned long long result = 1;

    for (int i = 1; i <= n; i++) {
        result *= i;
    }

    return result;
}
```

## main.cpp

```cpp
#include <iostream>
#include <fstream>
#include <thread>
#include <mutex>

#include "factorial.h"

using namespace std;

mutex fileMutex;

void computeFactorial(int n) {

    unsigned long long result = factorial(n);

    lock_guard<mutex> lock(fileMutex);

    ofstream file("result.txt");

    if (file.is_open()) {
        file << result;
        file.close();
    }
}

int main() {

    int n = 10;

    cout << "Calculating factorial of " << n << "..." << endl;

    thread calculator(computeFactorial, n);

    cout << "Parallel thread started..." << endl;
    cout << "Main thread continues working..." << endl;

    calculator.join();

    cout << "Thread finished." << endl;

    unsigned long long result;

    ifstream file("result.txt");

    if (file.is_open()) {
        file >> result;
        file.close();
    }

    cout << n << "! = " << result << endl;

    return 0;
}
```

---

# Компиляция

```bash
g++ main.cpp factorial.cpp -o main.exe
```

# Генерация ASM

```bash
g++ factorial.cpp -O0 -S -o factorial_O0.asm
g++ factorial.cpp -O3 -S -o factorial_O3.asm
g++ factorial.cpp -O0 -g -S -o factorial_debug.asm
```

---

# Анализ ASM-кода (-O0)

```asm
.file   "factorial.cpp"

.text

.globl  _Z9factoriali              ; объявление глобальной функции
.def    _Z9factoriali; .scl 2; .type 32; .endef

.seh_proc   _Z9factoriali

_Z9factoriali:
.LFB0:

    ; ===== ПРОЛОГ ФУНКЦИИ =====

    pushq  %rbp                    ; сохраняем старый базовый указатель
    movq   %rsp, %rbp              ; создаем новый стековый фрейм
    subq   $16, %rsp               ; выделяем память под локальные переменные

    ; сохраняем аргумент функции n
    movl   %ecx, 16(%rbp)          ; n -> стек

    ; ===== ПРОВЕРКА: if (n < 0) return 0 =====

    cmpl   $0, 16(%rbp)            ; сравниваем n и 0
    jns    .L2                     ; если n >= 0 → продолжаем

    movl   $0, %eax                ; eax = 0 (результат)
    jmp    .L3                     ; переход к выходу

.L2:

    ; ===== ИНИЦИАЛИЗАЦИЯ =====

    movq   $1, -8(%rbp)            ; result = 1
    movl   $1, -12(%rbp)           ; i = 1

    jmp    .L4                     ; переход к проверке условия цикла

.L5:

    ; ===== ТЕЛО ЦИКЛА =====

    movl   -12(%rbp), %eax         ; eax = i
    cltq                            ; преобразование int -> long long

    movq   -8(%rbp), %rdx          ; rdx = result

    imulq  %rdx, %rax              ; rax = result * i

    movq   %rax, -8(%rbp)          ; result = result * i

    addl   $1, -12(%rbp)           ; i++

.L4:

    ; ===== ПРОВЕРКА УСЛОВИЯ ЦИКЛА =====

    movl   -12(%rbp), %eax         ; eax = i

    cmpl   16(%rbp), %eax          ; сравниваем i и n

    jle    .L5                     ; если i <= n → повторяем цикл

    ; ===== ВОЗВРАТ РЕЗУЛЬТАТА =====

    movq   -8(%rbp), %rax          ; загружаем result в регистр возврата

.L3:

    ; ===== ЭПИЛОГ ФУНКЦИИ =====

    addq   $16, %rsp               ; освобождаем стек
    popq   %rbp                    ; восстанавливаем rbp
    ret                            ; выход из функции
```

---

# Makefile

```makefile
CXX = g++
CXXFLAGS = -Wall -std=c++11

all: main.exe

main.exe: main.o factorial.o
	$(CXX) main.o factorial.o -o main.exe

main.o: main.cpp factorial.h
	$(CXX) $(CXXFLAGS) -c main.cpp

factorial.o: factorial.cpp factorial.h
	$(CXX) $(CXXFLAGS) -c factorial.cpp

asm:
	$(CXX) factorial.cpp -O0 -S -o factorial_O0.asm
	$(CXX) factorial.cpp -O3 -S -o factorial_O3.asm
	$(CXX) factorial.cpp -O0 -g -S -o factorial_debug.asm

clean:
	rm -f *.o *.exe *.asm result.txt

run: main.exe
	./main.exe

.PHONY: all asm clean run
```

---

# Git

```bash
git init
git add .
git commit -m "Initial commit"
git status
git log
```

---

# Заключение

В ходе лабораторной работы были изучены:

- процесс компиляции программ;
- генерация ASM-кода;
- влияние оптимизации GCC;
- структура ASM-программы;
- Makefile;
- Git;
- основы многопоточности;
- синхронизация доступа к общему ресурсу.
