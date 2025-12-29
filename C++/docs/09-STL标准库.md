# STL 标准库

STL（Standard Template Library）是 C++ 标准库的核心部分，提供了容器、迭代器、算法等强大的工具。

## 容器

### 序列容器

#### std::vector

```cpp
#include <iostream>
#include <vector>

int main() {
    // 创建 vector
    std::vector<int> vec;
    std::vector<int> vec2(5, 10);  // 5 个元素，每个都是 10
    std::vector<int> vec3{1, 2, 3, 4, 5};  // 初始化列表
    
    // 添加元素
    vec.push_back(1);
    vec.push_back(2);
    vec.push_back(3);
    
    // 访问元素
    std::cout << vec[0] << std::endl;        // 1（不检查边界）
    std::cout << vec.at(1) << std::endl;    // 2（检查边界）
    std::cout << vec.front() << std::endl;  // 1
    std::cout << vec.back() << std::endl;   // 3
    
    // 大小
    std::cout << "Size: " << vec.size() << std::endl;
    std::cout << "Empty: " << vec.empty() << std::endl;
    
    // 遍历
    for (size_t i = 0; i < vec.size(); ++i) {
        std::cout << vec[i] << " ";
    }
    std::cout << std::endl;
    
    for (int elem : vec) {
        std::cout << elem << " ";
    }
    std::cout << std::endl;
    
    // 插入和删除
    vec.insert(vec.begin() + 1, 99);  // 在位置 1 插入 99
    vec.erase(vec.begin() + 2);       // 删除位置 2 的元素
    vec.pop_back();                   // 删除最后一个元素
    
    // 清空
    vec.clear();
    
    return 0;
}
```

#### std::deque

```cpp
#include <iostream>
#include <deque>

int main() {
    std::deque<int> dq;
    
    // 两端操作
    dq.push_back(1);
    dq.push_back(2);
    dq.push_front(0);
    dq.push_front(-1);
    
    // 访问
    std::cout << dq.front() << std::endl;  // -1
    std::cout << dq.back() << std::endl;   // 2
    
    // 删除
    dq.pop_front();
    dq.pop_back();
    
    return 0;
}
```

#### std::list

```cpp
#include <iostream>
#include <list>

int main() {
    std::list<int> lst{1, 2, 3, 4, 5};
    
    // 插入
    lst.push_front(0);
    lst.push_back(6);
    
    // 删除
    lst.pop_front();
    lst.pop_back();
    
    // 遍历（只能使用迭代器）
    for (auto it = lst.begin(); it != lst.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
    
    // 范围 for
    for (int elem : lst) {
        std::cout << elem << " ";
    }
    std::cout << std::endl;
    
    // 删除特定值
    lst.remove(3);
    
    // 排序
    lst.sort();
    
    return 0;
}
```

### 关联容器

#### std::map

```cpp
#include <iostream>
#include <map>
#include <string>

int main() {
    // 创建 map
    std::map<std::string, int> scores;
    
    // 插入元素
    scores["Alice"] = 95;
    scores["Bob"] = 87;
    scores.insert({"Charlie", 92});
    scores.emplace("David", 88);
    
    // 访问元素
    std::cout << scores["Alice"] << std::endl;  // 95
    std::cout << scores.at("Bob") << std::endl;  // 87（检查存在）
    
    // 查找
    auto it = scores.find("Alice");
    if (it != scores.end()) {
        std::cout << "Found: " << it->second << std::endl;
    }
    
    // 检查存在
    if (scores.count("Alice") > 0) {
        std::cout << "Alice exists" << std::endl;
    }
    
    // 遍历
    for (const auto& [name, score] : scores) {  // C++17
        std::cout << name << ": " << score << std::endl;
    }
    
    // 删除
    scores.erase("Bob");
    
    return 0;
}
```

#### std::set

```cpp
#include <iostream>
#include <set>

int main() {
    std::set<int> s{3, 1, 4, 1, 5, 9, 2, 6};
    
    // 插入
    s.insert(7);
    s.emplace(8);
    
    // 查找
    if (s.find(5) != s.end()) {
        std::cout << "5 found" << std::endl;
    }
    
    // 遍历（自动排序）
    for (int elem : s) {
        std::cout << elem << " ";  // 1 2 3 4 5 6 7 8 9
    }
    std::cout << std::endl;
    
    // 删除
    s.erase(5);
    
    return 0;
}
```

#### std::unordered_map（C++11）

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

int main() {
    std::unordered_map<std::string, int> scores;
    
    scores["Alice"] = 95;
    scores["Bob"] = 87;
    scores["Charlie"] = 92;
    
    // 查找（O(1) 平均时间复杂度）
    auto it = scores.find("Alice");
    if (it != scores.end()) {
        std::cout << "Found: " << it->second << std::endl;
    }
    
    // 遍历（无序）
    for (const auto& [name, score] : scores) {
        std::cout << name << ": " << score << std::endl;
    }
    
    return 0;
}
```

### 容器适配器

#### std::stack

```cpp
#include <iostream>
#include <stack>

