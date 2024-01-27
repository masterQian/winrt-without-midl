# Defining C++/WinRT runtime classes without MIDL


### I will use a header only code from my library to illustrate how to simply define a Windows runtime class without MIDL.

我将使用我写的库中一部分header only的代码来讲述如何定义一个Windows运行时类，而不使用MIDL。

---

- **MasterQian.WinRT.Args.h**
``` c++
#pragma once
#include "winrt/windows.foundation.h"

namespace MasterQian::WinRT {
    struct Args;
    namespace details {
        struct ArgsImpl;

        template<typename T>
        concept arg_pointer = std::is_pointer_v<T> || std::same_as<T, std::nullptr_t>;
    }
}

namespace winrt::impl {
    template <>
    inline constexpr guid guid_v<MasterQian::WinRT::Args>{ 0x2ED4939F, 0xD013, 0x56FD, { 0xAC, 0xB4, 0x87, 0xF2, 0xB1, 0x3A, 0x06, 0xDC } };

    template <>
    struct abi<MasterQian::WinRT::Args> {
        struct __declspec(novtable) type : inspectable_abi {
            virtual int32_t __stdcall Size(uint32_t*) noexcept = 0;
            virtual int32_t __stdcall Append(Windows::Foundation::IInspectable const&) noexcept = 0;
            virtual int32_t __stdcall GetAt(uint32_t, Windows::Foundation::IInspectable*) noexcept = 0;
            virtual int32_t __stdcall GetAll(Windows::Foundation::IInspectable**) noexcept = 0;
        };
    };

    template <typename D>
    struct produce<D, MasterQian::WinRT::Args> : produce_base<D, MasterQian::WinRT::Args> {
        int32_t __stdcall Size(uint32_t* result) noexcept final try {
            MasterQian::WinRT::details::ArgsImpl& args{ this->shim() };
            *result = static_cast<uint32_t>(args.objects.size());
            return 0;
        }
        catch (...) { return to_hresult(); }

        int32_t __stdcall Append(Windows::Foundation::IInspectable const& obj) noexcept final try {
            MasterQian::WinRT::details::ArgsImpl& args{ this->shim() };
            args.objects.emplace_back(obj);
            return 0;
        }
        catch (...) { return to_hresult(); }

        int32_t __stdcall GetAt(uint32_t index, Windows::Foundation::IInspectable* result) noexcept final try {
            MasterQian::WinRT::details::ArgsImpl& args{ this->shim() };
            typename D::abi_guard guard(args);
            *result = args.objects[index];
            return 0;
        }
        catch (...) { return to_hresult(); }

        int32_t __stdcall GetAll(Windows::Foundation::IInspectable** result) noexcept final try {
            MasterQian::WinRT::details::ArgsImpl& args{ this->shim() };
            typename D::abi_guard guard(args);
            *result = args.objects.data();
            return 0;
        }
        catch (...) { return to_hresult(); }
    };
}

namespace MasterQian::WinRT {
    struct Args : winrt::Windows::Foundation::IInspectable {
    private:
        using this_abi = winrt::impl::abi_t<Args>;

        static constexpr struct _placeholder_t {} _placeholder;
        template<typename... T>
        using replace_placeholder_tuple = std::tuple<std::conditional_t<std::same_as<T, void>, _placeholder_t, T>...>;

        template<typename U>
        static winrt::Windows::Foundation::IInspectable box_arg(U&& u) {
            if constexpr (details::arg_pointer<U>) {
                return winrt::box_value(reinterpret_cast<uint64_t>(u));
            }
            else {
                return winrt::box_value(std::forward<U>(u));
            }
        }

        template<typename U>
        static U unbox_arg(winrt::Windows::Foundation::IInspectable const& value) {
            if constexpr (std::same_as<U, _placeholder_t>) {
                return _placeholder;
            }
            else if constexpr (details::arg_pointer<U>) {
                return reinterpret_cast<U>(winrt::unbox_value<uint64_t>(value));
            }
            else {
                return winrt::unbox_value<U>(value);
            }
        }

        template<typename D, size_t... Index>
        D unbox_args(std::index_sequence<Index...>) {
            winrt::Windows::Foundation::IInspectable* data{ };
            winrt::check_hresult((*(this_abi**)this)->GetAll(&data));
            return std::make_tuple(Args::unbox_arg<std::tuple_element_t<Index, D>>(data[Index])...);
        }
    public:
        Args(std::nullptr_t) {}
        Args(void* ptr, winrt::take_ownership_from_abi_t) noexcept : winrt::Windows::Foundation::IInspectable{ ptr, winrt::take_ownership_from_abi } {}
        Args();

        [[nodiscard]] uint32_t size() const {
            uint32_t result{ };
            winrt::check_hresult((*(this_abi**)this)->Size(&result));
            return result;
        }

        [[nodiscard]] winrt::Windows::Foundation::IInspectable operator [] (uint32_t index) const {
            winrt::Windows::Foundation::IInspectable result;
            winrt::check_hresult((*(this_abi**)this)->GetAt(index, &result));
            return result;
        }

        template<typename... T>
        [[nodiscard]] static Args box(T&&... t) {
            Args args;
            auto impl{ *(this_abi**)&args };
            (winrt::check_hresult(impl->Append(Args::box_arg(std::forward<T>(t)))), ...);
            return args;
        }
        
        template<typename... T>
        [[nodiscard]] replace_placeholder_tuple<T...> unbox() {
            return unbox_args<replace_placeholder_tuple<T...>>(std::make_index_sequence<sizeof...(T)>{});
        }
    };

    namespace details {
        struct ArgsImpl : winrt::implements<ArgsImpl, Args> {
            std::vector<winrt::Windows::Foundation::IInspectable> objects;

            winrt::hstring GetRuntimeClassName() const { return L"MasterQian.WinRT.Args"; }
        };
    }

    Args::Args() : Args{ winrt::make<details::ArgsImpl>() } {}
}
```

