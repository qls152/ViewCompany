# ViewCompany

# 问题

> 给定一个长度为N(N=100, 000)的整数数组S，有M(M >= 2)个woker并发访问S并更新S
> 每个worker重复10，000次如下操作:
> 1) 随机生成i，j, 0 <= i, j <= 100000
> 2) 更新S 使得S(j) = S(i) + S(i+1) + S(i+2), 如果i+1，i+2越界，则用(i+1) % N 以及（i+2) % N代替

# 方案一

根据题目可知，该题为一个多线程相关问题。因此最直观的解法为

1. 利用一个全局锁 std::mutex保护数组S，使所有操作序列化

2. 需要保证随机数生成是线程安全的

其代码实现如下

```c++
#include <thread>
#include <vector>
#include <mutex>
#include <random>
#include <functional>
#include <numeric>
#include <algorithm>

int intRand(const int & min, const int & max) {
  static thread_local std::mt19937 generator;
  std::uniform_int_distribution<int> distribution(min,max);
  return distribution(generator);
}
constexpr int N{100000};
constexpr int M{3};
constexpr int Loop{10000};
int main() {
  
  std::vector<int> S(N);
  std::iota(S.begin(), S.end(), 0);

  std::mutex mut;
  std::vector<std::thread> thr_vec;
  auto thread_func = [](std::mutex& tmut, std::vector<int>& nums) {
    for (int i = 0; i < Loop; ++i) {
      int i = intRand(0, N);
      int j = intRand(0, N);
      {
        std::lock_guard<std::mutex> lgd(tmut);
        nums[j] = nums[i % N] + nums[(i+1) % N] + nums[(i+2) % N];
      }
    }
    
    return;
  };

  thr_vec.reserve(M);
  for (int i = 0; i < M; ++i) {
	   thr_vec.push_back(std::thread{thread_func, std::ref(mut), std::ref(S)});
  }

  for (auto& thr : thr_vec) {
    if (thr.joinable()) {
      thr.join();
    }
  }
  return 0;

}
```

**该方案耗时分析：**

假设std::mutex解锁/加锁耗时为t1，

假设std::mutex在线程间产生锁竞争的时间为t2

按最坏情况下计划耗时为：M * Loop * t1 + t2 * M(M-1) / 2 * Loop * Loop

**Note**

**由于采用一个锁，所以此刻不会产生死锁**

# 方案二

为了减小锁竞争，考虑每个元素都用一个锁保护。

这种场景可能导致耗尽系统资源(锁太多)

相应的核心代码实现如下

```c++
std::vector<std::mutex> mut_vec;
auto thread_func = [](std::vector<std::mutex> & tmut, std::vector<int>& nums) {
    for (int i = 0; i < Loop; ++i) {
      int i = intRand(0, N);
      int j = intRand(0, N);
      int num1{};
      int num2{};
      int num3{};
      {
        std::lock_guard<std::mutex> lgd(tmut[i % N]);
	num1 = nums[i % N];
      }

      {
        std::lock_guard<std::mutex> lgd(tmut[(i+1) % N]);
	num2 = nums[(i+1) % N];
      }

      {
        std::lock_guard<std::mutex> lgd(tmut[(i+2) % N]);
	num3 = nums[(i+2) % N];
      }
      {
        std::lock_guard<std::mutex> lgd(tmut[j]);
	nums[j] = num1 + num2 + num3;
      }
    }
    
    return;
  };
```

**该方案耗时分析**

假设std::mutex解锁/加锁耗时为t1，

假设std::mutex在线程间产生锁竞争的时间为t2

由于题目中给出 并发workers同时访问同一个元素的概率很低，假设不会发生则

按最坏情况下计划耗时为：M * Loop * t1 * 4 + t2 * M(M-1) / 2 * Loop * 4

**Note**

**该方案不会产生死锁**

# 方案三

采用两阶段锁策略，也即先获得各个锁，当完成计算后释放锁

相应的核心代码实现如下

