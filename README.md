# learn-pprof-by-example-for-c

## 内容

- [原理](#原理)
- [依赖](#依赖)
- [例程](#例程)

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
