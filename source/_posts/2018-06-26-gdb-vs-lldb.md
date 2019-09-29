---
title: GDB vs. LLDB
tags:
  - gdb
  - lldb
  - linux
date: 2018-06-26 20:56:13
---

[The LLDB Debugger](https://lldb.llvm.org/lldb-gdb.html)

## 执行命令（EXECUTION COMMANDS）

+ ### 执行程序（无参数）
```shell
(gdb) run                          | (lldb) process launch
(gdb) r                            | (lldb) run
                                   | (lldb) r
```
+ ### 带参数执行程序
```shell
$ gdb --args a.out 1 2 3           | $ lldb -- a.out 1 2 3
(gdb) run <args>                   | (lldb) process launch -- <args>
(gdb) r <args>                     | (lldb) r <args>
# 或
(gdb) set args 1 2 3               | (lldb) settings set target.run-args 1 2 3
(gdb) run                          | (lldb) run
```

<!-- more -->

+ ### 在程序运行前设置环境变量
```shell
(gdb) set env DEBUG 1              | (lldb) settings set target.env-vars DEBUG=1
                                   | (lldb) set se target.env-vars DEBUG=1
                                     | (lldb) env
# 运行程序时并设置环境变量
                                   | (lldb) process launch -v DEBUG=1
```
+ ### 在程序运行前删除环境变量
```shell
(gdb) unset env DEBUG              | (lldb) settings remove target.env-vars DEBUG
                                   | (lldb) set rem target.env-vars DEBUG
```
+ ### 显示程序运行时的参数
```shell
(gdb) show args                    | (lldb) settings show target.run-args

```

+ ### 附加现有进程
```shell
# 根据进程ID号
(gdb) attach 123                   | (lldb) process attach --pid 123
                                   | (lldb) attach -p 123
# 根据进程名称
(gdb) attach a.out                 | (lldb) process attach --name a.out
                                   | (lldb) pro at -n a.out
# 根据进程名称并等待运行
(gdb) attach --waitfor a.out       | (lldb) process attach --name a.out --waitfor
                                   | (lldb) pro at -n a.out -w
# 附加到远程支持GDB协议指定地址的进程
(gdb) target remote eorgadd:8000   | (lldb) gdb-remote eorgadd:8000
# 附加到本机支持GDB协议指定地址的进程
(gdb) target remote localhost:8000 | (lldb) gdb-remote 8000
# Attach to a Darwin kernel in kdp mode on system "eorgadd".
(gdb) kdp-reattach eorgadd         | (lldb) kdp-remote eorgadd
```
+ ### 调试步骤
``` shell
# 进入代码级单步调试（当前线程）
(gdb) step                         | (lldb) thread step-in
(gdb) s                            | (lldb) step
                                   | (lldb) s
# 结束代码级单步调试（当前线程）
(gdb) next                         | (lldb) thread step-over
(gdb) n                            | (lldb) next
                                   | (lldb) n
# 进入指令级单步调试（当前线程）
(gdb) stepi                        | (lldb) thread step-inst
(gdb) si                           | (lldb) si
# 结束指令级单步调试（当前线程）
(gdb) nexti                        | (lldb) thread step-inst-over
(gdb) ni                           | (lldb) ni
# 跳出当前帧（frame）
(gdb) finish                       | (lldb) thread step-out
                                   | (lldb) finish
# 从当前帧立即返回，带可选参数
(gdb) return <RETURN EXPRESSION>   | (lldb) thread return <RETURN EXPRESSION>
# 运行函数直到12行或离开当前函数
(gdb) until 12                     | (lldb) thread until 12
```


## 断点命令（BREAKPOINT COMMANDS）

+ ### 根据函数名设置断点
```shell
(gdb) break main                           | (lldb) breakpoint set --name main
                                           | (lldb) br s -n main
                                           | (lldb) b main
```
+ ### 根据文件和行号设置断点
```shell
(gdb) break test.c:12                      | (lldb) breakpoint set --file test.c --line 12
                                           | (lldb) br s -f test.c -l 12
                                           | (lldb) b test.c:12
```
+ ### 在C++中，设置所有basename为main处断点
```shell
(gdb) break main                           | (lldb) breakpoint set --method main
                                           | (lldb) br s -M main
```
+ ### 根据函数名正则表达式
```shell
(gdb) rbreak <regular-expression>          | (lldb) breakpoint set --func-regex <regular-expression>
                                           | (lldb) br s -r <regular-expression>
```
+ ### 条件断点
```shell
(gdb) break foo if strcmp(y, "hello") == 0 | (lldb) breakpoint set --name foo --condition '(int)strcmp(y, "hello") == 0'
```
+ ### 显示所有断点
```shell
(gdb) info break                           | (lldb) breakpoint list
                                           | (lldb) br l
```
+ ### 删除断点
```shell
(gdb) delete 1                             | (lldb) breakpoint delete 1
                                           | (lldb) br del 1
```


## 监视点命令（WATCHPOINT COMMANDS）

+ ### 在待写变量设置一个监视点
```shell
(gdb) watch global_var           | (lldb) watchpoint set variable global_var
                                 | (lldb) wa s v global_var
(gdb) watch -location g_char_ptr | (lldb) watchpoint set expression -- my_ptr
                                 | (lldb) wa s e -- my_ptr
```
+ ### 设置条件监视点
```shell
                                 | (lldb) watch set var global
                                 | (lldb) watchpoint modify -c '(global==5)'
```
+ ### 显示所有监视点
```shell
(gdb) info break                 | (lldb) watchpoint list
                                 | (lldb) watch l
```
+ ### 删除监视点
```shell
(gdb) delete 1                   | (lldb) watchpoint delete 1
                                 | (lldb) watch del 1
```


## 检查变量（EXAMINING VARIABLES）

+ ### 显示运行参数或当前帧本地变量
```shell
(gdb) info args    | (lldb) frame variable
(gdb) info locals  | (lldb) fr v
                   | (lldb) frame variable --no-args
                   | (lldb) fr v -a
```
+ ### 打印变量
```shell
(gdb) p bar        | (lldb) frame variable bar
                   | (lldb) fr v bar
                   | (lldb) p bar
```
+ ### 打印变量（16进制）
```shell
(gdb) p/x bar      | (lldb) frame variable --format x bar
                   | (lldb) fr v -f x bar
```
+ ### 打印全局变量
```shell
(gdb) p baz        | (lldb) target variable baz
                   | (lldb) ta v baz
```
+ ### 打印当前源文件中所有的全局或静态变量
```shell
                   | (lldb) target variable
                   | (lldb) ta v
```
+ ### 每次停下来的时候打印变量
```shell
(gdb) display argc | (lldb) target stop-hook add --one-liner "frame variable argc argv"
(gdb) display argv | (lldb) ta st a -o "fr v argc argv"
                   | (lldb) display argc
                   | (lldb) display argv
```
+ ### 仅当在指定函数中停下来的时候才打印变量
```shell
                   | (lldb) target stop-hook add --name main --one-liner "frame variable argc argv"
                   | (lldb) ta st a -n main -o "fr v argc argv"
```
+ ### 在指定类名中停下来的时候打印 `*this` 变量
```shell
                   | (lldb) target stop-hook add --classname MyClass --one-liner "frame variable *this"
                   | (lldb) ta st a -c MyClass -o "fr v *this"
```


## 执行表达式（EVALUATING EXPRESSIONS）

+ ### 在当前帧执行表达式
```shell
(gdb) print (int)printf("Print nine: %d.", 4 + 5) | (lldb) expr (int)printf("Print nine: %d.", 4 + 5)
(gdb) call (int)printf("Print nine: %d.", 4 + 5)  | (lldb) print (int)printf("Print nine: %d.", 4 + 5)
```
+ ### 创建变量并赋值
```shell
(gdb) set $foo  = 5                               | # 在 lldb 中执行在 C 语言中的表达式即可
(gdb) set variable $foo = 5                       | (lldb) expr unsigned int $foo = 5
(gdb) print $foo = 5                              |
(gdb) call $foo = 5                               |
# 指定变量类型
(gdb) set $foo = (unsigned int) 5                 |
```
+ ### 调用一个包含断点的函数并停下来
```shell
(gdb) set unwindonsignal 0                        | (lldb) expr -i 0 -- function_which_breakpoint()
(gdb) p function_which_breakpoint()               |
```
+ ### 调用会跪的函数并停下来
```shell
(gdb) set unwindonsignal 0                        | (lldb) expr -u 0 -- function_which_crashes()
(gdb) p function_which_crashes()                  |
```


## 检查线程状态
+ ### 显示当前程序线程
```shell
(gdb) info threads                   | (lldb) tread list
```
+ ### 选择线程
```shell
(gdb) thread 1                       | (lldb) thread select 1
                                     | (lldb) t 1
```
+ ### 显示当前线程堆栈
```shell
(gdb) bt                             | (lldb) thread backtrace
                                     | (lldb) bt
```
+ ### 显示所有线程的堆栈信息
```shell
(gdb) thread apply all bt            | (lldb) thread backtrace all
                                     | (lldb) bt all
```
+ ### 显示当前线程前N帧堆栈信息
```shell
(gdb) bt 5                           | (lldb) thread backtrace -c 5
                                     | (lldb) bt 5 # (lldb >= 169)
                                     | (lldb) bt -c 5 # (lldb >= 168)
```
+ ### 通过索引选择指定不通栈帧
```shell
(gdb) frame 12                       | (lldb) frame set 12
                                     | (lldb) fr s 12
                                     | (lldb) f 12
```
+ ### 显示当前栈帧信息
```shell
                                     | (lldb) frame info
```
+ ### 选择调用当前栈帧的帧
```shell
(gdb) up                             | (lldb) up
(gdb) up 2                           | (lldb) frame select --relative=1
```
+ ### 选择被当前栈帧调用的帧
```shell
(gdb) down                           | (lldb) down
(gdb) down 3                         | (lldb) frame select --relative=-3
                                     | (lldb) fr s -r=-1
```
+ ### 显示当前线程通用寄存器
```shell
(gdb) info registers                 | (lldb) register read
```
+ ### 修改当前线程寄存器
```shell
(gdb) p $rax = 123                   | (lldb) register write rax 123
```
+ ### 修改当前线程指令
```shell
(gdb) jump *$pc+8                    | (lldb) register write pc `$pc+8`
```
+ ### 显示当前线程所有寄存器
```shell
(gdb) info all-registers             | (lldb) register read --all
                                     | (lldb) re r -a
```
+ ### 显示当前线程指定寄存器
```shell
(gdb) info all-registers rax rsp rbp | register read rax rsp rbp
```
+ ### 用二进制显示当前线程寄存器
```shell
(gdb) p/t $rax                       | (lldb) register read --format binary rax
                                     | (lldb) re r -f b rax
                                     | (lldb) register read/t rax
                                     | (lldb) p/t $rax
```
+ ### 反汇编当前栈帧的当前函数
```shell
(gdb) disassemble                    | (lldb) disassemble --frame
                                     | (lldb) di -f
```
+ ### 反汇编指定函数
```shell
(gdb) disassemble main               | (lldb) disassemble --name main
                                     | (lldb) di -n main
```
+ ### 反汇编指定地址
```shell
(gdb) disassemble 0x1eb8 0x1ec3      | (lldb) disassemble --start-address 0x1eb8 --end-address 0x1ec3
                                     | (lldb) disassemble -s 0x1eb8 -e 0x1ec3
```
+ ### 反汇编N条指令
```shell
(gdb) x/20i 0x1eb8                   | (lldb) disassemble --start-address 0x1eb8 --count 20
                                     | (lldb) di -s 0x1eb8 -c 20
```
+ ### 显示源码及反汇编结果
```shell
                                     | (lldb) disassemble --frame --mixed
                                     | (lldb) di -f -m
```
+ ### 显示反汇编字节码(opcode bytes)
```shell
                                     | (lldb) disassemble --frame --bytes
                                     | (lldb) di -f -b
```
+ ### 反汇编当前行
```shell
                                     | (lldb) disassemble --line
                                     | (lldb) di -l
```


## 共享库查询命令（EXECUTABLE AND SHARED LIBRARY QUERY COMMANDS）

+ ### 显示依赖
```shell
(gdb) info shared | (lldb) image list
```


## 其他（MISCELLANEOUS）

+ ### 帮助命令
```shell
(gdb) apropos keyword                                    | (lldb) apropos keyword
```
+ ### 输出
```shell
(gdb) echo Here is some text\n                           | (lldb) script print "Here is some text"
```
+ ### 重新映射源码文件路径
```shell
(gdb) set pathname-substitutions /buildbot/path /my/path | (lldb) settings set target.source-map /buildbot/path /my/path
(gdb) directory                                          |
```
