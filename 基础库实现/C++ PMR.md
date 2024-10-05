C++没有`malloc` 函数，但是有`allocator`：
```cpp
std::allocator<char> alloc;
char *p = alloc.allocate(4);
alloc.deallocate(p, 4);
```
c语言的`malloc`函数，或者c++的`new` 关键字创造出来的内存空间会在前面藏一个信息控制头，包含了分配的内存空间大小，所以在`free` 或`delete` 时不需要传入长度信息。
为了实现定制化，可以自行实现一个`allocator`：
```cpp
static char buf[65535 * 10]

struct my_custom_allocator {
	char *allocate(size_t n) {
		return buf;
	}

	void deallocate(char *p, size_t n) {
		// do nothing
	}
}

my_custom_allocator alloc;
char *p = alloc.allocate(4);
alloc.deallocate(p, 4);
```
为了让自定义的`allocator` 支持容器，继续修改：
```cpp
// 在堆上
static char buf[65535 * 10];
static size_t watermark = 0;

template <class T>
struct my_custom_allocator {
	using value_type = T;
	
	char *allocate(size_t n) {
		char *p = buf + watermark;
		watermark += (n * sizeof(T));
		return (T*)p;
	}

	void deallocate(char *p, size_t n) {
		// do nothing
	}

	my_custom_allocator() = default;

	template <class U>
	constexpr my_custom_allocator(my_custom_allocator<U> const *other) noexcept {
	}

	constexpr bool operator==(my_custom_allocator const &other) noexcept {
		return this == &other;
	}
}

std::list<char, my_custom_allocator<char>> a;
a.push_back(42);
```
为了效率，甚至可以将缓存放到堆上：
```cpp
struct mem_resource {
	char m_buf[65535 * 10];
	size_t m_watermark = 0;
	char* do_allocate(size_t n, size_t align) {
		m_watermark = (m_watermark + align - 1) / align * align;
		char *p = m_buf + m_watermark;
		m_watermark += n;
		return p;
	}
};

template <class T>
struct my_custom_allocator {
	mem_resource *m_resource;
	
	using value_type = T;

	my_custom_allocator(mem_resource *resource) :m_resource(resource) {}
	
	char *allocate(size_t n) {
		char *p = m_resource->do_allocate(n*sizeof(T), alignof(T));
		return (T*)p;
	}

	void deallocate(char *p, size_t n) {
		// do nothing
	}

	my_custom_allocator() = default;

	template <class U>
	constexpr my_custom_allocator(my_custom_allocator<U> const *other) noexcept {
	}

	constexpr bool operator==(my_custom_allocator const &other) noexcept {
		return this == &other;
	}
}

int main()
{
	mem_resource mem;
	std::vector<char, my_custom_allocator<char>> a(42, my_custom_allocator<char>(&mem));
	std::vector<char, my_custom_allocator<char>> b{my_custom_allocator<char>{&mem}};
}
```
在标准库中是这么实现的：
```cpp
static char g_buf[65535*30];

struct memory_resource {
	virtual char *do_allocate(size_t n, size_t align) = 0;
}

struct my_memory_resource : memory_resource {
	size_t m_watermark = 0;
	char *m_buf = g_buf;
	char *do_allocate(size_t n, size_t align) override {
		// do something ...
	}
}
```
## 使用pmr
```cpp
std::pmr::synchronized_pool_resource pool;
int main() {
	std::pmr::monotonic_buffer_resource mem{65536*24, &pool}; // dervied from std::pmr::memory_resource
	std::pmr::list<char, std::pmr::polymorphic_allocator<char>> a{std::polymorphic_allocator<char>{&mem}};

	std::pmr::monotonic_buffer_resource mem{g_buf, sizeof(g_buf), &pool};
}
```

其他的pmr：
- `std::pmr::synchronized_pool_resource`
- `std::pmr::unsynchronized_pool_resource`
- `std::pmr::monotonic_buffer_resource`
```cpp
// 默认值
std::pmr::list<char> a{std::pmr::new_delete_resource()};

// or set
std::pmr::monotonic_buffer_resource mem(g_buf, sizeof(g_buf), &pool);
std::pmr::set_default_resource(&mem);
std::pmr::list<char> b;

// null
std::pmr::monotonic_buffer_resource mem(g_buf, sizeof(g_buf), std::pmr::null_memory_resource());
```
效率：
wendous malloc < sync < glibc malloc < unsync < monot
## 跟踪容器内存分配
```cpp
struct memory_resource_inspector : std::pmr::memory_resource {
public:
	explicit memory_resource_inspector(std::pmr::memory_resource *upstream) : m_upstream(upstream) {}

private:
	void *do_allocate(size_t bytes, size_t alignment) override {
		void *p = m_upstream->allocate(bytes, alignment);
		fmt::print("* allocate {} {} {}\n", p, bytes, alignment);
		return p;
	}

	bool do_is_equal(std::pmr::memory_resource const &other) const noexcept override {
		return other.is_equal(*m_upstream);
	}

	void do_deallocate(void *p, size_t bytes, size_t alignment) override {
		fmt::print("deallocate {} {} {}\n", p, bytes, alignment);
		return m_upstream->deallocate(p, bytes, alignment);
	}

	std::pmr::memory_resource *m_upstream;
};

memory_resource_inspector mem{std:::pmr::new_delete_resource()};

std::pmr::list<int> l{&mem};
```