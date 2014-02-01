BeagleG
=======

Step-motor controller (and eventually 3D printer controller) using the PRU
capability of the Beaglebone Black to create precisely timed stepper-pulses for
acceleration and travel.

See example here: http://www.youtube.com/watch?v=hIEY9077D64

The motor-interface API allows to enqueue
step-{count, {start,travel,end}-frequency}
of 8 steppers that are controlled in a coordinated move (G1), with real-time
controlled steps at rates that can go beyond 500kHz.
So: sufficient even for advanced step motors and drivers :)

The [acceleration - travel - deceleration] motion profile is entirely
created within the PRU from parameters sent by the host CPU (i.e. BeagleBone ARM)
via a ring-buffer.
The host CPU prepares the data, such as parsing the G-Code and doing travel
planning, while all the real-time critical parts are
done in the PRU. The host program needs less than 1% CPU-time processing a
typical G-Code file.

The `send-gcode` program is parsing G-Code, extracting axes moves and
enqueues them to the realtime unit.

## APIs
The functionality is encapsulated in independently usable APIs.

   - `motor-interface.h` : C-API to enqueue motor moves, that are
      executed in the PRU.
   - `gcode-parser.h` : C-API that parses G-Code and calls callbacks, while
      taking care of many internals, e.g. it automatically translates everything
      into metric, absolute coordinates.
   - `determine-print-stats.h`: C-API to determine some basic stats about
      a G-Code file; it processes the entire file and determines estimated
      print time, filament used etc. Implementation is mostly an example using
      gcode-parser.h.
      Used in the `gcode-print-stats` binary.
   - `gcode-machine-control.h` : highlevel C-API to control a machine via
      G-Code: it reads G-Code and emits the necessary machine commands.
      Used in the `send-gcode` binary.

## Machine control binary
To control a machine with G-Code, use `send-gcode`. This interpreter either
takes a filename or a TCP port to listen on.

    Usage: ./send-gcode [options] [<gcode-filename>]
    Options:
      --steps-mm <axis-steps>   : steps/mm, comma separated (Default 160,160,160,40,0, ...).
      --max-feedrate <rate> (-m): Max. feedrate per axis (mm/s), comma separated (Default: 200,200,90,0, ...).
      --accel <accel>       (-a): Acceleration per axis (mm/s^2), comma separated (Default 4000,4000,1000,10000,0, ...).
      --motor-output-mapping    : Motor index (=string pos) mapped to which axis.
                                  Axis letter or '_' for no mapping. (Default: 'XYZEABC')
      --port <port>         (-p): Listen on this TCP port.
      --bind-addr <bind-ip> (-b): Bind to this IP (Default: 0.0.0.0).
      -f <factor>               : Print speed factor (Default 1.0).
      -n                        : Dryrun; don't send to motors (Default: off).
      -P                        : Verbose: Print motor commands (Default: off).
      -S                        : Synchronous: don't queue (Default: off).
      -R                        : Repeat file forever.
All comma separated axis values are in the sequence X,Y,Z,E,A,B,C
You can either specify --port <port> to listen for commands or give a filename

The G-Code understands axes X, Y, Z, E, A, B, C and maps them to stepper [0..6],
which can be changed with the `--motor-output-mapping` flag.

### Configuration tip
For a particular machine, you might have some settings you always want to
use, and maybe add some comments. So create a file that contains all the
command line options

    $ cat type-a.config
    # Configuration for Type-A machine series 1, Motors @ 28V
    --steps-mm 75.075,150.14,800   # x has half the steps than y
    --max-feedrate 900,900,90
    --accel 18000,8000,1500
    --motor-output-mapping XZY

Now, you can invoke `send-gcode` like this

    sudo ./send-gcode $(sed 's/#.*//g' type-a.config) --port 4444

or, simpler, if you don't have any comments:

    sudo ./send-gcode $(cat type-a.config) --port 4444

The `sed` command dumps the configuration, but removes the comment characters.

### Examples

    sudo ./send-gcode -f 10 -m 1000 -R myfile.gcode

Output the file `myfile.gcode` in 10x the original speed, with a feedrate
capped at 1000mm/s. Repeat this file forever (say you want to stress-test).

    sudo ./send-gcode --port 4444