- **main.cpp**
``` c++
#include "MasterQian.WinRT.Args.h"
int main() {
    winrt::init_apartment();
    /*
        Boxing any number of parameters of different types
        as a parameter for event function calls or Tag as a control.
        将任意数量的不同类型的值装箱到一个运行时类型的参数包中
        可以作为事件函数的参数或控件的Tag进行传递
    */
    auto args = MasterQian::WinRT::Args::box(2, 3.5, L"123", winrt::Windows::UI::Xaml::Controls::Button{});
   
    // like frame.Navigate(xaml_type<MainPage>(), args);
    // or button.Tag(args);

    /*
    Unboxing from a single parameter
    For parameters that are not of concern, void can be used to ignore them, and as long as the quantity does not exceed the length of the original parameter package, it does not necessarily have to be the same.
    The types for r1 and r3 will be int and hstring
    从单个参数包进行拆箱
    若不关心某个参数，可以使用void来忽略它，且数量并不一定要相同，只需要不超过原参数包长度。
    r1和r3的类型将是int和hstring
    */
    auto [r1, _, r3] = args.unbox<int, void, winrt::hstring>();
}
```

Next, I will introduce step by step how to define Args as a runtime class without using MIDL.

下面我将来一步一步介绍如何不使用MIDL来定义Args这个运行时类。

- ### [ 1 ]

``` c++
namespace MasterQian::WinRT {
    struct Args;
    namespace details {
        struct ArgsImpl;
    }
}
```

I have declared in advance the runtime class ***Args*** that I want to define and its implementation ***details::ArgsImpl***, because of the later implementations rely on the Curiously Recurring Template Pattern (CRTP).

这里提前声明了我要定义的运行时类Args以及其实现details::ArgsImpl，因为后面一些实现依赖奇异递归模板（CRTP）。

- ### [ 2 ]

``` c++
namespace winrt::impl {
    template <>
    inline constexpr guid guid_v<MasterQian::WinRT::Args>{ 0x2ED4939F, 0xD013, 0x56FD, { 0xAC, 0xB4, 0x87, 0xF2, 0xB1, 0x3A, 0x06, 0xDC } };
}
```

Then a ***GUID*** was defined, and you can use the GUId generator to generate or write any one, which has a lower probability of repetition than the possibility of a cosmic explosion. Note that it and some subsequent content are in the ***winrt::impl*** namespace.

然后定义了一个GUID，你可以使用GUID生成器来生成或随便写一个，它重复的概率比宇宙爆炸的可能性还小。注意它及后面一些内容是在winrt::impl命名空间中的。


- ### [ 3 ]

``` c++
namespace winrt::impl {
    template <>
    struct abi<MasterQian::WinRT::Args> {
        struct __declspec(novtable) type : inspectable_abi {
            virtual int32_t __stdcall Size(uint32_t*) noexcept = 0;
        };
    };
}

```

This defines a binary interface that uses the custom type ***Args*** to specialize the ***abi*** template and specifies that its internal ***type*** inherits from ***inspectable_abi*** interface, and then define some virtual functions as interfaces for runtime classes.

这里定义了二进制接口，使用自定义类型Args来对abi模板特化，并指定其内部的type继承自inspectable_abi接口，然后定义一些虚函数，作为运行时类的接口。

The form of virtual functions is to return ***int32_t***. Represents ***hresult*** and identifies ***noexcept***, while the true return type is passed through parameters using pointers.

虚函数的形式都是返回int32_t，代表着hresult，并标识noexcept，而真正的返回类型使用指针通过参数传递。

- ### [ 4 ]