int main() {
    std::stack<int> st;
    
    // 入栈
    st.push(1);
    st.push(2);
    st.push(3);
    
    // 访问栈顶
    std::cout << st.top() << std::endl;  // 3
    
    // 出栈
    st.pop();
    
    // 检查空
    while (!st.empty()) {
        std::cout << st.top() << " ";
        st.pop();
    }
    std::cout << std::endl;
    
    return 0;
}
```

#### std::queue

```cpp
#include <iostream>
#include <queue>

int main() {
    std::queue<int> q;
    
    // 入队
    q.push(1);
    q.push(2);
    q.push(3);
    
    // 访问队首和队尾
    std::cout << q.front() << std::endl;  // 1
    std::cout << q.back() << std::endl;   // 3
    
    // 出队
    q.pop();
    
    return 0;
}
```

#### std::priority_queue

```cpp
#include <iostream>
#include <queue>
#include <vector>

int main() {
    // 最大堆（默认）
    std::priority_queue<int> max_heap;
    
    max_heap.push(3);
    max_heap.push(1);
    max_heap.push(4);
    max_heap.push(1);
    max_heap.push(5);
    
    while (!max_heap.empty()) {
        std::cout << max_heap.top() << " ";  // 5 4 3 1 1
        max_heap.pop();
    }
    std::cout << std::endl;
    
    // 最小堆
    std::priority_queue<int, std::vector<int>, std::greater<int>> min_heap;
    min_heap.push(3);
    min_heap.push(1);
    min_heap.push(4);
    
    while (!min_heap.empty()) {
        std::cout << min_heap.top() << " ";  // 1 3 4
        min_heap.pop();
    }
    std::cout << std::endl;
    
    return 0;
}
```

## 迭代器

### 迭代器类型

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <map>

int main() {
    std::vector<int> vec{1, 2, 3, 4, 5};
    
    // 正向迭代器
    for (auto it = vec.begin(); it != vec.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
    
    // 反向迭代器
    for (auto it = vec.rbegin(); it != vec.rend(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
    
    // 常量迭代器
    for (auto it = vec.cbegin(); it != vec.cend(); ++it) {
        // *it = 10;  // 错误！不能修改
        std::cout << *it << " ";
    }
    std::cout << std::endl;
    
    // map 迭代器
    std::map<std::string, int> scores{{"Alice", 95}, {"Bob", 87}};
    for (auto it = scores.begin(); it != scores.end(); ++it) {
        std::cout << it->first << ": " << it->second << std::endl;
    }
    
    return 0;
}
```

### 迭代器操作

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> vec{1, 2, 3, 4, 5};
    
    // 距离
    auto dist = std::distance(vec.begin(), vec.end());
    std::cout << "Distance: " << dist << std::endl;  // 5
    
    // 前进
    auto it = vec.begin();
    std::advance(it, 2);
    std::cout << *it << std::endl;  // 3
    
    // 下一个/上一个
    auto next_it = std::next(it);
    auto prev_it = std::prev(it);
    
    return 0;
}
```

## 算法

### 查找算法

```cpp
#include <iostream>
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec{1, 2, 3, 4, 5, 3, 2, 1};
    
    // find：查找第一个匹配的元素
    auto it = std::find(vec.begin(), vec.end(), 3);
    if (it != vec.end()) {
        std::cout << "Found at position: " 
                  << std::distance(vec.begin(), it) << std::endl;
    }
    
    // find_if：使用谓词查找
    auto it2 = std::find_if(vec.begin(), vec.end(), 
                            [](int x) { return x > 4; });
    if (it2 != vec.end()) {
        std::cout << "First element > 4: " << *it2 << std::endl;
    }
    
    // count：计数
    int count = std::count(vec.begin(), vec.end(), 3);
    std::cout << "Count of 3: " << count << std::endl;
    
    // binary_search：二分查找（需要有序）
    std::sort(vec.begin(), vec.end());
    bool found = std::binary_search(vec.begin(), vec.end(), 3);
    std::cout << "3 found: " << found << std::endl;
    
    return 0;
}
```

### 排序算法

```cpp
#include <iostream>
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec{3, 1, 4, 1, 5, 9, 2, 6};
    
    // sort：排序
    std::sort(vec.begin(), vec.end());
    for (int x : vec) {
        std::cout << x << " ";  // 1 1 2 3 4 5 6 9
    }
    std::cout << std::endl;
    
    // 自定义比较
    std::sort(vec.begin(), vec.end(), std::greater<int>());
    for (int x : vec) {
        std::cout << x << " ";  // 9 6 5 4 3 2 1 1
    }
    std::cout << std::endl;
    
    // partial_sort：部分排序
    std::vector<int> vec2{3, 1, 4, 1, 5, 9, 2, 6};
    std::partial_sort(vec2.begin(), vec2.begin() + 3, vec2.end());
    // 前 3 个元素已排序
    
    return 0;
}
```

### 变换算法

```cpp
#include <iostream>
#include <algorithm>
#include <vector>
#include <numeric>

