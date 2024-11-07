java cProject 5: Profiling an Assembly Program
Goal
In this project you will learn how to find where a program spends most of the execution time
using statistical profiling, and you will implement your own statistical profiler.
Task 0: Download the initial sources and start tsearch_asm6.s
To start your project clone the project5 repository:
git clone /homes/cs250/sourcecontrol/work/$USER/project5-src.git
cd project5-src
The implementation of binary tree search in C is similar to the one from project4. You will copy
your implementation from tsearch_asm5.s into tsearch_asm6.s
To test the implementation type
data 149 $ ./run_bench.sh
================== Running TreeSearch Iterative in C benchmark ================
Total CPU time: 4.125084397 seconds
real 0m7.960s
user 0m7.830s
sys 0m0.124s
================== Running ASM 6 benchmark ================
…
It will also try to run the tsearch_asm6.s but it will fail if it is not implemented yet.
Task 1:Insert profiling code in the benchmark
The file profil.c implements the code that starts profiling the program, and writes the histogram
of the file at the end:
void start_histogram();
void print_histogram();
Open the file profil.c and see how start_histogram creates an array of counters, that is passed
to profil(), that creates the execution histogram. See "man profil". This histogram is an array of
integers, where every integer represents an instruction or group of instruction. profil() activates a
timer that every .01secs looks at the program counter of the program, and increments the
counter in the histogram that corresponds to that program counter.
Open the file tsearch_bench_better.c and find the main(). Then above main, you will insert the
external prototypes of start_histogram() and print_histogram(): as follows. Also call
start_histogram() at the beginning of main() and print_histogram() at the end.
extern void start_histogram();
extern void print_histogram();
/*
* Main program function. Runs the benchmark.
*/
__attribute__ (( visibility("default") ))
int
main(int argc, char **argv)
start_histogram();
…..
print_histogram();
}
Modify run_bench, so both gcc compilation commands link profil.c
echo ================== Running TreeSearch Iterative in C benchmark ================
gcc -g -static -o tsearch_bench_iterative_c tsearch_bench_better.c tsearch.c AVLTree.c tsearch_iterative.c profil.c || exit 1
…
gcc -g -static -o tsearch_bench_asm6 tsearch_bench_better.c tsearch.c AVLTree.c tsearch_asm6.s profil.c || exit 1
…
Now type run_bench.
data149 $ ./run_bench.sh
You will find the following file:
ls *.hist
tsearch_bench_iterative_c.hist
Open this file, and you will observe that it contains the program counters in the histogram that
are larger than 0. The counter is multiplied by 1ms, so the counters are displayed in ms. Identify
the program counter where the program is spent most of its time:
…
0x45b86a 60ms
0x45b870 1120ms
0x45b878 10ms
0x45b87c 30ms
…
Then run the command "nm -v tsearch_bench_iterative_c | less"that prints all the functions in
the program sorted by address, and finds the function that includes this program counter. Use
the up/down arrow keys to navigate "less".
data 163 $ nm -v tsearch_bench_iterative_c | less
….
000000000045aee0 T __stpcpy_evex<代 写data、C++
代做程序编程语言br>000000000045b340 T __strchr_evex
000000000045b5e0 T __strchrnul_evex
000000000045b840 T __strcmp_evex
000000000045bcb0 T __strcpy_evex
000000000045c100 T __strlen_evex
000000000045c280 T __strncmp_evex
000000000045c7f0 T __strncpy_evex
….
__strcmp_evex is the function where tsearch_bench_iterative_c spends most of its time.
Now to find the assembly instruction type "objdump -d tsearch_bench_iterative_c | less" that
prints the assembly instructions that make the program and their address in the program. Find
the assembly instruction that includes the counter 45b870, that is 45b86c . This is because
0x45b870 is larger than 45b86c but smaller than 45b873.
objdump -d tsearch_bench_iterative_c | less
000000000045b840 :
45b840: f3 0f 1e fa endbr64
45b844: 89 f8 mov %edi,%eax
45b846: 31 d2 xor %edx,%edx
45b848: 62 a1 fd 00 ef c0 vpxorq %xmm16,%xmm16,%xmm16
45b84e: 09 f0 or %esi,%eax
45b850: 25 ff 0f 00 00 and $0xfff,%eax
45b855: 3d 80 0f 00 00 cmp $0xf80,%eax
45b85a: 0f 8f 70 03 00 00 jg 45bbd0 
45b860: 62 e1 fe 28 6f 0f vmovdqu64 (%rdi),%ymm17
45b866: 62 b2 75 20 26 d1 vptestmb %ymm17,%ymm17,%k2
45b86c: 62 f3 75 22 3f 0e 00 vpcmpeqb (%rsi),%ymm17,%k1{%k2}
45b873: c5 fb 93 c9 kmovd %k1,%ecx
45b877: ff c1 inc %ecx
45b879: 74 45 je 45b8c0 
45b87b: f3 0f bc d1 tzcnt %ecx,%edx
45b87f: 0f b6 04 17 movzbl (%rdi,%rdx,1),%eax
The instruction marked in red is the instruction that is taking the most time.
Task 2: Write your own profiler program.
Using the example in Task1, write a program myprof.c that prints a table with the top 10
functions where the program spends most of its time and it will also print for each function,
which instructions take most of the time . The program will take the following arguments:
myprof prog
The program will open prog.hist, and store the entries in an array of structs with the program
counter and the time in ms. Then it will call system("nm -v prog > nm.out") using the system()
function (see man system) that executes a command inside a C program, and redirect it into a
file nm.out. Myprof will read nm.out, and it will also store the entries in an array of structs with
program counters and function names. Then for every pc in the histogram, it will increment the
time in ms of the corresponding function. After this is done, it will sort the functions by time, and
identify the 10 top functions where the execution spends most of the time. Finally, it will also
print the assembly code of these functions using objdump, and print the time spend in each
assembly instruction. Only the instructions with a time greater than 0 are printed.
The output will look like the following example:
myprof tsearch_bench_iterative_c
Top 10 functions:
ith Function Time(ms) (%)
1: mystrcmp 120ms 25%
2: malloc 80ms 36%
….
Top 10 functions Assembly
1: mystrcmp 120ms 25%
120ms 42cf60: f6 c2 20 test $0x20,%dl
2: malloc 80ms 36%
20ms 4628cc: 48 89 de mov %rbx,%rsi
10ms 4628cf: e8 cc e3 ff ff call 460ca0
Task 3:Using your profiler, improve your tsearch_asm6.s
Using your profiler, optimize your implementation in tsearch_asm6.s
Grading
The grading will be done during lab time. You don't need to turn in the implementation since the
git repository will have your most recent implementation.

         
加QQ：99515681  WX：codinghelp  Email: 99515681@qq.com
