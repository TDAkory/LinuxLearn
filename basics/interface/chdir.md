# [chdir](https://man7.org/linux/man-pages/man2/chdir.2.html)

> changes the current working directory of the calling process to the directory specified in path.

```c
#include<stdio.h>
#include<unistd.h> 
int main()
{   
    char s[100];
  
    // printing current working directory
    printf("%s\n", getcwd(s, 100));
  
    // using the command
    chdir("..");
  
    // printing current working directory
    printf("%s\n", getcwd(s, 100));
  
    // after chdir is executed
    return 0;
}
```

```shell
zhaojieyi@n227-088-244 /d/h/z/C/self> ./a.out
/data00/home/zhaojieyi/CppWorkShop/self
/data00/home/zhaojieyi/CppWorkShop
```
