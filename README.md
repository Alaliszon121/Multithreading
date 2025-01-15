# Wielowątkowość w C++

![clock_processor](https://github.com/user-attachments/assets/b6a0f8ec-6aa9-4d28-a274-7b0ccaeadc11)
![threads](https://github.com/user-attachments/assets/76d5fc7e-5408-4d7c-b739-260b76c0a8b9)


## 1. Prosty przykład użycia wątków

```cpp
#include <iostream>
#include <thread>

void doWork(int id) {
    for (int i = 0; i < 5; ++i) {
        std::cout << "Thread " << id << ": " << i << std::endl;
    }
}

int main() {
    const int numThreads = std::thread::hardware_concurrency();
    std::thread threads[numThreads];

    for (int i = 0; i < numThreads; ++i) {
        threads[i] = std::thread(doWork, i);
    }

    for (int i = 0; i < numThreads; ++i) {
        threads[i].join();
    }

    std::cout << "Koniec programu" << std::endl;
    return 0;
}
```

### Wyjaśnienie:
- Funkcja `doWork` jest uruchamiana w wielu wątkach, każdy z nich wypisuje swoje ID i licznik od 0 do 4.
- `std::thread::hardware_concurrency()` zwraca liczbę dostępnych rdzeni procesora.
- Tworzenie i uruchamianie wątków za pomocą `std::thread`.
- Użycie `join()`, aby poczekać na zakończenie wszystkich wątków.
    
![race_condition](https://github.com/user-attachments/assets/b99454aa-5828-4333-9740-5dd952e42d33)

## 2. Problem współbieżności bez synchronizacji

### Kod:

```cpp
int g_data = 0;

void thread_func()
{
    g_data += 1;    
}
```

### Przykładowe scenariusze wykonania:

1. Wątki działają sekwencyjnie:
    - Wynik: `g_data == 2`
    ``` 
    thread 0: read g_data to reg
    thread 0: add 1 to reg
    thread 0: write reg to g_data
    thread 1: read g_data to reg
    thread 1: add 1 to reg
    thread 1: write reg to g_data
    ```

2. Wątki działają równolegle bez synchronizacji:
    - Wynik: `g_data == 1`
    ``` 
    thread 0: read g_data to reg
    thread 1: read g_data to reg
    thread 0: add 1 to reg
    thread 1: add 1 to reg
    thread 0: write reg to g_data
    thread 1: write reg to g_data
    ```

## 3. Problem z blokadą bez odblokowania

### Kod:

```cpp
int g_data = 0;
Mutex g_mutex;

void thread_func()
{
    g_mutex.lock();
    g_data += 1;
    // missing unlock!
}
```

### Scenariusz:

- Wynik: `g_data == 1`
``` 
thread 0: lock mutex            locked
thread 0: read g_data to reg    |
thread 0: add 1 to reg          |
thread 0: write reg to g_data   |
thread 1: lock mutex            (wait)
```

## 4. Synchronizacja z blokadą

### Kod:

```cpp
int g_data = 0;
Mutex g_mutex;

void thread_func()
{
    g_mutex.lock();
    g_data += 1;
    g_mutex.unlock();
}
```

### Scenariusz:

- Wynik: `g_data == 2`
``` 
thread 0: lock mutex            locked
thread 0: read g_data to reg    |
thread 0: add 1 to reg          |
thread 0: write reg to g_data   |
thread 0: unlock                x
thread 1: lock mutex            locked
thread 1: read g_data to reg    |
thread 1: add 1 to reg          |
thread 1: write reg to g_data   |
thread 1: unlock                x
```

## 5. Użycie `lock_guard` dla automatycznego zarządzania blokadą

### Kod:

```cpp
int g_data = 0;
Mutex g_mutex;

void thread_func()
{
    lock_guard guard(g_mutex);
    g_data += 1;
}
```

### Scenariusz:

- Wynik: `g_data == 2`
``` 
thread 0: lock mutex            locked
thread 0: read g_data to reg    |
thread 1: lock mutex            wait
thread 0: add 1 to reg          |
thread 0: write reg to g_data   |
thread 0: unlock                x
thread 1: lock mutex            locked
thread 1: read g_data to reg    |
thread 1: add 1 to reg          |
thread 1: write reg to g_data   |
thread 1: unlock                x
```

## 6. Użycie zmiennej atomowej

### Kod:

```cpp
atomic<int> g_data(0);

void thread_func()
{
    g_data.increment();
}
```

### Scenariusz:

- Wynik: `g_data == 2`
``` 
thread 0: read, increment, write g_data
thread 1: read, increment, write g_data
```

## 7. Problemy z użyciem zmiennej atomowej

### Kod:

```cpp
atomic<int> g_data(0);

void thread_func()
{
    if(g_data.get() == 0)
    {
        g_data.increment();
    }
}
```

### Scenariusz:

``` 
thread 0: read g_data
thread 1: read g_data
thread 0: compare g_data
thread 0: read, increment, write g_data
thread 1: compare g_data
thread 1: read, increment, write g_data
```

## 8. Synchronizacja przy użyciu muteksu i zmiennej atomowej

### Kod:

```cpp
atomic<int> g_data(0);
Mutex g_mutex;

void thread_func()
{
    g_mutex.lock();
    if(g_data.get() == 0)
    {
        g_data.increment();
    }
    g_mutex.unlock();
}
```

### Scenariusz:

``` 
thread 0: lock, read and compare g_data, unlock
thread 1: lock, read and compare g_data, unlock
thread 0: read, increment, write g_data
thread 1: read, increment, write g_data
```

## 9. Synchronizacja przy użyciu podwójnego sprawdzania

### Kod:

```cpp
atomic<int> g_data(0);
Mutex g_mutex;

void thread_func()
{
    if(g_data.get() == 0)
    {
        g_mutex.lock();
        if(g_data.get() == 0)
        {
            g_data.increment();
        }
        g_mutex.unlock();
    }
}
```

### Scenariusz:

``` 
thread 0: read g_data
thread 1: read g_data
thread 0: compare g_data
thread 1: compare g_data
thread 0: lock, read and compare g_data, read, increment, write g_data, unlock
thread 1: lock, read and compare g_data, read, increment, write g_data, unlock
```
![spinlock](https://github.com/user-attachments/assets/3b967890-80e0-470d-ba9a-1eb6a491fe81)

## 10. Implementacja SpinLock

### Kod:

```cpp
class SpinLock
{
public:
    void Lock()
    {
        for(int i=0; i<20; ++i)
        {
            if(!m_mutex.IsLocked())
                break;
        }
        
        m_mutex.Lock();
    }
    
    void Unlock()
    {
        m_mutex.Unlock();
    }
private:
    Mutex m_mutex;
};

int m_data = 0;
SpinLock m_lock;

void thread_func()
{
    m_lock.Lock();
    m_data += 1;
    m_lock.Unlock();
}
```

### Scenariusz:
``` 
thread 0: read g_data
thread 1: read g_data
thread 0: compare g_data
thread 1: compare g_data
thread 0: lock, read and compare g_data, read, increment, write g_data, unlock
thread 1: lock, read and compare g_data, read, increment, write g_data, unlock

