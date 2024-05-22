# Porting STM32CubeMX Projects to Zig

This repo serves as a guide for porting existing STM32CubeMX projects to the Zig C/C++ compiler, unlocking the ability to mix C/C++ and Zig code in your project.
It is intended to track major release versions of Zig, as Zig is still in development and subject to change. This repo is currently tracking Zig:

`0.12.0`  


## Project Setup

STM32CubeMX, using the "Makefile" toolchain selection, was used to generate a very simple blinky program for the STM32F750N8 MCU.
This MCU uses a Cortex M7, and supports hardware floating point operations. The program itself isn't the important part of this guide,
rather following along with what needs to be modified/changed to get generated STM32CubeMX code to compile with Zig's C/C++ compiler. 


## Methods

This repo examines two different methods for porting:
- Using `zig cc` as a "drop in" replacement for `arm-none-eabi-gcc` used in STM32CubeMX's generated Makefiles
- Using Zig's build system to replace STM32CubeMX's Makefiles

## Drop-in Replacement

### Information Gathering

Before we even touch Zig's compiler, it's useful to gather information on what exactly our *current* compiler is doing when we build.
Examing the [Makefile](./Makefile) generated by ST, I want to highlight a couple of things:

Compiler setup:
``` Makefile
PREFIX = arm-none-eabi-
# The gcc compiler bin path can be either defined in make command via GCC_PATH variable (> make GCC_PATH=xxx)
# either it can be added to the PATH environment variable.
ifdef GCC_PATH
CC = $(GCC_PATH)/$(PREFIX)gcc
AS = $(GCC_PATH)/$(PREFIX)gcc -x assembler-with-cpp
CP = $(GCC_PATH)/$(PREFIX)objcopy
SZ = $(GCC_PATH)/$(PREFIX)size
else
CC = $(PREFIX)gcc
AS = $(PREFIX)gcc -x assembler-with-cpp
CP = $(PREFIX)objcopy
SZ = $(PREFIX)size
endif
HEX = $(CP) -O ihex
BIN = $(CP) -O binary -S
```

This sets up what utility actually gets called at various steps in the compilation process. This isn't a tutorial on compilation, but one interesting thing of note is:

``` Makefile
AS = $(PREFIX)gcc -x assembler-with-cpp
```

The compiler used to compile assembly files (`.s/.S`) is just the normal `arm-none-eabi-gcc` with the arg `-x assembler-with-cpp` which roughly means: "The language I'm compiling is assembly code, but it also has C pre-processor commands (`#define`s, etc.) so please support those".

Some more critical lines to pay attention to in the order they appear:

``` Makefile
# cpu
CPU = -mcpu=cortex-m7

# fpu
FPU = -mfpu=fpv5-sp-d16

# float-abi
FLOAT-ABI = -mfloat-abi=hard

# mcu
MCU = $(CPU) -mthumb $(FPU) $(FLOAT-ABI)
```

This tells the compiler what architecture/cpu/floating-point hardware configuration to actually compile code for, we'll have to do the same equivalent setup with `zig cc` later (spoiler: the arguments are almost identical).

``` Makefile
LIBS = -lc -lm -lnosys 
```

Links in the C standard library, math library, and "nosys" library in that order. The "nosys" library provides non-functional stubs for all the normal "OS" level functions (`fwrite()`, `fopen()`, `exit()`, etc.) that aren't implemented due to not *having* an OS when running "bare metal". Confusingly, ST also auto-generates [syscalls.c](./Core/Src/syscalls.c) which *also* does this. That will come up later.

``` Makefile
-specs=nano.specs
```

