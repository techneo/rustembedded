#Rust Embedded
## [19.06.2021]

https://docs.rust-embedded.org/book/start/

##Setup

After Rust installation

Cortex M4F - STM32F401
Cortex M3  - STM32F103 (fake)

``` sh
rustup target add thumbv7em-none-eabihf 
```

errro that rust-std not there for the target x86-gnu


Had to update the rust setup using 'rustup update'

Now it works

Erro executing cargo install cargo-binutils

had to remove all other gcc from path

Now successfull.

``` sh
rustup component add llvm-tools-preview 
```

for build scrips

``` sh
cargo install cargo-generate 
```

Need to have one 64bit GCC in path to install crates.


Arm gcc compiler and OpenOCD has to be installed
And both should be in path

ocd : https://github.com/xpack-dev-tools/openocd-xpack/releases


Also ST-link has to be installed from ST tools

Testing openocd with

``` sh 
openocd -f interface/stlink.cfg -f target/gd32f1x.cfg 
```
for the STM32F1 fake blue pill

##Create the first app


Auto generate from template

``` sh
    cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart 
    cargo build --target thumbv7em-none-eabihf 
    cargo build --target thumbv7em-none-eabihf --release
    cargo build 
```

In case of error zlib1__.dll error

ref:https://github.com/rust-lang/rust/issues/85422
Yes, at least enough so it works for me. Just rename any zlib binary, the library interface has been stable for a very long time, so it should just work. Or, if you are ok with using binaries received from strangers, take this: zlib1__.zip.
When you have zlib1__.dll, you need to make it visible. Either place it in any location visible in PATH environment variable, or place it next to the rust-lld.exe. For me it is located in rustup\toolchains\nightly-2021-05-12-x86_64-pc-windows-gnu\lib\rustlib\x86_64-pc-windows-gnu\bin. 

zlib : https://github.com/rust-lang/rust/files/6550164/zlib1__.zip
path: C:\Users\Admin\.rustup\toolchains\stable-x86_64-pc-windows-gnu\lib\rustlib\x86_64-pc-windows-gnu\bin
more info : https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order#standard-search-order-for-desktop-applications



##Inspecting target specific files

app exe : app/target/thumbv7em-none-eabihf/debug/app

### Read header

``` sh
    cargo readobj --bin app -- -file-headers
```

### Find size
``` sh
    cargo size --bin app --release -- -A
```


### Disassembly

``` sh
cargo objdump --bin app --release -- --disassemble --no-show-raw-insn --print-imm-hex

```


##Qemu

``` sh

E:\Programs\qemu\qemu-system-arm.exe -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target\thumbv7em-none-eabihf\debug\examples\hello

```

Output

``` console
Hello, World!
```

To simplify execution

- Add the QEMU to path
- Modify the .cargo/config file 

```
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"
```


###Debug

The code is loaded into Qemu in one terminal and the debug session happens in another terminal. 

``` sh
 t1> qemu-system-arm   -cpu cortex-m3   -machine lm3s6965evb   -nographic   -semihosting-config enable=on,target=native   -gdb tcp::3333   -S   -kernel target/thumbv7em-none-eabihf/debug/examples/hello


 t2> arm-none-eabi-gdb -q target/thumbv7em-none-eabihf/debug/examples/hello

```

The above steps use the gdb provided by the arm toolchain. Ensure that the path is correct. 

###Commands
- debug enable  target remote :3333
- read function list main
- set breakpoint at line break 13
- execute  continue
- step next


##Hardware

- update the memory.x file to reflect the correct RAM/ROM size for the controller selected

``` console
MEMORY
{
  /* NOTE 1 K = 1 KiBi = 1024 bytes */
  /* TODO Adjust these memory regions to match your device memory layout */
  /* These values correspond to the LM3S6965, one of the few devices QEMU can emulate */
  FLASH : ORIGIN = 0x8000000, LENGTH = 64K
  RAM : ORIGIN = 0x20000000, LENGTH = 20K
}
```

- update .cargo/config to reflect the correct cortex core
``` console
  target = "thumbv7m-none-eabi"        # Cortex-M3
```

- configure the openocd config file 
``` console
# Sample OpenOCD configuration for the STM32F3DISCOVERY development board

# Depending on the hardware revision you got you'll have to pick ONE of these
# interfaces. At any time only one interface should be commented out.

# Revision C (newer revision)
source [find interface/stlink.cfg]

# Revision A and B (older revisions)
# source [find interface/stlink-v2.cfg]

source [find target/gd32f1x.cfg]
```

###Debug

- pre setup
``` sh
t1> openocd
t2> arm-none-eabi-gdb target\thumbv7m-none-eabi\debug\examples\hello
```

- commands
    + debug enable  target remote :3333
    + load
    + set breakpoint at line break 13
    + execute  continue
    + step next

###Automate the debug process

We can configure a new openocd script to avoid typing multiple commands

``` console
target extended-remote :3333

# print demangled symbols
set print asm-demangle on

# detect unhandled exceptions, hard faults and panics
break DefaultHandler
break HardFault
break rust_begin_unwind

monitor arm semihosting enable

load

# start the process but immediately halt the processor
stepi
```

##Memory Mapped Register
