---
layout: post
title: "Automated Test Generation of MC/DC (part 2)"
categories: ATG
author: "Roberto Bruttomesso"
---

<style>
.tablelines table, .tablelines td, .tablelines th {
    border: 1px solid black;
}
</style>

In the previous post ([part 1][part1]) we have introduced the concept of
MC/DC which finds application in the verification of avionics and automotive
applications. In this post we show how to generate MC/DC tests starting from
a Simulink or a Lustre specification, using [Intrepyd][intrepyd]. 

### Highlights

- QA for avionics critical software (DO-178C)
- The MC/DC coverage metric
- Masking and Unique-cause MC/DC

### Keywords

ATG, MC/DC, DO-178C, simulink, lustre, model-checking

### Installing Intrepyd

As of version `0.5.13`, Intrepyd releases are pushed onto the [PYPI archive][intrepydpypi], and it can be installed by simply issuing the following command
```
pip install intrepyd
```
or
```
pip install intrepyd --upgrade
```
if you want to be working with the latest release. The version of Intrepyd used in this post is `0.6.4`.

### Examples used in this post

All the examples and scripts used in this post are available from a [public github repository](http://formalmethods.github.io/atgexperiments).

## MC/DC test generation for Simulink

### Single decision

Consider the following Simulink circuit

![Example 1]({{ "/imgs/atg_example1.png" }})

The circuit (``example1.slx``) is a simple combinational network with three inputs (A, B, C) and one output (O1).
We assume that the circuit represents the decision for which we need to generate tests. In
order to generate tests with Intrepyd we need to write the following Python script (``example1.py``):

{% highlight python %}
import utils
import intrepyd
import intrepyd.tools as it
import intrepyd.atg.mcdc as mcdc

def doMain():
    # Creates the Intrepyd context
    ctx = intrepyd.Context()
    # Translates the circuit into an equivalent one for Intrepyd
    circ = it.translate_simulink('example1.slx', 'float')
    # The following maps a decision with the nets corresponding to its conditions
    decisions = { 'example1/O1' : ['A', 'B', 'C'] }
    # dec2tables and dec2indpairs contain raw information about MC/DC tests
    dec2tables, dec2indpairs, _ = mcdc.compute_mcdc(ctx, circ.SimulinkCircuit, decisions, 0)
    # Translates dec2tables into a pandas dataframe
    dec2df = mcdc.get_tables_as_dataframe(dec2tables)
    # Prints data on screen
    print dec2df['example1/O1']
    print
    utils.pretty_print_ind_pair(dec2indpairs['example1/O1'])

if __name__ == "__main__":
    doMain()
{% endhighlight %}

The script can be executed from a shell, and it produces an expected output as follows

```
$ python example1.py
   A  B  C example1/O1
0  T  T  F           T
1  F  T  F           F
2  T  F  F           F
3  F  F  T           T
4  F  F  F           F

A: 0, 1
C: 3, 4
B: 0, 2
```

The output is composed of two parts: the truth table showing the tests (one per each line), and the independence pairs for each condition. It is easy to verify that the tests are correct, i.e., that each independence pair
respects the clauses for MC/DC (as explained [here][part1]). The generation in this case is not optimal as
four tests are enough for this decision (we have generated five instead): this is a consequence of the approximations behind the optimization algorithm, whose goal is to find a compromise between optimality
and execution-time.

### Multiple decisions at once

The following circuit (``example2.slx``) encodes three decisions at once that share the same conditions. Each
decision has been highlighted with a rectangle:

![Example 2]({{ "/imgs/atg_example2.png" }})

For this circuit we shall use a slightly more complex script, in order to capture
the three decisions at once:

{% highlight python %}
import utils
import intrepyd
import intrepyd.tools as it
import intrepyd.atg.mcdc as mcdc

def doMain():
    ctx = intrepyd.Context()
    circ = it.translate_simulink('example2.slx', 'float')
    decisions = { 
        'example2/O1' : ['A', 'B', 'C'],
        'example2/O2' : ['A', 'B', 'C', 'D'],
        'example2/O3' : ['C', 'D']
    }
    dec2tables, dec2indpairs, _ = mcdc.compute_mcdc(ctx, circ.SimulinkCircuit, decisions, 0)
    dec2df = mcdc.get_tables_as_dataframe(dec2tables)
    for decision, df in dec2df.iteritems():
        print 'MC/DC table for', decision
        print df
        print
        print 'Independence pairs for', decision
        utils.pretty_print_ind_pair(dec2indpairs[decision])
        print

if __name__ == "__main__":
    doMain()
{% endhighlight %}

Intrepyd will compute a heuristically minimal amount of tests that provides
MC/DC coverage for the three decisions. With the above script they are presented
as follows:

```
$ python example2.py
MC/DC table for example2/O1
   A  B  C example2/O1
0  F  F  T           T
1  F  F  F           F
2  T  T  F           T
3  F  T  F           F
4  T  F  F           F

Independence pairs for example2/O1
A: 2, 3
C: 0, 1
B: 2, 4

MC/DC table for example2/O3
   C  D example2/O3
0  T  F           F
1  T  T           T
2  F  T           F

Independence pairs for example2/O3
C: 2, 1
D: 0, 1

MC/DC table for example2/O2
   A  B  C  D example2/O2
0  F  F  T  F           F
1  T  F  T  F           T
2  F  T  T  F           T
3  F  T  F  F           F
4  T  F  F  T           T

Independence pairs for example2/O2
A: 0, 1
C: 3, 1
B: 0, 2
D: 3, 4
```

## MC/DC test generation for Lustre

Generation can be performed on Lustre circuits as well. Consider the following specification (``example3.lus``):

{% highlight lustre %}
node top(a, b, c : bool) returns (d : bool);
let
	d = a and (b or c);
tel
{% endhighlight %}

The following script can be used to generate MC/DC tests:

{% highlight python %}
import utils
import intrepyd
import intrepyd.tools as it
import intrepyd.atg.mcdc as mcdc

def doMain():
    ctx = intrepyd.Context()
    circ = it.translate_lustre('example3.lus', 'top', 'float32')
    decisions = { 'd' : ['a', 'b', 'c'] }
    dec2tables, dec2indpairs, _ = mcdc.compute_mcdc(ctx, circ.LustreCircuit, decisions, 0)
    dec2df = mcdc.get_tables_as_dataframe(dec2tables)
    print dec2df['d']
    print
    utils.pretty_print_ind_pair(dec2indpairs['d'])

if __name__ == "__main__":
    doMain()
{% endhighlight %}

which can be executed as follows:

```
$ python example3.py
   a  b  c  d
0  F  T  F  F
1  T  T  F  T
2  T  F  F  F
3  T  F  T  T

a: 0, 1
c: 2, 3
b: 2, 1
```

## Extension to arithmetic

The following example contains single precision floating-point inputs that are fed to relational
operators:

![Example 4]({{ "/imgs/atg_example4.png" }})

The conditions here are the two relational operators, and the Boolean input `C`. Tests can be
computed with the following script:

{% highlight python %}
import utils
import intrepyd
import intrepyd.tools as it
import intrepyd.atg.mcdc as mcdc

def doMain():
    ctx = intrepyd.Context()
    circ = it.translate_simulink('example4.slx', 'float')
    decisions = { 'example4/O1' : ['example4/leq1', 'example4/leq2', 'C'] }
    dec2tables, dec2indpairs, _ = mcdc.compute_mcdc(ctx, circ.SimulinkCircuit, decisions, 0)
    dec2df = mcdc.get_tables_as_dataframe(dec2tables)
    print dec2df['example4/O1']
    print
    utils.pretty_print_ind_pair(dec2indpairs['example4/O1'])

if __name__ == "__main__":
    doMain()
{% endhighlight %}

When executed it produces the following output:

```
$ python example4.py
  example4/leq1 example4/leq2  C example4/O1
0             F             F  T           F
1             F             T  T           T
2             F             F  F           F
3             T             F  F           T
4             F             T  F           F

C: 2, 3
example4/leq2: 4, 1
example4/leq1: 0, 1
```

Values for primary inputs are not shown in the table, however, they are computed
internally and available to be exported if necessary.

## Algorithmic and Implementation details

(Here we briefly sketch some algorithmic details, without the intention of being exhaustive. If
you are interested to learn more please feel free to drop me an email at ``roberto.bruttomesso@gmail.com``.)

Test generation is achieved by planting suitable [**trap properties**][gh1999] (targets in Intrepyd's terminology)
inside the circuit, in such a way that each counter-example (trace in Intrepyd's terminology)
to such properties can transformed into a test representing a row of the tables we have shown
in the above examples. The core algorithm implementation is an incremental extension using Z3 of
[a previous work][ofm2014] on test generation for Simulink.

Solving is performed by feeding all the targets to Intrepyd's optimizing Bounded Model Checker,
which ultimately relies on the optimization extension of [Z3][z3]. The process is iterative: at each
iteration the model checker returns with a trace, satisfying a heuristically high number of targets.
The targets are then removed from the list of targets to be solved, and the model checker is called
again. In the end, a heuristically minimal number of traces are produced, and the corresponding
tests are generated.

Generation is efficient thanks to three noteworthy features of Intrepyd: the fact that it can
solve multiple targets at once, the ability of working incrementally (solve, drop satisfied
targets, solve againg, etc.), and the ability of performing optimizing solving.

### Possible Extensions

The current ATG toolchain is a proof-of-concept for demonstrating the potential of Intrepyd
for test generation. The toolchain may be extended with relatively little effort to:

- work on decisions containing strongly coupled conditions;
- compute coverage of an existing test suite;
- recycle and extend an existing test suite by computing only the missing test cases;
- presentation of tests with values propagated to primary inputs.

If you are interested in any of the above features feel free to drop me an email at ``roberto.bruttomesso@gmail.com``.

## Conclusion

We have shown how automatic test generation of MC/DC could be performed using Intrepyd.
The proof-of-concept that we have put in place can be used on Simulink and Lustre specifications
to achieve a heuristically minimal generation of tests cases. We have chosen a simple
presentation for test results, however extensions and modifications are easy to achieve
thanks to the python-based framework of Intrepyd.

[intrepyd]:        http://github.com/formalmethods/intrepyd
[part1]:           {{ site.baseurl }}{% post_url 2017-11-20-automated-test-generation-of-mcdc-part1 %}
[z3]:              https://link.springer.com/chapter/10.1007%2F978-3-662-46681-0_14
[gh1999]:          https://link.springer.com/chapter/10.1007/3-540-48166-4_10
[ofm2014]:         http://ieeexplore.ieee.org/document/6847843/