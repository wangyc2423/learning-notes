# 为啥ffi大量用Cython而不是Ctypes

1. 当有指针成员的时候，python的GC会回收无用的指针，导致c访问到野指针

2. python和c都不知道何时malloc或者free

3. 结构体中的对齐要求必须手动指定

4. 不支持c++对象

5. 无GIL控制（借助GIL控制可以实现多线程执行非python代码）