``` c++
namespace winrt::impl {
    template <typename D>
    struct produce<D, MasterQian::WinRT::Args> : produce_base<D, MasterQian::WinRT::Args> {
        int32_t __stdcall Size(uint32_t* result) noexcept final try {
            MasterQian::WinRT::details::ArgsImpl& args{ this->shim() };
            *result = static_cast<uint32_t>(args.objects.size());
            return 0;
        }
        catch (...) { return to_hresult(); }
    };
}
```

Here we define that the ***product*** class inherits from the ***product_base***, and use ***Args*** specialization. Note that ***D*** is actually ***ArgsImpl***, but we do not specialize. This is to facilitate the implementation of types until after template instantiation, otherwise undefined types may occur.

这里我们定义produce类继承自produce_base，且使用Args特化，注意D其实就是ArgsImpl，但我们并不特化，这是为了方便使用实现类型延迟到模板实例化后，否则会出现未定义类型的问题。

We implement the interface in ***abi*** here, using reference acceptance ***this->shim()*** to obtain its implementation type. You can also use ***abi_guard*** like the ***winrt*** source code to protect its lifecycle. By performing a series of operations on the implementation type, the return value is passed through the parameter ***result***. And return 0 as ***hresult***. If an exception occurs, it will be caught by ***try-catch*** and processed by ***winrt***.

我们在这里实现abi中的接口，使用引用接受this->shim()来获得其实现类型，你也可以像winrt源代码一样使用abi_guard来保护其生命周期。通过对实现类型进行一系列操作，将返回值通过参数result传递出去。并返回0作为hresult，如果发生异常将会被try-catch捕捉后由winrt处理。

- ### [ 5 ]

``` c++
namespace MasterQian::WinRT {
    struct Args : winrt::Windows::Foundation::IInspectable {
    private:
        using this_abi = winrt::impl::abi_t<Args>;
    
    public:
        Args(std::nullptr_t) {}
        Args(void* ptr, winrt::take_ownership_from_abi_t) noexcept : winrt::Windows::Foundation::IInspectable{ ptr, winrt::take_ownership_from_abi } {}
        Args();

        [[nodiscard]] uint32_t size() const {
            uint32_t result{ };
            winrt::check_hresult((*(this_abi**)this)->Size(&result));
            return result;
        }
    }
}
```

Then we define the runtime type ***Args***, which inherits from ***winrt::Windows::Foundation::IInspectable***, its abi type is ***winrt::impl::abi_t&lt;Args&gt;***.The first two constructors are used internally, creating a ***null*** object or for internal factory calls. The third one is our own defined constructor, which is a default constructor without parameters.

然后我们定义运行时类型Args，继承自winrt::Windows::Foundation::IInspectable。其abi类型是winrt::impl::abi_t&lt;Args&gt;。前两个构造函数都用于内部，创建一个null的对象或用于内部工厂调用，第三个是我们自己定义的构造函数，是无参的默认构造函数。

Then we implement the ***size()*** method, which calls the ***Size()*** method of the ***abi*** interface and ***check_hresult()*** is used to determine if an exception has occurred and convert the result returned by the pointer into a friendly form of return value.

然后我们实现size()方法，使用*(this_abi**)this来获得abi接口，然后调用Size()方法，并通过check_hresult()来判断是否发生异常，并将指针返回的结果转化成友好形式的返回值。


- ### [6]

```c++
namespace MasterQian::WinRT {
    namespace details {
        struct ArgsImpl : winrt::implements<ArgsImpl, Args> {
            std::vector<winrt::Windows::Foundation::IInspectable> objects;

            winrt::hstring GetRuntimeClassName() const { return L"MasterQian.WinRT.Args"; }
        };
    }
}
```

Here we define the implementation type ***ArgsImpl***, which is internally a ***std::vector***. We only need to implement a ***GetRuntimeClassName()*** interface, which informs us of the name of the runtime type. For example, you can use ***winrt::get_class_name*** to obtain the type name of a runtime type instance.

这里我们定义实现类型ArgsImpl，其内部是一个std::vector，我们只需要实现一个GetRuntimeClassName()接口即可，它告知了运行时类型的名称。例如可以使用winrt::get_class_name来获得一个运行时类型实例的类型名称。

- ### [7]

```c++
namespace MasterQian::WinRT {
     Args::Args() : Args{ winrt::make<details::ArgsImpl>() } {}
}
```

Don't forget, we only declared the unarmed constructor of ***Args*** earlier, as it requires a specific implementation type to construct it, and we can only delay defining it.

别忘了，前面我们只声明了Args的无参构造函数，因为这要使用具体的实现类型去构造它，我们只能延后定义。

---

### Afterwards, ***Args*** can be used just like other runtime types. We did not use ***MIDL*** files and did not generate corresponding definitions for *.h, *.cpp, and a series of factory patterns. We only used header only.

### 之后便可以像使用其他运行时类型一样使用Args了，我们没有使用MIDL文件且不会生成对应的*.h, *.cpp以及一系列工厂模式的定义，我们仅仅是header only的。 