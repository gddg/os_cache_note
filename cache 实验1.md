# cache 实验1

测试代码的locality。
数组的读区方式不同，按照行读，被**cache**也是按行连续加载的。
如果按照列读区，那么效率很低，除非cache足够大，而且也要遍历所有的数据，并且cache hash算法也好，实现的硬件还是多路组相联的cache硬件实现。

** valgrind --tool=cachegrind ./test2**


##code1: 
```c++
#include <stdio.h>
#define MAXROW 8000
#define MAXCOL 8000
int main () {
int i,j;
 static int x[MAXROW][MAXCOL];
 printf ("Starting!\n");
       for (i=0;i<MAXROW;i++)
       for (j=0;j<MAXCOL;j++)
              x[i][j] = i*j;
			 printf("Completed!\n");
return 0;                                                    
 }
```

##code2:

```c++
#include <stdio.h>                                                         
 #define MAXROW 8000
 #define MAXCOL 8000
 int main () {
 int i,j;
 static int x[MAXROW][MAXCOL];
 printf ("Starting!\n");
          for (j=0;j<MAXCOL;j++)
                         for (i=0;i<MAXROW;i++)
                 x[i][j] = i*j;
 printf("Completed!\n");
 return 0;
 }
 ```

##结果

```
Command: ./test1
Starting!
Completed!
 
 I   refs:      905,721,688
 I1  misses:          4,177
 LLi misses:          2,808
 I1  miss rate:        0.00%
 LLi miss rate:        0.00%
 
 D   refs:      514,830,867  (386,118,735 rd   + 128,712,132 wr)
 D1  misses:      4,025,828  (     23,565 rd   +   4,002,263 wr)
 LLd misses:      4,008,456  (      6,997 rd   +   4,001,459 wr)

D1  miss rate:         0.8% (        0.0%     +         3.1%  ) 
LLd miss rate:         0.8% (        0.0%     +         3.1%  ) 

 LL refs:         4,030,005  (     27,742 rd   +   4,002,263 wr)
 LL misses:       4,011,264  (      9,805 rd   +   4,001,459 wr)
 LL miss rate:          0.3% (        0.0%     +         3.1%  )

 gcc -o test2  test2.c
** valgrind --tool=cachegrind ./test2**



I   refs:      905,720,801
I1  misses:          4,113
LLi misses:          2,811
I1  miss rate:        0.00%
LLi miss rate:        0.00%

D   refs:      514,830,348  (386,118,427 rd   + 128,711,921 wr)
D1  misses:     64,025,705  (     23,462 rd   +  64,002,243 wr)
LLd misses:      4,016,427  (      6,977 rd   +   4,009,450 wr)
**D1  miss rate:        12.4% (        0.0%     +        49.7%  )
LLd miss rate:         0.8% (        0.0%     +         3.1%  )**

LL refs:        64,029,818  (     27,575 rd   +  64,002,243 wr)
LL misses:       4,019,238  (      9,788 rd   +   4,009,450 wr)
LL miss rate:          0.3% (        0.0%     +         3.1%  )
 
Starting!
Completed!
```


##参考：
valgrind调试CPU缓存命中率和内存泄漏
<http://laoxu.blog.51cto.com/4120547/1395236>