int main() {
    std::vector<int> vec{1, 2, 3, 4, 5};
    std::vector<int> result(5);
    
    // transform：变换
    std::transform(vec.begin(), vec.end(), result.begin(),
                   [](int x) { return x * 2; });
    for (int x : result) {
        std::cout << x << " ";  // 2 4 6 8 10
    }
    std::cout << std::endl;
    
    // accumulate：累加
    int sum = std::accumulate(vec.begin(), vec.end(), 0);
    std::cout << "Sum: " << sum << std::endl;  // 15
    
    // accumulate：累乘
    int product = std::accumulate(vec.begin(), vec.end(), 1,
                                  [](int a, int b) { return a * b; });
    std::cout << "Product: " << product << std::endl;  // 120
    
    return 0;
}
```

### 其他常用算法

```cpp
#include <iostream>
#include <algorithm>
#include <vector>

int main() {
    std::vector<int> vec{1, 2, 3, 4, 5};
    
    // reverse：反转
    std::reverse(vec.begin(), vec.end());
    // vec: 5 4 3 2 1
    
    // rotate：旋转
    std::rotate(vec.begin(), vec.begin() + 2, vec.end());
    // vec: 3 2 1 5 4
    
    // unique：去重（需要先排序）
    std::vector<int> vec2{1, 2, 2, 3, 3, 3, 4};
    auto it = std::unique(vec2.begin(), vec2.end());
    vec2.erase(it, vec2.end());
    // vec2: 1 2 3 4
    
    // remove_if：条件删除
    std::vector<int> vec3{1, 2, 3, 4, 5};
    vec3.erase(std::remove_if(vec3.begin(), vec3.end(),
                              [](int x) { return x % 2 == 0; }),
               vec3.end());
    // vec3: 1 3 5
    
    return 0;
}
```

## 函数对象和 Lambda

```cpp
#include <iostream>
#include <algorithm>
#include <vector>
#include <functional>

int main() {
    std::vector<int> vec{1, 2, 3, 4, 5};
    
    // Lambda 表达式
    std::for_each(vec.begin(), vec.end(),
                  [](int x) { std::cout << x << " "; });
    std::cout << std::endl;
    
    // 函数对象
    std::greater<int> greater_obj;
    std::sort(vec.begin(), vec.end(), greater_obj);
    
    // bind：绑定参数
    auto add = [](int a, int b) { return a + b; };
    auto add_10 = std::bind(add, std::placeholders::_1, 10);
    std::cout << add_10(5) << std::endl;  // 15
    
    return 0;
}
```

## 实践练习

### 练习 1：统计单词频率

```cpp
#include <iostream>
#include <map>
#include <string>
#include <sstream>

int main() {
    std::string text = "hello world hello cpp world";
    std::map<std::string, int> word_count;
    
    std::istringstream iss(text);
    std::string word;
    
    while (iss >> word) {
        word_count[word]++;
    }
    
    for (const auto& [w, count] : word_count) {
        std::cout << w << ": " << count << std::endl;
    }
    
    return 0;
}
```

### 练习 2：使用算法处理数据

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>

int main() {
    std::vector<int> numbers{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    
    // 过滤偶数
    std::vector<int> evens;
    std::copy_if(numbers.begin(), numbers.end(),
                 std::back_inserter(evens),
                 [](int x) { return x % 2 == 0; });
    
    // 计算平方和
    int sum_of_squares = std::accumulate(
        numbers.begin(), numbers.end(), 0,
        [](int sum, int x) { return sum + x * x; }
    );
    
    std::cout << "Sum of squares: " << sum_of_squares << std::endl;
    
    return 0;
}
```

## 容器选择指南

| 容器 | 特点 | 适用场景 |
|------|------|---------|
| **vector** | 动态数组，随机访问快 | 需要随机访问，频繁在末尾操作 |
| **deque** | 双端队列 | 需要在两端操作 |
| **list** | 双向链表 | 需要频繁插入/删除 |
| **map** | 有序键值对 | 需要按键排序 |
| **set** | 有序集合 | 需要有序且唯一 |
| **unordered_map** | 哈希表 | 需要快速查找（O(1)） |
| **unordered_set** | 哈希集合 | 需要快速查找且唯一 |

## 与其他语言对比

| 特性 | C++ STL | Java Collections | Python | Go |
|------|---------|------------------|--------|-----|
| **容器** | ✅ | ✅ | ✅ | ✅ |
| **迭代器** | ✅ | ✅ | ✅ | ✅ |
| **算法** | ✅ | 部分 | 部分 | 部分 |
| **泛型** | 模板 | 泛型 | 类型提示 | 无 |

## 下一步

现在你已经掌握了 STL 标准库，可以继续学习：

1. **[异常处理](./10-异常处理.md)** - 学习错误处理
2. **[并发编程](./11-并发编程.md)** - 掌握多线程编程
3. **[现代 C++ 特性](./12-现代C++特性.md)** - 了解最新特性

祝你学习愉快！🎉