```c++
std::vector<std::mutex> mut_vec;
auto thread_func = [](std::vector<std::mutex> & tmut, std::vector<int>& nums) {
    for (int i = 0; i < Loop; ++i) {
      int i = intRand(0, N);
      int j = intRand(0, N);
      {
        std::lock_guard<std::mutex> lgd(tmut[i % N]);
        std::lock_guard<std::mutex> lgd(tmut[(i+1) % N]);
	std::lock_guard<std::mutex> lgd(tmut[(i+2) % N]);
	std::lock_guard<std::mutex> lgd(tmut[j]);
	nums[j] = nums[i] + nums[(i+1) % N] + nums[(i+2) % N];
      }
      
    }
    
    return;
  };
```

**Note**

当j处于i和i+2中间时，上述在单线程中会产生死锁

此种解决方案如下

```c++
      int sum{0};
      {
        std::lock_guard<std::mutex> lgd(tmut[i]);
        std::lock_guard<std::mutex> lgd(tmut[(i+1) % N]);
	std::lock_guard<std::mutex> lgd(tmut[(i+2) % N]);
	
	sum = nums[i] + nums[(i+1) % N] + nums[(i+2) % N];
      }
      std::lock_guard<std::mutex> lgd(tmut[j]);
      nums[j] = sum;
```

由于上述加锁规则为层次锁，也即各个锁的加锁顺序固定，按照(i) % N 的顺序加锁，故不会产生死锁

# 方案四

如果给数组中每个元素都加锁，那么锁的资源会很多，同时锁越多 越容易产生锁竞争，因此可以考虑适当增加锁的粒度，也即每3个元素共享一个锁，譬如0-2 锁1， 3-5 锁2等

根据上述方式可以有如下实现

```c++
std::vector<std::mutex> mut_vec(N/3 + 1);
auto thread_func = [](std::vector<std::mutex> & tmut, std::vector<int>& nums) {
    for (int i = 0; i < Loop; ++i) {
      int i = intRand(0, N);
      int j = intRand(0, N);
      int sum{};
      {
	if (i % 3 == 0) {
	  std::lock_guard<std::mutex> lgd(tmut[(i % N)/3]);
          sum = nums[i] + nums[(i+1) % N] + nums[(i+2) % N];
	} else if (((i+1) % N) / 3 == i / 3) {
	    std::lock_guard<std::mutex> lgd(tmut[(i % N)/3]);
	    std::lock_guard<std::mutex> lgd(tmut[((i+2) % N)/3]);
	    sum = nums[i] + nums[(i+1) % N] + nums[(i+2) % N];
	}
        
	std::lock_guard<std::mutex> lgd(tmut[j % N]);
	nums[j] = sum;
      }
      
    }
    
    return;
  };
```

**Note**

上述加锁策略依然不会产生死锁

# 方案五

将上述std::mutex改成读写锁，也即在C++中的shared_timed_mutex（C++14），shared_mutex(C++17)

当读元素的时候，通过如下形式加锁

```c++
std::shared_mutex mut;
std::shared_lock lgd(mut);
```

当更新元素的时候，通过如下形式加锁

```c++
std::shared_mutex mut;
std::unique_lock lgd(mut);
```

# 方案六

通过原子变量来进行相应的计算，也即将数组中的数存储为atomic_int类型。

其核心代码如下

```c++
  std::vector<atomic_int> S(N);
  std::iota(S.begin(), S.end(), 0);

  auto thread_func = [](std::mutex& tmut, std::vector<int>& nums) {
    for (int i = 0; i < 10000; ++i) {
      int i = intRand(0, N);
      int j = intRand(0, N);

      int num1 = S[(i%N)].load();
      int num2 = S[(i+1) % N].load();
      int num3 = S[(i+2) % N].load();
      int expected = S[j % N].load();
      while (!S[j % N].compare_exchange_weak(expected, num1 + num2 + num3));
    }
    
    return;
  };
```

**Note**

该方案属于最基本的无锁编程，故不会存在死锁。


# 参考

[How do I generate thread-safe uniform random numbers?](https://stackoverflow.com/questions/21237905/how-do-i-generate-thread-safe-uniform-random-numbers)

[Two Phase Locking Protocol](https://www.scaler.com/topics/two-phase-locking-protocol/)
