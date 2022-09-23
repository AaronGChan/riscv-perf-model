# Generating Input for Olympia

Olympia can take input in two formats: JSON and
[STF](https://github.com/sparcians/stf_spec).

JSON files are usually reserved for doing *very* small and quick
"what-if" analysis or general testing.  JSON format should not be used
to represent an entire program.

STF format is the preferred format to represent a workload to run on
Olympia.  STFs are typically created by a RISC-V functional model that
has been instrumented to generate one.

## JSON Inputs

JSON inputs are typically generated by hand.  There are no automated
tools in this repository (or in
[Sparcians](https://github.com/sparcians)) that will create these
files.

The grammar for the JSON file is simple.  Keep in mind that JSON does
not support comments.

Top level:

```
[
   {
      ... instruction record 1
   },
   ....
   {
      ... instruction record N
   }
]
```
Instruction record can have:
```
    {
        "mnemonic" : "<risc-v mnemonic>",

        ... for integer
        "rs1"      :  <numeric value, decimal>,
        "rs2"      :  <numeric value, decimal>,
        "rd"       :  <numeric value, decimal>,

        ... for float
        "fs1"      :  <numeric value, decimal>,
        "fs2"      :  <numeric value, decimal>,
        "fd"       :  <numeric value, decimal>,

        ... for immediates
        "imm"      :  <numeric value, decimal>,

        ... for branches, loads, stores
        "vaddr"    :  "<string value>",
    }
```

Example:
```
[
    {
        "mnemonic": "add",
        "rs1": 1,
        "rs2": 2,
        "rd": 3
    },
    {
        "mnemonic": "mul",
        "rs1": 3,
        "rs2": 1,
        "rd": 4
    },
    {
        "mnemonic": "lw",
        "rs1": 4,
        "rs2": 3,
        "rd":  5,
        "vaddr" : "0xdeadbeef"
    }

]
```

To run the JSON file, just provide it to olympia:
```
cd build
./olympia ../traces/example_json.json`
```

## STF Inputs

Using the [stf_lib]() in the [Sparcians](https://github.com/sparcians)
repo, Olympia is capable of reading a RISC-V STF trace generated from
a functional simulator.  Included in Olympia (in this directory) is a
trace of the "hot spot" of Dhrystone generated from the Dromajo
simulator.

To run the STF file, just provide it to olympia:
```
cd build
./olympia ../traces/dhrystone.zstf
```

### Generating an STF Trace with Dromajo

#### Build an STF-Capable Dromajo

Included with olympia is a patch to Dromajo to enable it to generate
an STF trace.  This patch must be applied to Dromajo before a trace
can be generated.

```
# Enter into the STF trace generation directory (not required)
cd olympia/traces/stf_trace_gen

# Clone Dromajo from Chips Alliance and cd into it
git clone https://github.com/chipsalliance/dromajo

# Checkout a Known-to-work SHA
git checkout 86125b31

# Apply the patch
cd dromajo
git apply ../dromajo_stf_lib.patch

# Create a sym link to the stf_lib used by Olympia
ln -s ../../../stf_lib

# Build Dromajo
mkdir build; cd build
cmake ..
make
```
If compilation was successful, dromajo will have support for generating an STF trace:
```
./dromajo
usage: ./dromajo {options} [config|elf-file]
       .... other stuff
       --stf_trace <filename>  Dump an STF trace to the given file
       .... more stuff
```

#### Instrument a Workload for Tracing

Dromajo can run either a [baremetal application](https://github.com/chipsalliance/dromajo/blob/master/doc/setup.md#small-baremetal-program) or
a full application using Linux.

For this example, since Dromajo does not support system call
emulation, the workload dhrystone is instrumented and run inside
Linux.

The Dhrystone included with Olympia (`dry.c`) is instrumented with two
extra lines of code to start/stop tracing as well as an early out of
the main loop.

``` main() {
    // ... init code

    START_TRACE;
    // Dhrystone benchmark code
    STOP_TRACE;

    // ... teardown
   }

```
The `START_TRACE` and `STOP_TRACE` markers are defined in `dromajo/include/trace_markers.h` included in the dromajo patch.

[Follow the directions](https://github.com/chipsalliance/dromajo/blob/master/doc/setup.md#linux-with-buildroot)
on booting Linux with a buildroot on Dromajo's page.

Build `dry.c` that's included with olympia.  Remember that dry.c is a _shell script_ that builds itself.

```
cd dromajo/run
CC=riscv64-linux-gcc CFLAGS="-I../include -DTIME" sh ../../dry.c
```
Copy `dry2` into the buildroot and rebuild it:
```
cp dry2 ./buildroot-2020.05.1/output/target/sbin/
make -C buildroot-2020.05.1
```
Don't forget to recopy the images.

Run Dromajo with the flag `--stf_trace`, log in, and run Dhrystone:
```
../build/dromajo --stf_trace dry.zstf boot.cfg


... OpenSBI boot


Welcome to Dromajo Buildroot
buildroot login: root
Password: root
# dry2 10000

```
After Dhrystone has finished, Ctrl-C out of Dromajo and observe the generated trace:
```
% ls dry.zstf
dry.zstf
```
Now, run that trace on olympia:
```
cd olympia/build
./olympia ../traces/stf_trace_gen/dromajo/run/dry.zstf
Running...
olympia: STF file input detected
Retired 1000000 instructions in 875797 cycles.  Period IPC: 1.14182 overall IPC: 1.14182
Retired 2000000 instructions in 1751570 cycles.  Period IPC: 1.14185 overall IPC: 1.14183
Retired 3000000 instructions in 2627287 cycles.  Period IPC: 1.14192 overall IPC: 1.14186
Retired 4000000 instructions in 3503057 cycles.  Period IPC: 1.14185 overall IPC: 1.14186
Retired 5000000 instructions in 4378790 cycles.  Period IPC: 1.1419 overall IPC: 1.14187
Retired 6000000 instructions in 5254496 cycles.  Period IPC: 1.14194 overall IPC: 1.14188
Retired 7000000 instructions in 6130274 cycles.  Period IPC: 1.14184 overall IPC: 1.14187
...
```