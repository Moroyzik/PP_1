#include <iostream>
#include <atomic>
#include <map>
#include <vector>
#include <thread>
#include <shared_mutex>
#include <mutex>

using ul = unsigned long long;

int main()
{
    //матрица
    std::vector<std::vector<int>> matx = {{3,2,3},
                                          {3,2,1},
                                          {3,2,5},
                                          };
    std::shared_mutex mutxVal;//мьютекс
    int maxVal{std::numeric_limits<int>::min()};// сюда будем записывать макс значение
    std::atomic<ul> counter{0};//атомарный счётчик
    // попытка задать значение в аргументы передаём предлагаемое
    auto setVal = [&](int newVal)
    {
        // уникальная блокировка (только один поток может её установить)
        std::unique_lock lock(mutxVal);
        //задаём значение
        if(newVal>maxVal)
        {
            maxVal = newVal;
            //счётчику присваиваем 1
            counter.store(1);
        }
    };
    // мьютекс задаёт порядок памяти.
    //поэтому мы можем получить более свежее значение с мьютексом, чем через maxVal
    auto getVal = [&](){
        // разделяемая блокировка (много потоков могут войти сюда)
        std::shared_lock sherLock(mutxVal);
        return maxVal;
    };
    // попытка увеличить счётчик.
    auto compareIncr = [&](int val){
        std::shared_lock sherLock(mutxVal);
        if(maxVal==val)
        ++counter;
    };
    // получить рекомендуемое количество потоков
    const auto hcc = std::thread::hardware_concurrency();
    // может вернуть 0, если 0, то использовать 2
    const auto MAX_THREADS_NUMBER{hcc?hcc:2};
    // если кудахтер поддерживает работу без блокировок, то напечатать "чудесно"
    std::cout << (std::atomic_is_lock_free(&counter)?"чудесно\n": "печально\n");
    // вектор для управления потоками
    std::vector<std::thread> threads(MAX_THREADS_NUMBER);
    // счётчик количества потоков
    ul tNum{MAX_THREADS_NUMBER};
    // пройтись по всем строкам
    for(auto& line: matx)
    {
        //пройтись по элементам
        for(auto& element: line)
        {
            // если количество работающих потоков большое, то подождать завершения всех:
            if(!tNum)
            {
                //пройтись по потокам
                for(auto& td: threads)
                    // можно ли завершить поток? (нельзя, если он отсоеденился)
                    if(td.joinable())
                        td.join(); // тогда завершаем поток
                //освободить вектор. Потоки завершены.
                threads.clear();
                // можно создавать новые
                tNum = MAX_THREADS_NUMBER;
            }
            //создаём новый поток
            std::thread thread([&,element](){
                //сравниваем текущий элемент с "кэшем": примерным значенимем максимума
                if(element>=maxVal)
                {
                    //уточняем максимальное значение
                    auto val(getVal());
                    //если уточнённое макс больше текущего, то выходим
                    if(element<val)
                        return;
                    //если равен, то пытаемся увеличить счётчик
                    if(element==val)
                        compareIncr(element);
                    //если больше, то пытаемся обновить максимум
                    if(element>val)
                        setVal(element);
                }
            });
            // переместить ссылку потока в вектор потоков
            threads.push_back(std::move(thread));
            // запустили один поток, счётчик доступных уменьшаем
            --tNum;
        }
    }
    // если ещё остались потоки, то завершаем их
    for(auto& td: threads)
        if(td.joinable())
            td.join();
    threads.clear();
    std::cout<<"max value("<<getVal()<<") occurs "<<
        counter.load(std::memory_order_relaxed)<<" time(s)";
    return 0;
}
