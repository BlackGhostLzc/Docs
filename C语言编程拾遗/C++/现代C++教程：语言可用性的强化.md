## 常量

### nullptr

在 C++ 中，`nullptr` 和 `NULL` 都用于表示空指针，但它们在类型安全、适用场景和设计理念上有显著区别。

`nullptr` 出现的目的是为了替代 `NULL`。 C 与 C++ 语言中有**空指针常量**，它们能被隐式转换成任何指针类型的空指针值，或 C++ 中的任何成员指针类型的空成员指针值。 `NULL` 由标准库实现提供，并被定义为实现定义的空指针常量。

**类型安全问题**

在 C++ 中，`NULL` 通常定义为0或者((void*)0)，这种定义会导致：

- 整数和指针的歧义。
- 模板推导错误。

```c
void func(int) { std::cout << "int\n"; }
void func(int*) { std::cout << "int*\n"; }

int main() {
    func(NULL);    // 可能调用 func(int)（不符合预期）
    func(nullptr); // 明确调用 func(int*)
}
```

- `nullptr` 是类型安全的“空指针常量”，其类型 `std::nullptr_t` 可隐式转换为任何指针类型，但不会转换为整数。

  ```c
  int* p1 = nullptr; // 合法
  int n = nullptr;   // 编译错误！
  ```

- `NULL` 仅是预处理器替换的文本，缺乏类型信息。



## constexpr

