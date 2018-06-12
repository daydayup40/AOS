# KLEE实验

## 实验环境搭建

- ubuntu: 14.04
- KLEE: 1.4.0.0
- LLVM: version 3.4
  官方提供了Dockerfile和现成的docker镜像,这里直接使用docker镜像

```
docker pull klee/klee
docker run -t -d klee/klee
```

## Klee解决迷宫问题

简单的迷宫C语言代码如下:

```
#define H 7
#define W 11
char maze[H][W] = { "+-+---+---+",
                    "| |     |#|",
                    "| | --+ | |",
                    "| |   | | |",
                    "| +-- | | |",
                    "|     |   |",
                    "+-----+---+" };
void draw ()
{
	int i, j;
	for (i = 0; i < H; i++)
	{
		for (j = 0; j < W; j++)
			printf ("%c", maze[i][j]);
		printf ("\n");
	}
	printf ("\n");
}
int main (int argc, char *argv[])
{
    int x, y;     //Player position
    int ox, oy;   //Old player position
    int i = 0;    //Iteration number
    #define ITERS 28
    char program[ITERS];
    x = 1;
    y = 1;
    maze[y][x]='X';
    read(0,program,ITERS);
    while(i < ITERS)
    {
        ox = x;    //Save old player position
        oy = y;
		switch (program[i])
        {
            case 'w':
                y--;
                break;
            case 's':
                y++;
                break;
            case 'a':
                x--;
                break;
            case 'd':
                x++;
                break;
            default:
                printf("Wrong command!(only w,s,a,d accepted!)\n");
                printf("You lose!\n");
                exit(-1);
        }
   		if (maze[y][x] == '#')
        {
            printf ("You win!\n");
            printf ("Your solution \n",program);
            exit (1);
        }
 		if (maze[y][x] != ' ' &&
            !((y == 2 && maze[y][x] == '|' && x > 0 && x < W)))
		{
		    x = ox;
		    y = oy;
		}
   		if (ox==x && oy==y){
            printf("You lose\n");
            exit(-2);
        }
    	maze[y][x]='X';
        draw ();          //draw it
        i++;
        sleep(1); //me wait to human
    }
	printf("You lose\n");
}
```

使用LLVM编译后,执行输入一段路径就会判断路径是否正确,给出loss或者win的结果.

```
clang -emit-llvm -c -g maze.c -o maze.bc
```

执行结果如下:

```
klee@nislv:~/lab$ lli maze.bc 
ss
+-+---+---+
|X|     |#|
|X| --+ | |
| |   | | |
| +-- | | |
|     |   |
+-----+---+

+-+---+---+
|X|     |#|
|X| --+ | |
|X|   | | |
| +-- | | |
|     |   |
+-----+---+

Wrong command!(only w,s,a,d accepted!)
You lose!
```

**下面用Klee解决这个问题,并且求出所有的解**

### 引入klee_make_symbolic

在程序代码开头引入头文件,增加一行

```
#include <klee/klee.h>
```

把目标变量program的输入改为klee_make_symbolic(program,sizeof program,"program");

自此,program变量将作为一个符号变量.

用llvm把新的程序编译成bytecode,注意需要-I参数制定klee的头文件路径

```
klee@nislv:~/lab$ clang -I /home/klee/klee_src/include/ -emit-llvm -c -g maze_klee.c -o maze_klee.bc
```

用klee执行程序:

```
klee@nislv:~/lab$ klee maze_klee.bc
```

几秒钟后可以得到如下结果:

```
---+--------+-+--+-------+-
+
+You lose




You lose
You lose

KLEE: done: total instructions = 120262
KLEE: done: completed paths = 309
KLEE: done: generated tests = 306
```

306个测试用例,覆盖309条路径
测试用例存放在klee-last文件夹中,可以使用ktest-tool来查看测试用例

```
klee@nislv:~/lab$ ktest-tool klee-last/test000143.ktest 
ktest file : 'klee-last/test000143.ktest'
args       : ['maze_klee.bc']
num objects: 1
object    0: name: b'program'
object    0: size: 28
object    0: data: b'sddsddssaaaad\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

但是要在306个测试用例里找到稀少的几个win的测试用例不太明智,可以使用assert断言报错帮助我们寻找

### 利用error快速定位测试用例

修改迷宫源代码,在

```
printf ("You win!\n");
```

下添加一行

```
printf ("You win!\n");
klee_assert(0);
```

这样在win的时候,program当前下标字符为'#',assert断言会产生error

```
klee@nislv:~/lab$ klee -emit-all-errors maze_klee.bc
klee@nislv:~/lab$ ls klee-last| grep err 
test000140.assert.err
test000212.assert.err
test000251.assert.err
test000301.assert.err
klee@nislv:~/lab$ ktest-tool klee-last/test000140.ktest      
ktest file : 'klee-last/test000140.ktest'
args       : ['maze_klee.bc']
num objects: 1
object    0: name: b'program'
object    0: size: 28
object    0: data: b'sddwddddsddw\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

这样就可以方便的找到迷宫程序的所有4个解了.