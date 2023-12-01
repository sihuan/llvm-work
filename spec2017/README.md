# 在 licheepi 4a 上安装运行 SPEC CPU2017 测试

### 准备工作

- 给 licheepi 4a 刷入最新的 revyos，见 https://wiki.sipeed.com/hardware/en/lichee/th1520/lpi4a/4_burn_image.html
- 下载 SPEC CPU2017 安装镜像
- [编译]安装 C、C++ 和 Fortran 编译器。

## 编译工具集

spec2017 没有提供 riscv 架构的二进制工具集，需自行编译，编译之前为了避免权限问题，需要复制安装文件到其他位置。

### 1. 挂载镜像复制文件（目标路径仅作示意）

   ```shell
   mount /path/to/cpu2017.iso /mnt -o loop
   mkdir /home/test/spec2017
   cp -r /mnt/* /home/test/spec2017_iso
   ```

### 2. 修改权限

   ```shell
   chmod -R +w /home/test/spec2017_iso
   ```

### 3. 解压缩源码包，以供编译

   为了方便后续 `packagetools` 打包，将源码包解压到 `$SPEC/tools` 目录

   ```shell
   export SPEC=/home/test/spec2017_iso
   cd $SPEC/install_archives
   tar xf tools-src.tar -C ..
   ```

### 4. 编译工具集，具体可以参考 https://www.spec.org/cpu2017/Docs/tools-build.html

编译过程中会遇到一些问题如下：

1. `config.guess` 和 `config.sub` 太过老旧，不能识别 riscv 架构
从以下链接下载最新版本的 `config.guess` 和 `config.sub` 
http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.guess
http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.sub
替换以下位置的旧版本
```
$SPEC/tools/src/specinvoke/
$SPEC/tools/src/specsum/build-aux/
$SPEC/tools/src/tar-1.28/build-aux/
$SPEC/tools/src/make-4.2.1/config/
$SPEC/tools/src/rxp-1.5.0/
$SPEC/tools/src/expat-2.1.0/conftools/
$SPEC/tools/src/xz-5.2.2/build-aux/
```

2. 编译 perl 时，可能会将 gcc 10 识别为，gcc 1.x
`$SPEC/tools/src/perl-5.24.0/Configure` 和 `$SPEC/tools/src/perl-5.24.0/cflags.SH` 中使用 `1*` 匹配 gcc 版本号，改为 `1.*`
使用 以下 patch 来修复
```diff
--- Configure
+++ Configure
@@ -4686,7 +4686,7 @@ else
 fi
 $rm -f try try.*
 case "$gccversion" in
-1*) cpp=`./loc gcc-cpp $cpp $pth` ;;
+1.*) cpp=`./loc gcc-cpp $cpp $pth` ;;
 esac
 case "$gccversion" in
 '') gccosandvers='' ;;
@@ -5438,7 +5438,7 @@ fi
 case "$hint" in
 default|recommended)
        case "$gccversion" in
-       1*) dflt="$dflt -fpcc-struct-return" ;;
+       1.*) dflt="$dflt -fpcc-struct-return" ;;
        esac
        case "$optimize:$DEBUGGING" in
        *-g*:old) dflt="$dflt -DDEBUGGING";;
@@ -5453,7 +5453,7 @@ default|recommended)
                ;;
        esac
        case "$gccversion" in
-       1*) ;;
+       1.*) ;;
        2.[0-8]*) ;;
        ?*)     set strict-aliasing -fno-strict-aliasing
                eval $checkccflag
@@ -5571,7 +5571,7 @@ case "$cppflags" in
     ;;
 esac
 case "$gccversion" in
-1*) cppflags="$cppflags -D__GNUC__"
+1.*) cppflags="$cppflags -D__GNUC__"
 esac
 case "$mips_type" in
 '');;
diff --git cflags.SH cflags.SH
index a50044e..fe1079a 100755
--- cflags.SH
+++ cflags.SH
@@ -164,7 +164,7 @@ esac
 
 case "$gccversion" in
 '') ;;
-[12]*) ;; # gcc versions 1 (gasp!) and 2 are not good for this.
+[12].*) ;; # gcc versions 1 (gasp!) and 2 are not good for this.
 Intel*) ;; # # Is that you, Intel C++?
 #
 # NOTE 1: the -std=c89 without -pedantic is a bit pointless.
```

