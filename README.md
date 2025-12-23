# Fuzzware

Origin and official repo [fuzzware](https://github.com/fuzzware-fuzzer/fuzzware)

This is a fixed version for my own project.

# Installation

For a local setup, your system will have to have a list of local tooling installed to handle building unicorn, setting up virtual environments and finally running different pipeline components. You can see how to set those dependencies up in the [Docker file](dockerfile). Without installing all the dependencies first, different steps of the installation process will complain and you will be able to install them one by one.

To install locally, python == 3.10 is best:
```
sudo apt update
sudo apt install -y redis-server tmux python3-pip virtualenv virtualenvwrapper gcc-arm-none-eabi automake clang unzip cmake
pip3 install virtualenvwrapper
source /usr/share/virtualenvwrapper/virtualenvwrapper.sh

./install_local.sh
```

The script will set up two Python virtualenvs:
1. `fuzzware`: The virtualenv containing the local pipeline and emulator modules. This also includes the `fuzzware` executable which exposes different parts of the system.
2. `fuzzware-modeling`: The virtualenv used for performing symbolic execution-based MMIO access modeling. You should not try installing this without a virtualenv as `angr` is one of its dependencies.

To use Fuzzware from here, simply use the `fuzzware` virtualenv.
```
workon fuzzware
```

# Configuring Firmware Images For Fuzzing
Find a detailed overview of configuration options in [fuzzware-emulator/README_config.yml](https://github.com/fuzzware-fuzzer/fuzzware-emulator/blob/main/README_config.yml) and more in-depth documentation in [docs/
target_configuration.md](docs/target_configuration.md).

At minimum, you will need a bare-metal firmware blob and know where it is located in memory. With this, you can setup a memory map. For a firmware blob `fw.bin` located at address `0x08000000` in ROM, a config located in a newly created `examples/my-fw` directory would look like this:
```
include:
    - ../configs/hw/cortexm_memory.yml
    # For optional interrupts
    - ./../configs/fuzzing/round_robin_interrupts.yml

memory_map:
    rom: 
        base_addr: 0x08000000
        size: 0x800000
        permissions: r-x
        file: ./fw.bin
```
Alternatively, you may also try out the experimental `fuzzware genconfig` utility which creates a basic configuration based on an elf file (extracts the binary from the ELF file, parses sections, and creates an initial memory config).

This will get your firmware image up and running initially. There are additional configuration options to set which are specific to a firmware image and can support hardware features such as DMA, increase performance / decrease MMIO overhead, guide the firmware boot process, focus the fuzzer on specific interrupts, add introspection and debug symbols.

An inline-documentation of firmware image configuration features can be found in [the config README of the emulator](https://github.com/fuzzware-fuzzer/fuzzware-emulator/blob/main/README_config.yml). These configuration options allow you to configure different aspects concerning:
- Interrupt raising (When? Which? How often?)
- Firmware Boot guidance (snapshot state after boot is successful)
- Symbols
- Custom input regions (e.g. to feed input whenever a static DMA buffer is accessed)
- Custom code hooks (function replacement, providing input, corruption detection)

# Fuzzware Workflow

Workflow for fellow academics:

1. bin -> elf `arm-none-eabi-objcopy -I binary -O elf32-littlearm -B arm --set-start 0x08000000 {binary_path} {elf_path}`
2. Configure your target image (don't blindly trust `fuzzware genconfig -o {config_path} {elf_path}`)
3. Fuzz target: `fuzzware pipeline --run-for 24:00:00`
4. Collect coverage statistics: `fuzzware genstats coverage`
5. Find your coverage info in `fuzzware-project/stats`

If you want to get the best out of Fuzzware (as a human in the loop), you should prefer the following steps:

1. Build or obtain the target firmware image
2. Configure basic memory ranges: Create config manually or use `fuzzware genconfig` (works best with elf files, still take the output with a grain of salt and verify manually!)
3. Fuzz the target: `fuzzware pipeline`
4. Check coverage: `fuzzware cov`, `fuzzware cov -o cov.txt` and `fuzzware cov <target_function>`, `fuzzware replay --covering <target_function>`
5. Adapt the configuration: [fuzzware-emulator/README_config.yml](https://github.com/fuzzware-fuzzer/fuzzware-emulator/blob/main/README_config.yml) and fuzz again. If the image requires a rebuild, go to step 1. If the config needs adaption, goto step 3.
6. Once you are reasonably sure that meaningful functionality is reached in the current setup, it might make sense to scale up cores: `fuzzware pipeline -n 16`.
7. Check for crashes: `fuzzware genstats crashcontexts`
8. Replay and analyze crashes: `fuzzware replay -M -t mainXXX/fuzzers/fuzzerY/crashes/idZZZ`

There is a range of fuzzware utilities which we created that you may find useful along the way. The utils and their command-line arguments are documented in `fuzzware` itself:
```
fuzzware -h
```

For additional descriptions of different steps of the workflow, also check out
- How to [optimize the config](docs/target_configuration.md).
- How to [analyze coverage progress](docs/coverage_analysis.md).
- How to [analyze crashes](docs/crash_analysis.md).