You can find more information on spec files [here](https://gcc.gnu.org/onlinedocs/gcc/Spec-Files.html), however this particular one does the following:
- Any time `-lc` (the C standard library) is linked in, actually use `-lc_nano`, which is the "nano" variant of the newlib standard C library. 

Newlib, and newlib-nano, are bundled with `arm-none-eabi-gcc` as pre-compiled binaries. The compiler knows which ones to choose automatically based on `-mcpu`, `-mfpu`, `-mfloat-abi` and `-mthumb` compiler args. `zig cc` will not know which one to choose, or even where to find it by default, which we'll get to later.

Now we're *almost* ready to get started with porting, but there's one more extremely helpful thing we can do. If the linker argument `Wl,--verbose` is added to the linker args via `LDFLAGS` variable, it can give us some easy information about what precisely is getting linked into our code. Running `make | grep succeeded` produces the following:

```
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/thumb/v7e-m+fp/hard/crti.o succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/thumb/v7e-m+fp/hard/crtbegin.o succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/../../../../arm-none-eabi/lib/thumb/v7e-m+fp/hard/crt0.o succeeded
attempt to open build/main.o succeeded
attempt to open build/gpio.o succeeded
attempt to open build/stm32f7xx_it.o succeeded
attempt to open build/stm32f7xx_hal_msp.o succeeded
attempt to open build/stm32f7xx_hal_cortex.o succeeded
attempt to open build/stm32f7xx_hal_rcc.o succeeded
attempt to open build/stm32f7xx_hal_rcc_ex.o succeeded
attempt to open build/stm32f7xx_hal_flash.o succeeded
attempt to open build/stm32f7xx_hal_flash_ex.o succeeded
attempt to open build/stm32f7xx_hal_gpio.o succeeded
attempt to open build/stm32f7xx_hal_dma.o succeeded
attempt to open build/stm32f7xx_hal_dma_ex.o succeeded
attempt to open build/stm32f7xx_hal_pwr.o succeeded
attempt to open build/stm32f7xx_hal_pwr_ex.o succeeded
attempt to open build/stm32f7xx_hal.o succeeded
attempt to open build/stm32f7xx_hal_i2c.o succeeded
attempt to open build/stm32f7xx_hal_i2c_ex.o succeeded
attempt to open build/stm32f7xx_hal_exti.o succeeded
attempt to open build/stm32f7xx_hal_tim.o succeeded
attempt to open build/stm32f7xx_hal_tim_ex.o succeeded
attempt to open build/system_stm32f7xx.o succeeded
attempt to open build/sysmem.o succeeded
attempt to open build/syscalls.o succeeded
attempt to open build/startup_stm32f750xx.o succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/../../../../arm-none-eabi/lib/thumb/v7e-m+fp/hard/libc_nano.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/../../../../arm-none-eabi/lib/thumb/v7e-m+fp/hard/libm.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/../../../../arm-none-eabi/lib/thumb/v7e-m+fp/hard/libnosys.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/thumb/v7e-m+fp/hard/libgcc.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/../../../../arm-none-eabi/lib/thumb/v7e-m+fp/hard/libc_nano.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/thumb/v7e-m+fp/hard/libgcc.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/../../../../arm-none-eabi/lib/thumb/v7e-m+fp/hard/libc_nano.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/thumb/v7e-m+fp/hard/crtend.o succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/thumb/v7e-m+fp/hard/crtn.o succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/../../../../arm-none-eabi/lib/thumb/v7e-m+fp/hard/libc.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/../../../../arm-none-eabi/lib/thumb/v7e-m+fp/hard/libm.a succeeded
attempt to open /mygccarmpath/bin/../lib/gcc/arm-none-eabi/10.3.1/thumb/v7e-m+fp/hard/libgcc.a succeeded
```

Interesting! Putting the object files generated from our source code aside, we can see we linked in all sorts of stuff! Also note how the paths are tuned to our specific architecture: `.../thumb/v7e-m+fp/hard/...`. Try messing around with floating point compile flags to see how the paths change.

The following files are responsible for initializing the C runtime on your device. You can read more about them [here](https://www.filibeto.org/unix/tru64/lib/ossc/doc/cygnus_doc-99r1/html/6_embed/embcrt0_the_main_startup_file.html) but generally speaking they are what's responsible for gracefully starting/ending your C program, calling C "constructors" and "destructors", etc.
```
crt0.o
crti.o
crtbegin.o
crtend.o
crtn.o
```

And here are the actual pre-compiled library files corresponding to our `-l...` library includes seen earlier, with one additional one, `libgcc.a`. An explanation of that one can be found [here](https://gcc.gnu.org/onlinedocs/gccint/Libgcc.html);
```
libc_nano.a
libm.a
libnosys.a
libgcc.a
```

### Modifying our Makefile

Armed with our previously gathered information, we can make a new Makefile to use `zig cc` instead of `arm-none-eabi-gcc`. You can diff [DropinZigccMakefile](./DropinZigccMakefile) and [Makefile](./Makefile) to see what changed. I've marked + commented changed lines with `CHANGE NOTE:` so all explanations are contained within the modified Makefile. An overview of the steps taken:
- We switch our compiler to use `zig cc`
- We change some compiler arguments to match what `zig cc` expects
- We add an additional system include path using `-isystem` to point at our `arm-none-eabi-gcc` installation so we can manually link in pre-compiled libraries
With these modifications
- We ditch linking in `-lgcc` and `-lnosys`, see [DropinZigccMakefile](./DropinZigccMakefile) for a more detailed explanation

And that's it! We (or at least I) now have a blinky program that blinks as expected when compiled with `arm-none-eabi-gcc` OR `zig cc`. 

## Migrating To Zig's Build System

But we can do better, after all Makefiles are a bit of a pain to write. Zig also includes its own build-system. Documentation is on the... light... side for now, but there are enough examples out there (like this one!) to get something working. I won't go into the nitty gritty of how precisely Zig's build system works, but it should be fairly easy to look at our Makefile and see how that translates to [build.zig](./build.zig). 

Some specific callouts:

``` zig
const target = b.resolveTargetQuery(.{
    .cpu_arch = .thumb,
    .os_tag = .freestanding,
    .abi = .eabihf,
    .cpu_model = std.zig.CrossTarget.CpuModel{ .explicit = &std.Target.arm.cpu.cortex_m7 },
    .cpu_features_add = std.Target.arm.featureSet(&[_]std.Target.arm.Feature{std.Target.arm.Feature.fp_armv8d16sp}),
});
```

This does the equivalent of our `-mcpu`, `-mfpu`, `-mfloat-abi` and `-mthumb` flags from earlier. This describes our target, which is an arm processor using the thumb instruction set, it has no OS, it uses the "embedded application binary interface" with an "hf" at the end to signify hardware floating point, it is a Cortex M7 processor, and finally we manually add the feature that actually enables the hardware floating point instructions. That last one was the only one that was difficult to figure out, as while there are "features" named `vfp4d16sp`, and `vfp3d16sp`, there is NOT one named `vfp5d16sp`... However as best as my research serves, in LLVM land `fp_armv8d16sp` is functionally equivalent to our `-mfpu=fpv5-sp-d16` flag from earlier. It's worth noting you could forgo the feature addition in `resolveTargetQuery()` and just use the same compile flags as earlier for floating point, however I wanted to do this the most "Zig" way possible. For anyone who's ever written a toolchain file in CMake, this declarative way of defining a target architecture (complete with code-completion!) is pretty refreshing.

``` zig
blinky_exe.link_gc_sections = true;
blinky_exe.link_data_sections = true;
blinky_exe.link_function_sections = true;
```

This allows us to remove manually specified `-ffunction-sections` and `-fdata-sections` compile flags as well as `Wl,--gc-sections` linker flag. Zig does this for us now that we've asked it to.  

And now... We should just be able to run `zig build` and get a blink-tastic binary!

## Miscellaneous Notes + Thoughts
- Zig emits to stderror when `blinky_exe.setVerboseLink(true);` is used, see [here](https://github.com/ziglang/zig/issues/19410)
- When `blinky_exe.setVerboseLink(true);` is used, linker command appears to use "armelf_linux_eabi" as its triple which is... odd, and I don't think accurate

