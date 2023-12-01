# licheepi 4a 上 flang 编译测试 spec2017 各算例结果

> 由于指定了个别用例单独进行测试，所以报告中由于其他用例未被运行会有 `INVALID RUN` 提示

### 548.exchange2_r

代码：Fortran ~1k 行

详细日志见 [548](./548)

548.exchange2_r base 编译成功，运行成功

548.exchange2_r peak 编译成功，运行成功

### 648.exchange2_s

代码：Fortran ~1k 行

详细日志见 [648](./648)

648.exchange2_s base 编译成功，运行成功

648.exchange2_s peak 编译成功，运行成功

### 503.bwaves_r

代码：Fortran ~1k 行

详细日志见 [503](./503)

503.bwaves_r base 编译成功

503.bwaves_r peak 编译成功

运行测试时段错误退出。

### 603.bwaves_s

代码：Fortran ~1k 行

详细日志见 [603](./603)

603.bwaves_r base 编译成功

603.bwaves_r peak 编译成功

运行测试时段错误退出。

### 507.cactuBSSN_r

代码：C++, C, Fortran ~257k 行

详细日志见 [507](./507)

507.cactuBSSN_r base 编译成功，运行成功

507.cactuBSSN_r peak 编译成功，运行成功

### 607.cactuBSSN_s

代码：C++, C, Fortran ~257k 行

详细日志见 [607](./607)

607.cactuBSSN_r base 编译成功，运行成功

607.cactuBSSN_r peak 编译成功，运行成功

### 521.wrf_r

代码：C, Fortran ~991k 行

详细日志见 [result022](./result022)

521.wrf_r base 编译成功

521.wrf_r peak 编译成功

运行测试 Fortran runtime error

### 621.wrf_s

代码：C, Fortran ~991k 行

详细日志见 [result022](./result022)

621.wrf_r base 编译 linker 报错

621.wrf_r peak 编译 linker 报错

### 628.pop2_s

代码：C, Fortran ~338k 行

详细日志见 [result022](./result022)

628.pop2_s base 编译报错

```
error: loc("/home/sipeed/spec2017/benchspec/CPU/628.pop2_s/build/build_base_clang17-riscv64-64.0000/ecosys_mod.fppized.f90":5519:10): flang/lib/Lower/ConvertExpr.cpp:4453: not yet implemented: character array expression temp with dynamic length
LLVM ERROR: aborting
```

### 649.fotonik3d_s

代码：Fortran ~14k 行

详细日志见 [result022](./result022)

649.fotonik3d_s base 编译成功，运行成功

649.fotonik3d_s peak 编译成功，运行成功

### 554.roms_r

代码：Fortran ~210k 行

详细日志见 [result022](./result022)

554.roms_r base 编译成功，运行成功

554.roms_r peak 编译成功，运行成功

### 654.roms_s

代码：Fortran ~210k 行

详细日志见 [result022](./result022)

654.roms_r base 编译时 linker 报错

654.roms_r peak 编译成功，运行成功