3. `$SPEC/tools/src/TimeDate-2.30/t/getdate.t` 测试失败
需要做以下修改
```diff
--- getdate.t
+++ getdate.t
@@ -156,7 +156,7 @@ Jul 22 10:00:00 UTC 2002         ;1027332000
 !;
 
 require Time::Local;
-my $offset = Time::Local::timegm(0,0,0,1,0,70);
+my $offset = Time::Local::timegm(0,0,0,1,0,1970);
 
 @data = split(/\n/, $data);
```

4. 编译并打包
使用 `$SPEC/tools/src/buildtools` 脚本编译
```shell
export MAKEFLAGS=-j4
./buildtools
```
编译完成后在 `$SPEC/tools/bin` 目录下新建 `linux-riscv64` 目录，并在其中添加 `description` 文件，内容如下

```
For riscv64 Linux systems.
                              Built with GCC 13.2.0 (Debian 13.2.0-4revyos)
                              on an lpi4a running Debian GNU/Linux 12 (bookworm) riscv64.
```

使用 `$SPEC/bin/packagetools` 进行打包

```shell
./bin/packagetools linux-riscv64
```
## 安装 SPEC CPU 2017 并进行测试

### 安装

在安装文件夹目录，执行（安装目标目录仅供参考）

```shell
 ./install.sh -u linux-riscv64 -d /home/test/spec2017
```

### 配置

运行测试前配置环境，设置堆栈大小

```shell
source shrc
ulimit -s unlimited
```

编辑配置文件，配置文件位于 `config` 目录下，在 `licheepi 4a` 上，使用 clang、clang++、flang-new 作为编译器的样例配置如下（需要编译安装或从软件源安装 `clang-17`、`flang-17`）

配置中遇到的一些问题

1. clang 16 将 `Wimplicit-int` 从警告升级为了错误，对于较新版本的 clang 需要添加 `-Wno-implicit-int` 参数来正常编译 spec2017 的老代码

llvm-test.cfg

