# Hyperkernel实验

## 实验环境搭建
- ubuntu: 17.10

- QEMU: 2.10.1

- Z3: 4.5.0.0

- LLVM: 4.0.1

  这里直接使用了mancanfly大佬做好的docker镜像,在容器里安装一下llvm即可
```bash
docker pull mancanfly53373931/docker_hv6:hv6
docker run -t -d mancanfly53373931/docker_hv6:hv6
```
## hv6目录结构
```
.
|-- boot
|   `-- isolinux
|-- docs
|-- drivers
|-- hv6
|   |-- arch
|   |-- spec
|   `-- user
|-- include
|   `-- uapi
|-- irpy
|   |-- compiler
|   |-- libirpy
|   `-- test
|-- kernel
|   `-- include
|-- lib
|-- o.x86_64
|   |-- drivers
|   |-- hv6
|   |-- include
|   |-- irpy
|   |-- kernel
|   `-- lib
|-- scripts
|-- user
|   `-- lwip
`-- web
```
- hv6目录包含了hv6的内核实现
- irpy目录是用来解析hv6的LLVM IR的,用于转化成python后求解

比较重要的是hv6/spec目录下的文件,都是关于hv6内核的规格说明书specification,例如hv6/spec/kernel/spec/specs.py文件就是状态机规格说明书,hv6/spec/kernel/spec/top.py是声明规格说明书

## 内核验证
验证流程都在makefile里配置好了,只需要简单的
```
make hv6-verify
```
就可以对整个内核的正确性进行验证:

```
root@ubuntu:~/hv6# make hv6-verify
     PY2      hv6-verify
Using z3 v4.5.0.0
If(If(ULE(63, 18446744073709551615 + pid.3),
      4294967293,
      If(@proc_table->struct.proc::state.0(0, pid.3) == 4,
         If(ULE(8192, inpn.0),
            4294967274,
            If(@page_desc_table->struct.page_desc::pid.0(0,
                                        inpn.0) ==
               @current.0(0),
               If(And(Extract(63, 13, size.0) == 0,
                      ULE(Extract(12, 0, size.0), 4096)),
                  If(ULE(8192, outpn.0),
                     4294967274,
                     If(@page_desc_table->struct.page_desc::pid.0(0,
                                        outpn.0) ==
                        @current.0(0),
                        If(@page_desc_table->struct.page_desc::type.0(0,
                                        outpn.0) ==
                           3,
                           If(ULE(16, outfd.0),
                              If(Or(@proc_table->struct.proc::ipc_from.0(0,
                                        pid.3) ==
                                    0,
                                    @proc_table->struct.proc::ipc_from.0(0,
                                        pid.3) ==
                                    @current.0(0)),
                                 0,
...
```

整个验证过程需要比较长的时间,验证器支持对局部单个函数的验证,例如:

```
make hv6-verify -- -v --failfast HV6.test_sys_lseek 
```

就是单独验证lseek系统调用,验证结果如下:

```
root@ubuntu:~/hv6# make hv6-verify -- -v --failfast HV6.test_sys_lseek
     PY2      hv6-verify
Using z3 v4.5.0.0
test_sys_lseek (__main__.HV6) ... If(Not(ULE(16, fd.3)),
   If(Not(ULE(127,
              18446744073709551615 +
              @proc_table.0(0,
                            @current.0(0),
                            SignExt(32, fd.3)))),
      If(Or(Not(0 <= offset.0),
            Not(@file_table->struct.file::type.0(0,
                                        @proc_table.0(0,
                                        @current.0(0),
                                        SignExt(32, fd.3))) ==
                2)),
         4294967274,
         0),
      4294967274),
   4294967287)
ok

----------------------------------------------------------------------
Ran 1 test in 4.406s

OK
```

### 修改sys_lseek代码,触发验证失败

sys_lseek的代码在hv6/hv6/fd.c中:

```
int sys_lseek(int fd, off_t offset)
{
    fn_t fn;
    struct file *file;

    if (!is_fd_valid(fd))
        return -EBADF;
    fn = get_fd(current, fd);
    if (!is_file_type(fn, FD_INODE))
        return -EINVAL;
    if (offset < 0)
        return -EINVAL;
    file = get_file(fn);

    file->offset = offset;
    return 0;
}
```

对应的规格说明书代码如下:

```
def sys_lseek(old, fd, offset):
    cond = z3.And(
        is_fd_valid(fd),
        is_fn_valid(old.procs[old.current].ofile(fd)),
        old.files[old.procs[old.current].ofile(fd)].type == dt.file_type.FD_INODE,
        offset >= 0,
    )

    new = old.copy()

    fn = old.procs[old.current].ofile(fd)
    new.files[fn].offset = offset
    
    return cond, util.If(cond, new, old)
```

- 修改规格说明书,将offset放大100倍:

```
new.files[fn].offset = offset
```

结果如下:

```
root@ubuntu:~/hv6# make hv6-verify -- -v --failfast HV6.test_sys_lseek
     PY2      hv6-verify
Using z3 v4.5.0.0
test_sys_lseek (__main__.HV6) ... If(Not(ULE(16, fd.3)),
   If(Not(ULE(127,
              18446744073709551615 +
              @proc_table.0(0,
                            @current.0(0),
                            SignExt(32, fd.3)))),
      If(Or(Not(0 <= 100*offset.0),
            Not(@file_table->struct.file::type.0(0,
                                        @proc_table.0(0,
                                        @current.0(0),
                                        SignExt(32, fd.3))) ==
                2)),
         4294967274,
         0),
      4294967274),
   4294967287)
Could not prove, trying to find a minimal ce
Could not prove, trying to find a minimal ce
Could not prove, trying to find a minimal ce
Can not minimize condition further
Precondition
...
```

- 修改c代码,将offset放大100倍
```
    file->offset = offset * 100;
```
结果如下:

```
root@ubuntu:~/hv6# make hv6-verify -- -v --failfast HV6.test_sys_lseek
     CC_IR    o.x86_64/hv6/fd.ll
     GEN      o.x86_64/hv6/hv6.ll
     IRPY     o.x86_64/hv6/hv6.py
Parsing took 35.635 ms.
Emitting took 10533.7 ms.
     PY2      hv6-verify
Using z3 v4.5.0.0
test_sys_lseek (__main__.HV6) ... If(Not(ULE(16, fd.3)),
   If(Not(ULE(127,
              18446744073709551615 +
              @proc_table.0(0,
                            @current.0(0),
                            SignExt(32, fd.3)))),
      If(Or(Not(0 <= 100*offset.0),
            Not(@file_table->struct.file::type.0(0,
                                        @proc_table.0(0,
                                        @current.0(0),
                                        SignExt(32, fd.3))) ==
                2)),
         4294967274,
         0),
      4294967274),
   4294967287)
Could not prove, trying to find a minimal ce
Could not prove, trying to find a minimal ce
Could not prove, trying to find a minimal ce
Can not minimize condition further
Precondition
...
```

可以发现修改过c源码后,verifier在验证之前对c代码进行了重新编译

并且修改过的代码不能通过验证