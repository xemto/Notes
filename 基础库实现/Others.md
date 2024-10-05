# 字符和字符串
C语言只规定了`unsigned char` 是无符号8位整数，`signed char` 是有符号8位整数，而`char` 类型只需要是8位整数即可。
以GCC为例，他规定`char` 在X86架构是有符号的(`char = signed char`)，而在arm架构上则认为是无符号的(`char = unsigned char`)。
顺便一提，C++标准保证`char`，`signed char`，`unsigned char` 是三个完全不同的类型，`std::is_same_v` 分别判断他们总会得到false，无论X86还是arm。
```cpp
if(std::is_signed<char>::value) {
	printf("char is signed, x86");
} else {
	printf("char is unsigned, arm");
}
```

- `'h'` 是一个语法糖，等价于`h` 的ASCII码---整数104
- `"hello"`也是一个语法糖，等价于数组`{'h', 'e', 'l', 'l', 'o', 0}`，注意：`'\0'` 等价于`0` ，但是不等价于`'0'` 
 ## C++ string
```cpp
const char *cs;
{
	string s = "hello world";
	cs = s.c_str();
	// 堆上的数据会被释放
}
// 当 s.len() > 15 时，打印乱码
cout << cs << endl;
```
 `string` 有小字符串优化，当字符串长度小于15字节时，数据存储在栈上。大于15字节时，数据存储在堆上。
 
