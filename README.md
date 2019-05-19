# learn-pprof-by-example-for-c

## 内容

- [原理](#原理)
- [依赖](#依赖)
- [例程](#例程)
- [运行](#运行)

## 原理

## 依赖

libunwind:
  
    下载http://download.savannah.nongnu.org/releases/libunwind/libunwind-xxx.tar.gz
    tar -zxvf libunwind-xxx.tar.gz
    . /configure --prefix=`pwd`
    make;make install

gperftools:

    wget https://github.com/gperftools/gperftools/archive/gperftools-xx.tar.gz
    tar -zxvf gperftools-xx.tar.gz
    ./configure --prefix=`pwd` LDFLAGS=-L/xxxx/libunwind/lib CPPFLAGS=-I/xxxx/libunwind/include

## 例程


```c
#include <gperftools/profiler.h>
#include <gperftools/heap-profiler.h>
 
#include <stdlib.h>
#include <stdint.h>
 
typedef struct testbuffer{
    int iValue;
} testbuffer;
 

testbuffer* createbuffer()
{
    testbuffer* p = (testbuffer*)(malloc(sizeof(testbuffer) * 100000000));
    return p;
}

void test()
{
    for(uint64_t i = 0; i<10000000000; i++ ){

    }
}


int main()
{
   ProfilerStart("./cpu_profile.prof");
   HeapProfilerStart("./mem_profile.prof");

   testbuffer* p = createbuffer();
   testbuffer* p1 = createbuffer();
   HeapProfilerDump("createbuffer");

   (void)(p);
   (void)(p1);

   free(p);
   p = NULL;

   HeapProfilerDump("afterfree");

   free(p1);
   p1 = NULL;

   HeapProfilerDump("afterfree1");
   HeapProfilerStop();
   
   test();
   ProfilerStop();
}
```

CMakeLists.txt 如下:

```cmake
CMAKE_MINIMUM_REQUIRED(VERSION 3.5)
PROJECT(learn_pprof)

SET(CMAKE_BUILD_TYPE "debug")
SET(C_FLAGS
    -g                  # for gdb
    -Wall
    -Wextra
    -Wno-unused-parameter
    -Wno-switch-enum
    -rdynamic           # for backtrace
    -Wconversion
    -Wpointer-arith
    -march=native
    -Werror
)

STRING(REPLACE ";" " " CMAKE_C_FLAGS "${C_FLAGS}")
SET(CMAKE_C_COMPILER "g++")
SET(CMAKE_C_FLAGS_DEBUG "-o0")
SET(CMAKE_C_FLAGS_RELEASE "-o2 -finline-limit=1000 -DNDEBUG")
SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})
INCLUDE_DIRECTORIES(/xxxx/gperftools-gperftools-2.7/include)
INCLUDE_DIRECTORIES(/xxxx/libunwind-2.3.1/include)

ADD_EXECUTABLE(learn_pprof learn_pprof.c)
TARGET_LINK_LIBRARIES(learn_pprof /xxxx/gperftools-gperftools-2.7/lib/libprofiler.a)
TARGET_LINK_LIBRARIES(learn_pprof /xxxx/gperftools-gperftools-2.7/lib/libtcmalloc.a)
TARGET_LINK_LIBRARIES(learn_pprof pthread)
TARGET_LINK_LIBRARIES(learn_pprof /xxxx/libunwind-1.3.1/lib/libunwind.a)
```


## 运行

    $ env HEAP_PROFILE_INUSE_INTERVAL=1048576000 ./learn_pprof 
    Starting tracking the heap
    Dumping heap profile to ./mem_profile.prof.0001.heap (createbuffer)
    Dumping heap profile to ./mem_profile.prof.0002.heap (afterfree)
    Dumping heap profile to ./mem_profile.prof.0003.heap (afterfree1)
    PROFILE: interrupts/evictions/bytes = 1402/0/160


打开cpu_profile.prof 进行函数优化:

```
$ pprof learn_pprof cpu_profile.prof 
Using local file learn_pprof.
Using local file cpu_profile.prof.
Removing main from all stack traces.
Removing __libc_start_main from all stack traces.
Removing _start from all stack traces.
Welcome to pprof!  For help, type 'help'.
(pprof) top
Total: 1402 samples
    1402 100.0% 100.0%     1402 100.0% test
(pprof) list test
Total: 1402 samples
ROUTINE ====================== test in /home/dan/work/37prof/learn_pprof.c
  1402   1402 Total samples (flat / cumulative)
     .      .   14:     testbuffer* p = (testbuffer*)(malloc(sizeof(testbuffer) * 100000000));
     .      .   15:     return p;
     .      .   16: }
     .      .   17: 
     .      .   18: void test()
---
     .      .   19: {
  1402   1402   20:     for(uint64_t i = 0; i<10000000000; i++ ){
     .      .   21:         
     .      .   22:     }
     .      .   23: }
---
     .      .   24: 
     .      .   25: 
     .      .   26: int main()
     .      .   27: {
     .      .   28:     ProfilerStart("./cpu_profile.prof");
```

导出callgrind，然后用KCachegrind打开进行可视化，例如:

```
$ pprof learn_pprof cpu_profile.prof 
Using local file learn_pprof.
Using local file cpu_profile.prof.
Removing main from all stack traces.
Removing __libc_start_main from all stack traces.
Removing _start from all stack traces.
Welcome to pprof!  For help, type 'help'.
(pprof) callgrind ./callgrind.out.7
Writing callgrind file to './callgrind.out.7'.
```


打开进行内存检测，例如:

```
$ pprof learn_pprof mem_profile.prof.0001.heap
Using local file learn_pprof.
Using local file mem_profile.prof.0001.heap.
Welcome to pprof!  For help, type 'help'.
(pprof) top
Total: 762.9 MB
   762.9 100.0% 100.0%    762.9 100.0% createbuffer
     0.0   0.0% 100.0%    762.9 100.0% __libc_start_main
     0.0   0.0% 100.0%    762.9 100.0% _start
     0.0   0.0% 100.0%    762.9 100.0% main
(pprof) list main
Total: 762.9 MB
ROUTINE ====================== main in /home/dan/work/37prof/learn_pprof.c
   0.0  762.9 Total MB (flat / cumulative)
     .      .   22:     }
     .      .   23: }
     .      .   24: 
     .      .   25: 
     .      .   26: int main()
---
     .      .   27: {
     .      .   28:     ProfilerStart("./cpu_profile.prof");
     .      .   29:     HeapProfilerStart("./mem_profile.prof");
     .      .   30: 
     .  381.5   31:     testbuffer* p = createbuffer();
     .  381.5   32:     testbuffer* p1 = createbuffer();
     .      .   33:     HeapProfilerDump("createbuffer");
     .      .   34:     
     .      .   35:     (void)(p);
     .      .   36:     (void)(p1);
     .      .   37: 
     .      .   38:     free(p);
     .      .   39:     p = NULL;
     .      .   40: 
     .      .   41:     HeapProfilerDump("afterfree");
     .      .   42:  
     .      .   43:     free(p1);
     .      .   44:     p1 = NULL;
     .      .   45: 
     .      .   46:     HeapProfilerDump("afterfree1");
     .      .   47:     HeapProfilerStop();
     .      .   48:     
     .      .   49:     test(); 
     .      .   50:     ProfilerStop();
     .      .   51: }
     
$ pprof learn_pprof mem_profile.prof.0002.heap
Using local file learn_pprof.
Using local file mem_profile.prof.0002.heap.
Welcome to pprof!  For help, type 'help'.
(pprof) top
Total: 381.5 MB
   381.5 100.0% 100.0%    381.5 100.0% createbuffer
     0.0   0.0% 100.0%    381.5 100.0% __libc_start_main
     0.0   0.0% 100.0%    381.5 100.0% _start
     0.0   0.0% 100.0%    381.5 100.0% main
(pprof) list main
Total: 381.5 MB
ROUTINE ====================== main in /home/dan/work/37prof/learn_pprof.c
   0.0  381.5 Total MB (flat / cumulative)
     .      .   22:     }
     .      .   23: }
     .      .   24: 
     .      .   25: 
     .      .   26: int main()
---
     .      .   27: {
     .      .   28:     ProfilerStart("./cpu_profile.prof");
     .      .   29:     HeapProfilerStart("./mem_profile.prof");
     .      .   30: 
     .      .   31:     testbuffer* p = createbuffer();
     .  381.5   32:     testbuffer* p1 = createbuffer();
     .      .   33:     HeapProfilerDump("createbuffer");
     .      .   34:     
     .      .   35:     (void)(p);
     .      .   36:     (void)(p1);
     .      .   37: 
     .      .   38:     free(p);
     .      .   39:     p = NULL;
     .      .   40: 
     .      .   41:     HeapProfilerDump("afterfree");
     .      .   42:  
     .      .   43:     free(p1);
     .      .   44:     p1 = NULL;
     .      .   45: 
     .      .   46:     HeapProfilerDump("afterfree1");
     .      .   47:     HeapProfilerStop();
     .      .   48:     
     .      .   49:     test(); 
     .      .   50:     ProfilerStop();
     .      .   51: }
     
     
$ pprof learn_pprof mem_profile.prof.0003.heap
Using local file learn_pprof.
Using local file mem_profile.prof.0003.heap.
Welcome to pprof!  For help, type 'help'.
(pprof) top
Total: 0.0 MB
(pprof) list main
Total: 0.0 MB
ROUTINE ====================== main in /home/dan/work/37prof/learn_pprof.c
   0.0    0.0 Total MB (flat / cumulative)
     .      .   22:     }
     .      .   23: }
     .      .   24: 
     .      .   25: 
     .      .   26: int main()
---
     .      .   27: {
     .      .   28:     ProfilerStart("./cpu_profile.prof");
     .      .   29:     HeapProfilerStart("./mem_profile.prof");
     .      .   30: 
     .      .   31:     testbuffer* p = createbuffer();
     .      .   32:     testbuffer* p1 = createbuffer();
     .      .   33:     HeapProfilerDump("createbuffer");
     .      .   34:     
     .      .   35:     (void)(p);
     .      .   36:     (void)(p1);
     .      .   37: 
     .      .   38:     free(p);
     .      .   39:     p = NULL;
     .      .   40: 
     .      .   41:     HeapProfilerDump("afterfree");
     .      .   42:  
     .      .   43:     free(p1);
     .      .   44:     p1 = NULL;
     .      .   45: 
     .      .   46:     HeapProfilerDump("afterfree1");
     .      .   47:     HeapProfilerStop();
     .      .   48:     
     .      .   49:     test(); 
     .      .   50:     ProfilerStop();
     .      .   51: }    
     
     
```