```
#------------------------------------------------------------------------------
# SPEC CPU(R) 2017 config for clang on riscv64
#------------------------------------------------------------------------------
#
# Usage: (1) Copy this to a new name
#             cd $SPEC/config
#             cp Example-x.cfg myname.cfg
#        (2) Change items that are marked 'EDIT' (search for it)
#
# SPEC tested this config file with:
#    Compiler version(s):    Various.  See note "Older GCC" below.
#    Operating system(s):    Oracle Linux Server 6, 7, 8  / 
#                            Red Hat Enterprise Linux Server 6, 7, 8
#                            SUSE Linux Enterprise Server 15
#                            Ubuntu 19.04
#    Hardware:               Xeon, EPYC
#
# If your system differs, this config file might not work.
# You might find a better config file at https://www.spec.org/cpu2017/results
#
# Note: Older GCC
#
#   Please use the newest GCC that you can. The default version packaged with 
#   your operating system may be very old; look for alternate packages with a
#   newer version.  
#
#   If you have no choice and must use an old version, here is what to expect:
#
#    - "peak" tuning: Several benchmarks will fail at peak tuning if you use
#                     compilers older than GCC 7.  
#                     In that case, please use base only. 
#                     See: https://www.spec.org/cpu2017/Docs/overview.html#Q16
#                          https://www.spec.org/cpu2017/Docs/config.html#tune
#                     Peak tuning is expected to work for all or nearly all 
#                     benchmarks as of GCC 7 or later.  
#                     Exception: 
#                        - See topic "628.pop2_s basepeak", below.
#
#    - "base" tuning: This config file is expected to work for base tuning with 
#                     GCC 4.8.5 or later
#                     Exception: 
#                      - Compilers vintage about 4.9 may need to turn off the
#                        tree vectorizer, by adding to the base OPTIMIZE flags:
#                             -fno-tree-loop-vectorize
#
# Unexpected errors?  Try reducing the optimization level, or try removing: 
#                           -march=native
#
# Compiler issues: Contact your compiler vendor, not SPEC.
# For SPEC help:   https://www.spec.org/cpu2017/Docs/techsupport.html
#------------------------------------------------------------------------------


#--------- Label --------------------------------------------------------------
# Arbitrary string to tag binaries (no spaces allowed)
#                  Two Suggestions: # (1) EDIT this label as you try new ideas.
%ifndef %{label}
%   define label "clang17-riscv64"           # (2)      Use a label meaningful to *you*.
%endif


#--------- Preprocessor -------------------------------------------------------
%ifndef %{bits}                # EDIT to control 32 or 64 bit compilation.  Or,
%   define  bits        64     #      you can set it on the command line using:
%endif                         #      'runcpu --define bits=nn'

%ifndef %{build_ncpus}         # EDIT to adjust number of simultaneous compiles.
%   define  build_ncpus 4      #      Or, you can set it on the command line:
%endif                         #      'runcpu --define build_ncpus=nn'

# Don't change this part.
%if %{bits} == 64
%   define model         -march=rv64gc
%elif %{bits} == 32
%   define model         -march=rv32gc
%else
%   error Please define number of bits - see instructions in config file
%endif
%if %{label} =~ m/ /
%   error Your label "%{label}" contains spaces.  Please try underscores instead.
%endif
%if %{label} !~ m/^[a-zA-Z0-9._-]+$/
%   error Illegal character in label "%{label}".  Please use only alphanumerics, underscore, hyphen, and period.
%endif


#--------- Global Settings ----------------------------------------------------
# For info, see:
#            https://www.spec.org/cpu2017/Docs/config.html#fieldname
#   Example: https://www.spec.org/cpu2017/Docs/config.html#tune

command_add_redirect = 1
flagsurl             = $[top]/config/flags/clang.xml
ignore_errors        = 1
iterations           = 1
label                = %{label}-%{bits}
line_width           = 1020
log_line_width       = 1020
makeflags            = --jobs=%{build_ncpus}
mean_anyway          = 1
output_format        = txt,html,cfg,pdf,csv
preenv               = 1
reportable           = 0
tune                 = base,peak  # EDIT if needed: set to "base" for old GCC. 
                                  #      See note "Older GCC" above.


#--------- How Many CPUs? -----------------------------------------------------
# Both SPECrate and SPECspeed can test multiple chips / cores / hw threads
#    - For SPECrate,  you set the number of copies.
#    - For SPECspeed, you set the number of threads.
# See: https://www.spec.org/cpu2017/Docs/system-requirements.html#MultipleCPUs
#
#    q. How many should I set?
#    a. Unknown, you will have to try it and see!
#
# To get you started, some suggestions:
#
#     copies - This config file defaults to testing only 1 copy.   You might
#              try changing it to match the number of cores on your system,
#              or perhaps the number of virtual CPUs as reported by:
#                     grep -c processor /proc/cpuinfo
#              Be sure you have enough memory.  See:
#              https://www.spec.org/cpu2017/Docs/system-requirements.html#memory
#
#     threads - This config file sets a starting point.  You could try raising
#               it.  A higher thread count is much more likely to be useful for
#               fpspeed than for intspeed.
#
intrate,fprate:
   copies           = 4   # EDIT to change number of copies (see above)
intspeed,fpspeed:
   threads          = 4   # EDIT to change number of OpenMP threads (see above)


#------- Compilers ------------------------------------------------------------
default:
#  EDIT: The parent directory for your compiler.
#        Do not include the trailing /bin/
#        Do not include a trailing slash
#  Examples:
#   1  On a Red Hat system, you said:
#      'yum install devtoolset-9'
#      Use:                 %   define gcc_dir "/opt/rh/devtoolset-9/root/usr"
#
#   2  You built GCC in:                        /disk1/mybuild/gcc-10.1.0/bin/gcc
#      Use:                 %   define gcc_dir "/disk1/mybuild/gcc-10.1.0"
#
#   3  You want:                                /usr/bin/gcc
#      Use:                 %   define gcc_dir "/usr"
#      WARNING: See section "Older GCC" above.
#
%ifndef %{gcc_dir}
%   define  gcc_dir        "/usr"  # EDIT  above)
%endif

# EDIT: If your compiler version is 10 or greater, you must enable the next 
#       line to avoid compile errors for several FP benchmarks
#
#%define GCCge10  # EDIT: remove the '#' from column 1 if using GCC 10 or later

# EDIT if needed: the preENV line adds library directories to the runtime
#      path.  You can adjust it, or add lines for other environment variables.
#      See: https://www.spec.org/cpu2017/Docs/config.html#preenv
#      and: https://gcc.gnu.org/onlinedocs/gcc/Environment-Variables.html
   preENV_LD_LIBRARY_PATH  = %{gcc_dir}/lib64/:%{gcc_dir}/lib/:/lib64
  #preENV_LD_LIBRARY_PATH  = %{gcc_dir}/lib64/:%{gcc_dir}/lib/:/lib64:%{ENV_LD_LIBRARY_PATH}
   SPECLANG                = %{gcc_dir}/bin/
   CC                      = $(SPECLANG)clang-17   -Wno-implicit-int   $(model)
   CXX                     = $(SPECLANG)clang++-17     -std=c++98    $(model)
   FC                      = $(SPECLANG)flang-new-17    $(model)
   # How to say "Show me your version, please"
   CC_VERSION_OPTION       = --version
   CXX_VERSION_OPTION      = --version
   FC_VERSION_OPTION       = --version

default:
%if %{bits} == 64
   sw_base_ptrsize = 64-bit
   sw_peak_ptrsize = 64-bit
%else
   sw_base_ptrsize = 32-bit
   sw_peak_ptrsize = 32-bit
%endif
#--- No Fortran -------
# Fortran benchmarks are not expected to work with this config file.
# If you wish, you can run the other benchmarks using:
#    runcpu no_fortran           - all CPU 2017 benchmarks that do not use Fortran
#    runcpu intrate_no_fortran   - integer rate benchmarks that do not use Fortran
#    runcpu intspeed_no_fortran  - integer speed benchmarks that do not use Fortran
#    runcpu fprate_no_fortran    - floating point rate benchmarks that do not use Fortran
#    runcpu fpspeed_no_fortran   - floating point speed benchmarks that do not use Fortran
#
# If you *do* have a Fortran compiler, then you will need to set the correct options for
# 'FC' and 'FC_VERSION_OPTION' just above -- see
#                http://www.spec.org/cpu2017/Docs/config.html#makeCompiler
#  You must also remove the 2 lines right after this comment.
#any_fortran:
#   fail_build = 1

#--------- Portability --------------------------------------------------------
default:               # data model applies to all benchmarks
%if %{bits} == 32
    # Strongly recommended because at run-time, operations using modern file
    # systems may fail spectacularly and frequently (or, worse, quietly and
    # randomly) if a program does not accommodate 64-bit metadata.
    EXTRA_PORTABILITY = -D_FILE_OFFSET_BITS=64
%else
    EXTRA_PORTABILITY = -DSPEC_LP64
%endif

# Benchmark-specific portability (ordered by last 2 digits of bmark number)

500.perlbench_r,600.perlbench_s:  #lang='C'
%if %{bits} == 32
%   define suffix IA32
%else
%   define suffix X64
%endif
   PORTABILITY   = -DSPEC_LINUX_%{suffix}

521.wrf_r,621.wrf_s:  #lang='F,C'
   CPORTABILITY  = -DSPEC_CASE_FLAG -DSPEC_LP64 
#   FPORTABILITY  = -fconvert=big-endian

523.xalancbmk_r,623.xalancbmk_s:  #lang='CXX'
   PORTABILITY   = -DSPEC_LINUX

526.blender_r:  #lang='CXX,C'
   PORTABILITY   = -funsigned-char -DSPEC_LINUX

527.cam4_r,627.cam4_s:  #lang='F,C'
   PORTABILITY   = -DSPEC_CASE_FLAG  -DSPEC_LP64 

619.lbm_s,627.cam4_s,628.pop2_s,638.imagick_s,644.nab_s,657.xz_s:
   PORTABILITY = -fopenmp -I/usr/lib/gcc/riscv64-linux-gnu/10/include

628.pop2_s:  #lang='F,C'
   CPORTABILITY  = -DSPEC_CASE_FLAG -DSPEC_LP64 
   EXTRA_OPTIMIZE = -fopenmp
#   FPORTABILITY  = -fconvert=big-endian

#----------------------------------------------------------------------
#       GCC workarounds that do not count as PORTABILITY 
#----------------------------------------------------------------------
# The workarounds in this section would not qualify under the SPEC CPU
# PORTABILITY rule.  
#   - In peak, they can be set as needed for individual benchmarks.
#   - In base, individual settings are not allowed; set for whole suite.
# See: 
#     https://www.spec.org/cpu2017/Docs/runrules.html#portability
#     https://www.spec.org/cpu2017/Docs/runrules.html#BaseFlags
#
# Integer workarounds - peak
#
   500.perlbench_r,600.perlbench_s=peak:    # https://www.spec.org/cpu2017/Docs/benchmarks/500.perlbench_r.html
      EXTRA_CFLAGS = -fno-strict-aliasing -fno-unsafe-math-optimizations -fno-finite-math-only  
   502.gcc_r,602.gcc_s=peak:                # https://www.spec.org/cpu2017/Docs/benchmarks/502.gcc_r.html
      EXTRA_CFLAGS = -fno-strict-aliasing -fgnu89-inline
   505.mcf_r,605.mcf_s=peak:                # https://www.spec.org/cpu2017/Docs/benchmarks/505.mcf_r.html
      EXTRA_CFLAGS = -fno-strict-aliasing 
   525.x264_r,625.x264_s=peak:              # https://www.spec.org/cpu2017/Docs/benchmarks/525.x264_r.html
      EXTRA_CFLAGS = -fcommon -I/usr/lib/gcc/riscv64-linux-gnu/10/include
   638.imagick_s=peak:
      EXTRA_OPTIMIZE = -fopenmp
   644.nab_s=peak:
      EXTRA_OPTIMIZE = -fopenmp
   654.roms_s=peak:
      EXTRA_OPTIMIZE = -fopenmp

#
# Integer workarounds - base - combine the above - https://www.spec.org/cpu2017/Docs/runrules.html#BaseFlags
#
   intrate,intspeed=base:
      EXTRA_COPTIMIZE = -fgnu89-inline  -fno-strict-aliasing -fno-unsafe-math-optimizations -fno-finite-math-only -fcommon
   525.x264_r,625.x264_s=base:              # https://www.spec.org/cpu2017/Docs/benchmarks/525.x264_r.html
      EXTRA_CFLAGS = -fcommon -I/usr/lib/gcc/riscv64-linux-gnu/10/include
   557.xz_r,657.xz_s=base:
      EXTRA_CFLAGS = -fopenmp -I/usr/lib/gcc/riscv64-linux-gnu/10/include

   fpspeed=base:
#      EXTRA_LDFLAGS        = -Wl,-stack_size,0x10000000,-fgnu89-inline
      EXTRA_CFLAGS = -fno-strict-aliasing -fno-unsafe-math-optimizations -fno-finite-math-only -fgnu89-inline -fcommon
#
# Floating Point workarounds - peak
#
   511.povray_r=peak:                       # https://www.spec.org/cpu2017/Docs/benchmarks/511.povray_r.html
      EXTRA_CFLAGS = -fno-strict-aliasing 
   521.wrf_r,621.wrf_s=peak:                # https://www.spec.org/cpu2017/Docs/benchmarks/521.wrf_r.html
%     ifdef %{GCCge10}                      # workaround for GCC v10 (and presumably later)
         EXTRA_FFLAGS = -fallow-argument-mismatch
%     endif
   527.cam4_r,627.cam4_s=peak:              # https://www.spec.org/cpu2017/Docs/benchmarks/527.cam4_r.html
     EXTRA_CFLAGS = -fno-strict-aliasing 
%     ifdef %{GCCge10}                      # workaround for GCC v10 (and presumably later)
        EXTRA_FFLAGS = -fallow-argument-mismatch
%     endif
   # See also topic "628.pop2_s basepeak" below
   628.pop2_s=peak:                         # https://www.spec.org/cpu2017/Docs/benchmarks/628.pop2_s.html
%     ifdef %{GCCge10}                      # workaround for GCC v10 (and presumably later)
         EXTRA_FFLAGS = -fallow-argument-mismatch
%     endif
#
# FP workarounds - base - combine the above - https://www.spec.org/cpu2017/Docs/runrules.html#BaseFlags
#
   fprate,fpspeed=base:    
      EXTRA_CFLAGS = -fno-strict-aliasing 
%     ifdef %{GCCge10}                      # workaround for GCC v10 (and presumably later)
         EXTRA_FFLAGS = -fallow-argument-mismatch
%     endif


#-------- Tuning Flags common to Base and Peak --------------------------------
#
# Speed (O#penMP and Autopar allowed)
#
%if %{bits} == 32
   intspeed,fpspeed:
   #
   # Many of the speed benchmarks (6nn.benchmark_s) do not fit in 32 bits
   # If you wish to run SPECint2017_speed or SPECfp2017_speed, please use
   #
   #     runcpu --define bits=64
   #
   fail_build = 1
%else
   intspeed,fpspeed:
      EXTRA_OPTIMIZE =  -DSPEC_OPENMP  -DUSE_OPENMP 
   fpspeed:
      #
      # 627.cam4 needs a big stack; the preENV will apply it to all
      # benchmarks in the set, as required by the rules.
      #
      preENV_OMP_STACKSIZE = 120M
%endif

#--------  Base Tuning Flags ----------------------------------------------
# EDIT if needed -- If you run into errors, you may need to adjust the
#                   optimization - for example you may need to remove
#                   the -march=native.   See topic "Older GCC" above.
#
default=base:     # flags for all base
   OPTIMIZE      = -g -O3


#--------  Peak Tuning Flags ----------------------------------------------
default=peak:
   OPTIMIZE         = -g -Ofast
   #PASS1_FLAGS      = -fprofile-instr-generate=pgo_data
   #PASS2_FLAGS      = -fprofile-instr-use
   #fdo_post1        = xcrun llvm-profdata merge -output=default.profdata pgo_data


# 628.pop2_s basepeak: iDepending on the interplay of several optimizations,
#            628.pop2_s might not validate with peak tuning.  Use the base 
#            version instead.  See: 
#            https:// www.spec.org/cpu2017/Docs/benchmarks/628.pop2_s.html
628.pop2_s=peak:  
   basepeak         = yes


#------------------------------------------------------------------------------
# Tester and System Descriptions - EDIT all sections below this point
#------------------------------------------------------------------------------
#   For info about any field, see
#             https://www.spec.org/cpu2017/Docs/config.html#fieldname
#   Example:  https://www.spec.org/cpu2017/Docs/config.html#hw_memory
#-------------------------------------------------------------------------------

#--------- EDIT to match your version -----------------------------------------
default:
   sw_compiler001   = Debian clang version 17.0.0 (+rc4-1~exp5revyos1)

#--------- EDIT info about you ------------------------------------------------
# To understand the difference between hw_vendor/sponsor/tester, see:
#     https://www.spec.org/cpu2017/Docs/config.html#test_sponsor
intrate,intspeed,fprate,fpspeed: # Important: keep this line
   hw_vendor          = sipeed
   tester             = PLCT
   test_sponsor       = PLCT
   license_num        = 0
#  prepared_by        = # Ima Pseudonym                       # Whatever you like: is never output


#--------- EDIT system availability dates -------------------------------------
intrate,intspeed,fprate,fpspeed: # Important: keep this line
                        # Example                             # Brief info about field
   hw_avail           = May-2023                            # Date of LAST hardware component to ship
   sw_avail           = Nov-2023                            # Date of LAST software component to ship
   fw_bios            = Nov-2022    # Firmware information

#--------- EDIT system information --------------------------------------------
intrate,intspeed,fprate,fpspeed: # Important: keep this line
                        # Example                             # Brief info about field
   hw_cpu_name        = TH1520               # chip name
   hw_cpu_nominal_mhz = 2000MHz                                # Nominal chip frequency, in MHz
   hw_cpu_max_mhz     = 2000MHz                                # Max chip frequency, in MHz
#  hw_disk            = # 9 x 9 TB SATA III 9999 RPM          # Size, type, other perf-relevant info
   hw_model           =  Licheepi 4a                   # system model name
   hw_nchips          =  1                                  # number chips enabled
   hw_ncores          = 4                                # number cores enabled
#   hw_ncpuorder       = 1                           # Ordering options
   hw_nthreadspercore = 1                                   # number threads enabled per core
   hw_other           = None              # Other perf-relevant hw, or "None"

#  hw_memory001       = # 999 GB (99 x 9 GB 2Rx4 PC4-2133P-R, # The 'PCn-etc' is from the JEDEC
#  hw_memory002       = # running at 1600 MHz)                # label on the DIMM.

   hw_pcache          =  64 KB I +  64 Kb D on chip per core  # Primary cache size, type, location
   hw_scache          = 1MB L2 Cache       # Second cache or "None"
   hw_tcache          = None           # Third  cache or "None"
   hw_ocache          = None  # Other cache or "None"

#  sw_file            = # ext99                               # File system
#  sw_os001           = # Linux Sailboat                      # Operating system
#  sw_os002           = # Distribution 7.2 SP1                # and version
   sw_other           = None              # Other perf-relevant sw, or "None"
#  sw_state           = # Run level 99                        # Software state.

#   power_management   = # briefly summarize power settings 

# Note: Some commented-out fields above are automatically set to preliminary
# values by sysinfo
#       https://www.spec.org/cpu2017/Docs/config.html#sysinfo
# Uncomment lines for which you already know a better answer than sysinfo
```

### 测试

INT SPEED

```
runcpu -c llvm-test.cfg --noreportable -n 1 -I -T base intspeed
```



INT RATE

```
runcpu -c llvm-test.cfg --noreportable -n 1 -I -T base -C 4 intrate
```



FP SPEED

```
runcpu -c llvm-test.cfg --noreportable -n 1 -I -T base fpspeed
```



FP RATE

```
runcpu -c llvm-test.cfg --noreportable -n 1 -I -T base -C 4 fprate
```

如果仅为了测试编译编译情况，可以使用最小的测试范围 `--size=test`，并指定具体的用例

如 `**runcpu** --config llvm.cfg --size=test 648.exchange2_s`

### 结果

测试 riscv 上 llvm flang 编译器编译 spec2017 各算例，分别运行了由/含 Fortran 编写的测试

结果见 [results](./results)