Listen on TCP port 4444 for incoming connections and execute G-Codes over this
line. So you could use `telnet beaglebone-hostname 4444` to have an interactive
session or send a file with `socat`:

     cat myfile.gcode | socat -t5 - TCP4:beaglebone-hostname:4444

Use `socat`, don't use the ancient `nc` (netcat) - its buffering seems to be
broken so that it can get stuck. With `socat`, it should be possible to connect
to a pseudo-terminal in case your printer-software only talks to a terminal
(haven't tried that yet, please let me know if it works).

Note, there can only be one open TCP connection at any given time.

## G-Code stats binary
There is a binary `gcode-print-stats` to extract information from the G-Code
file e.g. estimated print-time, Object height (=maximum Z-axis), filament
length.

    Usage: ./gcode-print-stats [options] <gcode-file> [<gcode-file> ..]
    Options:
            -m <max-feedrate> : Maximum feedrate in mm/s
            -f <factor>       : Speedup-factor for print
    Use filename '-' for stdin.

The output is in column form, so you can use standard tools to process them.
For instance, from a bunch of gcode files, find the one that takes the longest
time

    ./gcode-print-stats *.gcode | sort -k2 -n


## Pinout

These are the GPIO bits associated with the motor outputs. The actual physical
pins are all over the place on the Beaglebone Black extension headers P8 and P9,
see table below.

Before we can use all pins, we need to tell the Beaglebone Black pin multiplexer
which we're going to use for GPIO. For that, we need to install a device
tree overlay. Just run the script beagleg-cape-pinmux.sh as root

    sudo ./beagleg-cape-pinmux.sh

Now, all pins are mapped to be used by beagleg. This is the pinout

    Axis G-Code name  |  X     Y     Z     E     A      B     C   <unassigned>
    Step     : GPIO-0 |  2,    3,    4,    5,    7,    14,   15,   20
           BBB Header |P9-22 P9-21 P9-18 P9-17 P9-42A P9-26 P9-24 P9-41A
                      |
    Direction: GPIO-1 | 12,   13,   14,   15,    16,    17,  18,   19
           BBB Header |P8-12 P8-11 P8-16 P8-15 P9-15 P9-23  P9-14 P9-16

Motor enable for all motors is on `GPIO-1`, bit 28, P9-12
(The mapping right now was done because these are consecutive GPIO pins that
can be used, but the mapping to P9-42A (P11-22) and P9-41A (P11-21) should
probably move to an unambiguated pin)

For your electrical interface: note this is 3.3V level (and assume not more
than ~4mA). The RAMPS driver board for instance only works if you power the
5V input with 3.3V, so that the Pololu inputs detect the logic level properly.

Here is my experimental manual cape
![Manual Cape][manual-cape]

This allows to plug in a RAMPS driver. At the middle/bottom of the test board
you see a headpone connector: many of the early experiments didn't have yet a
stepper motor installed, but just listening to the step-frequency.

## Build
The Makefile is assuming that you build this either on the Beaglebone Black
directly, or using a cross compiler (see Makefile).

You need to have checked out https://github.com/beagleboard/am335x_pru_package
which provides the pasm PRU assembler and the library to push this code to the
PRU.

    # Check out and build am335 package
    git clone git@github.com:beagleboard/am335x_pru_package.git
    cd am335x_pru_package/
    cd pru_sw/utils/pasm_source ; ./linuxbuild ; cd -
    CROSS_COMPILE="" make -C pru_sw/app_loader/interface/
    cd ..

    # Check out BeagleG and build
    git clone git@github.com:hzeller/beagleg.git
    cd beagleg
    make

If you run into compile problems, make sure to have both, am335x_pru_package and
beagleg up-to-date from git; looks like there were some recent changes in the
am335x_pru_package.
    
## License
BeagleG is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

## TODO
   - Read end-switches
   - Do planning: no need to decelerate fully if we're going on an (almost)
     straight line between line segments.
   - Needed for full 3D printer solution: add PWM for heaters.
   - Fast pause without waiting for queues to empty, but still be able to
     recover exact last position. That way pause/resume is possible.
   - ...

[manual-cape]: https://github.com/hzeller/beagleg/raw/master/img/manual-ramps-cape.jpg
