# [1] NavigationView嵌套崩溃问题

### 问题描述

Window内有一个NavigationView控件，在这个控件的某个Page中，包含另一个NavigationView控件。这个控件里包含几个NavigationViewItem。

这时关闭Window后会崩溃。因为我使用自己的Release版本的库导致我只能在Release下调试，得到的信息有限。根据最后的堆栈调用信息，可以看到应该是在NagivationView的析构函数中崩溃的，崩溃位置应该是在释放NavigationViewItem时。根据反汇编可以粗略认为是hresult_error，发生在WinRT运行时类的自减引用，应该是引用数为0时又发生了自减引用。

但我不知道为什么会发生这个，在只有一个NavigationView的程序中没有任何问题，当有嵌套NavigationView时理论上应该也是正常，我保证我写的代码没有任何问题，且代码已经删减到足够短，看不出任何问题了。

### 解决过程

 - 我尝试删去内部NavigationView的所有NavigationViewItem，编译运行正常。
 - 我尝试在C++代码中手动创建NavigationViewItem并添加，编译运行崩溃。
 - 我尝试将Page的NavigationCacheMode删去Required，编译运行崩溃。
 - 我尝试删去所有的OnNavigationTo及OnSelected之类的方法，只剩一个NavigationView及Item，编译运行崩溃。
 - 当我打开Window后最顶层的NavigationView不切换到那个有NavigationView的Page，编译运行正常。当我切换到这个Page，关闭Window，运行崩溃。但我切换到这个Page再切换到别的Page，关闭Window，运行崩溃。

 ### 最终答案

很离谱，我给嵌套内层的NavigationView添加一个名称x:Name时程序就能正常运行了？？？？？？？？？

添加一个x:Name会让运行时类的Page拥有一个对它的引用所有权吗，让引用次数加1？这也不会影响什么啊？如果没有嵌套的NavigationView，就一个单独的NavigationView不加x:Name也能正常运行。

我仍然在等待新的答案！！！！

### 浪费时间： 4h