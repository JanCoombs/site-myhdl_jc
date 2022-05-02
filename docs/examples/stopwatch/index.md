---
title:   StopWatch 
layout:  wide_article
content: []
summary.md: |
    * writing a unit test before the implementation
    * using `py.test` as a unit test framework 
    * coding a hierarchical design
    * coding for ROM inference
    * synthesis of the output Verilog code
---

Introduction
============

On this page, we will present a stopwatch design. It is similar to the design
in the [Xilinx ISE tutorial](http://www.xilinx.com/support/techsup/tutorials/tutorials7.htm). We
will tackle it "the MyHDL way" and take it from spec to implementation.

This is an extensive example, and we will use it to present all aspects of a
MyHDL-based design flow. It's also a relatively advanced. If you have
difficulties understanding the material on this page, consider reading the
first chapters of the
[manual](http://www.jandecaluwe.com/Tools/MyHDL/manual/MyHDL.html) or the
earlier examples in this Cookbook first.

Specification
=============

Compared to the design in the Xilinx ISE tutorial, our design is somewhat
simplified. The intention is not to avoid complexity, but merely to make the
code and the explanations better fit on a single web page. In particular, our
stopwatch will only have three digits: two digits for the seconds, and one for
the tenths of a second. Also, we will not consider clock generation issues and
simply assume that a 10Hz clock is available.

The interface of the stopwatch design looks as follows:

```python
@block
def StopWatch(tens_led, ones_led, tenths_led, startstop, reset, clock):

    """ 3 digit stopwatch with seconds and tenths of a second.

    tens_led: 7 segment led for most significant digit of the seconds
    ones_led: 7 segment led for least significant digit of the seconds
    tenths_led: 7 segment led for tenths of a second
    startstop: input that starts or stops the stopwatch on a posedge
    reset: reset input
    clock: 10Hz clock input

    """
```

Architecture
============

A stopwatch system is naturally partitioned as follows:

* a subsystem that counts time, expressed as digits in bcd (binary coded decimal) code
* a subsystem that displays the count, by converting each bcd digit to a 7 segment led display

A natural partitioning often works best, and that's how we will approach the
design. We will first design a  time counter and then a bcd to led convertor.

Time counter design
===================

Approach
--------

One of the goals of the MyHDL project is to promote the use of modern software
development techniques for hardware design. One such technique is the concept
of unit testing, a cornerstone of *extreme programming* (XP).

Unit testing means writing a dedicated test for each building block of a
design, and aggregating all tests in a regression test suite using a unit test
framework. Moreover, the XP idea is to *write the unit test first*, before the
actual implementation. This makes sure that the test writer concentrates on all
aspects of the high level specification, without being influenced by lower
level implementation details.

At the start of an implementation, the existing unit test will fail, and it
will continue to do so until a valid implementation is achieved. The unit test
thus serves as a metric for completion. Moreover, to see the unit test fail on
incomplete or invalid designs enhances the confidence in the test quality
itself. This is of crucial importance when making design changes later on.

Unit test
---------

To write a unit test for building block, we need two things: the specification
and the interface. The specification was described in previous sections. The
interface of the time counter looks as follows:

```python
from myhdl import *

def TimeCount(tens, ones, tenths, startstop, reset, clock):

    """ 3 digit time counter in seconds and tenths of a second.

    tens: most significant digit of the seconds
    ones: least significant digit of the seconds
    tenths: tenths of a second
    startstop: input that starts or stops the counter on a posedge
    reset: reset input
    clock: 10Hz clock input

    """
```

The actual implementation is left open for now. We will first write the test, using the interface.

The following code is the unit test for the time counter subsystem in file `test_TimeCount.py`:

```python
from random import randrange

from myhdl import *

from TimeCount import TimeCount

LOW, HIGH = bool(0), bool(1)

MAX_COUNT = 6 * 10 * 10
PERIOD = 10

@block
def bench():

    """ Unit test for time counter. """

    tens, ones, tenths = [Signal(intbv(0)[4:]) for i in range(3)]
    startstop, reset, clock = [Signal(LOW) for i in range(3)]

    dut = TimeCount(tens, ones, tenths, startstop, reset, clock)
    
    count = Signal(0)
    counting = Signal(False)

    @always(delay(PERIOD//2))
    def clkgen():
        clock.next = not clock

    @always(startstop.posedge, reset.posedge)
    def action():
        if reset:
            counting.next = False
            count.next = 0
        else:
            counting.next = not counting      

    @always(clock.posedge)
    def counter():
        if counting:
            count.next = (count + 1) % MAX_COUNT
            
    @always(clock.negedge)
    def monitor():
        assert ((tens*100) + (ones*10) + tenths) == count

    @instance
    def stimulus():
        for maxInterval in (100*PERIOD, 2*MAX_COUNT*PERIOD):
            for sig in (reset, startstop,
                        reset, startstop, startstop,
                        reset, startstop, startstop, startstop,
                        reset, startstop, reset, startstop, startstop, startstop):
               yield delay(randrange(10*PERIOD, maxInterval))
               yield clock.negedge # sync to avoid race condition
               sig.next = HIGH
               yield delay(100)
               sig.next = LOW
        raise StopSimulation
  
    return dut, clkgen, action, counter, monitor, stimulus


def test_bench():
    sim = Simulation(bench())
    sim.run()
```

`dut` is the design under test. `clkgen` is a clock generator. `action` defines
the stopwatch state, based on a rising edge on either of the input signals
`startstop` or `reset`. `counter` maintains the expected time count. `monitor`
is the actual test: it asserts that the actual time count from the design
equals the expected time count. Finally, `stimulus` defines a number of test
cases for the stopwatch. Note that it has an inner `for` loop over signals, as
a concise way to define test patterns. This is straightforward in Python. But
think for a moment on how you would do it in Verilog or VHDL.

Also in `stimulus`, note the `yield clock.negedge` statement. This statement
synchronizes signal changes with the falling clock edge. This is needed to
avoid race conditions when signals change "simultaneously" with the rising
clock edge. This is commonly done in digital tests. As you can expect, this
statement was not present in the first version of the test: it was added after
the test was run against the implementation and found to fail occasionally,
even when the implementation was believed to be correct. This shows that in
practice there may be a good reason why a test needs to be adapted to get
everything working. But it in any case it is better to start with a "general"
unit test that is not influenced by an implementation.

Our unit test is now ready to run. We could actually run it directly against an
implementation. However, we will use it via the unit testing framework
`py.test` instead. The framework provides the following functionality:

* it redefines the Python `assert` statement for extensive error reporting
* it looks up and runs each method whose name starts with "test_"
* it looks up test modules by searching for modules whose name starts with "test_"

There's a lot more to say about `py.test` and you are probably also curious
where to get it from. You can find that info further on this page, in the
section [More about py.test](cookbook:stopwatch#more_about_py.test).

Design
------
The following is an implementation of the time counter, in file `TimeCount.py`:

```python
from myhdl import *

@block
def TimeCount(tens, ones, tenths, startstop, reset, clock):

    """ 3 digit time counter in seconds and tenths of a second.

    tens: most significant digit of the seconds
    ones: least significant digit of the seconds
    tenths: tenths of a second
    startstop: input that starts or stops the counter on a posedge
    reset: reset input
    clock: 10kHz clock input

    """

    @instance
    def logic():
        seen = False
        counting = False

        while True:
            yield clock.posedge, reset.posedge

            if reset:
                tens.next = 0
                ones.next = 0
                tenths.next = 0
                seen = False
                counting = False

            else:
                if startstop and not seen:
                    seen = True
                    counting = not counting
                elif not startstop:
                    seen = False

                if counting:
                    if tenths == 9:
                        tenths.next = 0
                        if ones == 9:
                            ones.next = 0
                            if tens == 5:
                                tens.next = 0
                            else:
                                tens.next = tens + 1
                        else:
                            ones.next = ones + 1
                    else:
                        tenths.next = tenths + 1

    return logic

```

`py.test` confirms that this is a valid implementation:

```
$ py.test test_TimeCount.py
============================= test session starts ==============================
platform linux -- Python 3.7.3, pytest-7.1.1, pluggy-1.0.0
rootdir: /home/jan/tmp/projects/myhdl/myhdl.org/work/testing-examples-code/stopwatch
collected 1 item                                                               

test_TimeCount.py .                                                      [100%]

============================== 1 passed in 0.68s ===============================
```

bcd to led convertor design
===========================

Approach
--------

For the design of the bcd to led convertor , we will follow a similar approach
as before. We will write a unit test first, and then use it to complete the
design.

We first put the encoding data in a separate module, `seven_segment.py`, to
make it reusable. The appropriate data structure for the encoding is a
dictionary:

```python
# 7 segment encoding
#      0
#     ---  
#  5 |   | 1
#     ---   <- 6
#  4 |   | 2
#     ---
#      3
   
encoding =  {0: "1000000",
             1: "1111001",
             2: "0100100",
             3: "0110000",
             4: "0011001", 
             5: "0010010",
             6: "0000010",
             7: "1111000",
             8: "0000000",
             9: "0010000"
            }
```

Unit test
---------

This is the unit test, in `test_bcd2led.py`:

```python
from random import randrange
import seven_segment
from myhdl import *
from bcd2led import bcd2led


PERIOD = 10

@block
def bench():
    
      led = Signal(intbv(0)[7:])
      bcd = Signal(intbv(0)[4:])
      clock = Signal(bool(0))
    
      dut = bcd2led(led, bcd, clock)

      @always(delay(PERIOD//2))
      def clkgen():
            clock.next = not clock

      @instance
      def check():
            for i in range(100):
                  bcd.next = randrange(10)
                  yield clock.posedge
                  yield clock.negedge
                  expected = int(seven_segment.encoding[int(bcd)], 2)
                 assert led == expected
           raise StopSimulation

      return dut, clkgen, check


def test_bench():
      sim = Simulation(bench())
      sim.run()
```

This test asserts that the led output from the design matches the appropriate
encoding for a digit.

Design
------

Here is an implementation, in `bcd2led.py`:

```python
import seven_segment

from myhdl import *

code = [None] * 10
for key, val in seven_segment.encoding.items():
    if 0 <= key <= 9:
        code[key] = int(val, 2)
code = tuple(code)

@block
def bcd2led(led, bcd, clock):

    """ bcd to seven segment led convertor.

    led: seven segment led output
    bcd: bcd input
    clock: clock input

    """

    @always(clock.posedge)
    def logic():
        led.next = code[int(bcd)]

    return logic
```

Note how we derive the tuple `code` from the `encoding` dictionary. We need a
tuple because that's the data structure that the Verilog convertor supports.
It maps tuple indexing to a case statement to support ROM inferencing by
synthesis tools.

When we run `py.test`, we get the following output:

```
$ py.test
============================= test process starts ==============================
platform linux -- Python 3.7.3, pytest-7.1.1, pluggy-1.0.0
rootdir: /.../testing-examples-code/stopwatch
collected 2 items                                                              

test_TimeCount.py .                                                      [ 50%]
test_bcd2led.py .                                                        [100%]

============================== 2 passed in 0.53s ===============================
```

Note that when run with no arguments, `py.test` finds and runs all test
modules. This is done recursively through all subdirectories, making it
straightforward to run a full regression test suite.

Top level design
================

The top-level design in `StopWatch.py` is just an assembly of the previously
designed modules:

```python
from myhdl import *

from TimeCount import TimeCount
from bcd2led import bcd2led

@block
def StopWatch(tens_led, ones_led, tenths_led, startstop, reset, clock):

    """ 3 digit stopwatch with seconds and tenths of a second.
    
    tens_led: 7 segment led for most significant digit of the seconds
    ones_led: 7 segment led for least significant digit of the seconds
    tenths_led: 7 segment led for tenths of a second
    startstop: input that starts or stops the stopwatch on a posedge
    reset: reset input
    clock: 10Hz clock input

    """

    tens, ones, tenths = [Signal(intbv(0)[4:]) for i in range(3)]

    timecount_inst = TimeCount(tens, ones, tenths, startstop, reset, clock)
    bcd2led_tens = bcd2led(tens_led, tens, clock)
    bcd2led_ones = bcd2led(ones_led, ones, clock)
    bcd2led_tenths = bcd2led(tenths_led, tenths, clock)

    return timecount_inst, bcd2led_tens, bcd2led_ones, bcd2led_tenths
```

Implementation
==============

Automatic conversion to Verilog or VHDL
---------------------------------------

To go to an implementation, we first convert the design to Verilog
automatically, using MyHDL's `toVerilog` function:

```python
def convert():
    tens_led, ones_led, tenths_led = [Signal(intbv(0)[7:]) for i in range(3)]
    startstop, reset, clock = [Signal(bool(0)) for i in range(3)]
    convInst = StopWatch(tens_led, ones_led, tenths_led, startstop, reset, clock)
    convInst.convert(hdl='Verilog')
    convInst.convert(hdl='VHDL')

convert()
```

The resulting Verilog code is included in full:

```verilog
module StopWatch (
    tens_led,
    ones_led,
    tenths_led,
    startstop,
    reset,
    clock
);

output [6:0] tens_led;
reg [6:0] tens_led;
output [6:0] ones_led;
reg [6:0] ones_led;
output [6:0] tenths_led;
reg [6:0] tenths_led;
input startstop;
input reset;
input clock;

reg [3:0] ones;
reg [3:0] tens;
reg [3:0] tenths;


always @(posedge clock, posedge reset) begin: STOPWATCH_TIMECOUNT0_LOGIC
    reg seen;
    reg counting;
    if (reset) begin
        tens <= 0;
        ones <= 0;
        tenths <= 0;
        seen = 1'b0;
        counting = 1'b0;
    end
    else begin
        if ((startstop && (!seen))) begin
            seen = 1'b1;
            counting = (!counting);
        end
        else if ((!startstop)) begin
            seen = 1'b0;
        end
        if (counting) begin
            if ((tenths == 9)) begin
                tenths <= 0;
                if ((ones == 9)) begin
                    ones <= 0;
                    if ((tens == 5)) begin
                        tens <= 0;
                    end
                    else begin
                        tens <= (tens + 1);
                    end
                end
                else begin
                    ones <= (ones + 1);
                end
            end
            else begin
                tenths <= (tenths + 1);
            end
        end
    end
end

always @(posedge clock) begin: STOPWATCH_BCD2LED0_LOGIC
    case (tens)
        0: tens_led <= 64;
        1: tens_led <= 121;
        2: tens_led <= 36;
        3: tens_led <= 48;
        4: tens_led <= 25;
        5: tens_led <= 18;
        6: tens_led <= 2;
        7: tens_led <= 120;
        8: tens_led <= 0;
        default: tens_led <= 16;
    endcase
end

always @(posedge clock) begin: STOPWATCH_BCD2LED1_LOGIC
    case (ones)
        0: ones_led <= 64;
        1: ones_led <= 121;
        2: ones_led <= 36;
        3: ones_led <= 48;
        4: ones_led <= 25;
        5: ones_led <= 18;
        6: ones_led <= 2;
        7: ones_led <= 120;
        8: ones_led <= 0;
        default: ones_led <= 16;
    endcase
end

always @(posedge clock) begin: STOPWATCH_BCD2LED2_LOGIC
    case (tenths)
        0: tenths_led <= 64;
        1: tenths_led <= 121;
        2: tenths_led <= 36;
        3: tenths_led <= 48;
        4: tenths_led <= 25;
        5: tenths_led <= 18;
        6: tenths_led <= 2;
        7: tenths_led <= 120;
        8: tenths_led <= 0;
        default: tenths_led <= 16;
    endcase
end

endmodule
```

Note how the Verilog convertor expands the hierarchical design into a "flat net
list of always blocks". The Verilog ouput is really an intermediate step
towards an implementation. The whole design is flat and contained in a single
file, which may make it easier to hand it off to back-end synthesis and
implementation tools.

Note also how the convertor expands tuple indexing in MyHDL into a case
statement in Verilog.

Synthesis
---------

We will synthesize the design with Xilinx ISE 8.1. We first create a project in
the ISE environment, add the source of the Verilog file to it, and we are ready
to go.

The following is extracted from the synthesis report. It shows how the
synthesis tool recognizes higher-level functions such as ROMs and counters:

```
=========================================================================
*                           HDL Synthesis                               *
=========================================================================

Synthesizing Unit <StopWatch>.
    Related source file is "/home/jand/dev/myhdl/example/cookbook/stopwatch/StopWatch.v".
    Found 16x7-bit ROM for signal <$n0007> created at line 68.
    Found 16x7-bit ROM for signal <$n0008> created at line 84.
    Found 16x7-bit ROM for signal <$n0009> created at line 100.
    Found 7-bit register for signal <tenths_led>.
    Found 7-bit register for signal <ones_led>.
    Found 7-bit register for signal <tens_led>.
    Found 1-bit register for signal <_StopWatch_timecount_inst_logic/counting>.
    Found 1-bit register for signal <_StopWatch_timecount_inst_logic/seen>.
    Found 4-bit up counter for signal <ones>.
    Found 4-bit up counter for signal <tens>.
    Found 4-bit up counter for signal <tenths>.
    Summary:
	inferred   3 ROM(s).
	inferred   3 Counter(s).
	inferred  23 D-type flip-flop(s).
Unit <StopWatch> synthesized.
```

How these blocks are actually implemented depends on the target technology and
the capabilities of the synthesis tool.

You can review the full FPGA synthesis report [here](synthesis.txt).

FPGA implementation
-------------------

The FPGA implementation report can be reviewed [here](map.txt).

CPLD implementation
-------------------

The same design was also targetted to a CPLD technology. The detailed report
can be viewed [here](cpldfit.txt).

More about py.test
==================

To verify the stopwatch design, we have been using `py.test`. However, this is
not the only unit testing framework available for Python. In fact, the standard
unit testing framework that comes with Python is the `unittest` module. The
`unittest` framework is presented in the MyHDL manual, and is used to verify
MyHDL itself. On the other hand, `py.test` is not part of the standard Python
library currently. Why then did we use `py.test` in this case?

The reason is that I believe that `py.test` will be a better option in the
future. As demonstrated on this page, `py.test` is non-intrusive. The only
thing we need to do for basic usage is to obey some simple naming conventions
and to use the `assert` statement for testing - things we might want to do
without a testing framework anyway.  In contrast, `unittest` requires us to
wrap our tests into dedicated subclasses and to use special test methods. This
can be especially awkward with MyHDL, because MyHDL hardware is typically
described using top-level and embedded functions, not classes and methods.

In short, it is much easier to develop unit tests with `py.test` than it is
with `unittest`, in particular in the case of MyHDL code. However, `py.test`
also has its disadvantages:

* As `py.test` is not part of the standard Python library, it has to be
installed separately.
* `py.test` is currently not distributed in a convential way such as a tar
file. It is part of the `py.lib` library that has to be checked out from a
subversion repository. This requires the installation of a subversion client.
* The use of the `assert` statement for unit testing is controversial in
Python. The `assert` statement is originally intended for programmer usage,
to make programs safer. However, in my opinion the use of `assert` for
testing is natural and warranted.
* `py.test` uses a lot of "magic" behind the scenes to modify Python's behavior
for its purposes, such as extensive error reporting.

However, I believe that the benefits are far more important than the
disadvantages. Moreover, some disadvantages may disappear over time.
Consequently, I plan to promote `py.test` as the unit testing framework of
choice for MyHDL in the future.

More info on the usage and installation of `py.test` can be found
[here](http://pytest.org/latest/getting-started.html).
