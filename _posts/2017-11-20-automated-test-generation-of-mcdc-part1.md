---
layout: post
title: "Automated Test Generation of MC/DC (part 1)"
categories: ATG, DO-178C, simulink, lustre
author: "Roberto Bruttomesso"
---

<style>
.tablelines table, .tablelines td, .tablelines th {
    border: 1px solid black;
}
</style>

## Summary

In this post (and its companion [part 2][part2]) we show how to automatically generate
tests from a Simulink or Lustre specification using [Intrepyd][intrepyd]. 
The tests will provide full coverage using the Modified Condition/Decision Coverage (MD/DC
from now on), which is of paramount importance in the avionics domain.
The focus of this post is on the description of the MC/DC criterion.

### Highlights

- QA for avionics critical software (DO-178C)
- The MC/DC coverage metric
- Masking and Unique-cause MC/DC

### Keywords

ATG, MC/DC, DO-178C, simulink, lustre, model-checking

## Quality assurance for avionics critical software

Avionics software is perhaps the most representative example of
**critical** software, as bugs can result into catastrophic events
such as the loss of hundreds of human lives. Moreover airborne
software cannot be patched or fixed, nor it can easily undergo
major adjustments. It is no surprise that the development process
is regulated by a standard entitled "Software Considerations in
Airborne Systems and Equipment Certification", aka [DO-178C][do178c]
(formerly DO-178B). 

This standard defines the software development
process to be used, starting from the requirements down to the
verification. In particular here "verification" is not intended as property
verification as can be done via theorem proving or model checking,
but it is rather meant a heavy and pedantic testing process. A good part
of this process consists in providing tests showing 100% structural
coverage of Boolean expressions using the MC/DC metric (at least
for "Level A" software, the category that includes software whose
failure is considered "catastrophic").
These tests are to be presented in tables so that they could be inspected
(and hopefully approved) by the regulating agency (e.g., the FAA).
This is the context where our test generation procedure is supposed to operate.

## What is MC/DC anyway ?

In software it is common to find branch statements like the following
{% highlight c %}
if ((x && y) || (!x && z)) {
    ...
}
{% endhighlight %}
The Boolean expression \\((x \wedge y) \vee (\neg x \wedge z)\\) is called **decision**, while
the first occurrence of \\(x\\), \\(y\\), the second occurrence of \\(x\\), and \\(z\\)
are called **conditions**. Notice that the number of conditions is greater
or equal to the number of Boolean variables in the expression. In the following
- \\(c_1\\) is "the first occurrence of \\(x\\)"
- \\(c_2\\) is "the occurrence of \\(y\\)"
- \\(c_3\\) is "the second occurrence of \\(x\\)"
- \\(c_4\\) is "the occurrence of \\(z\\)"

Let's forget about the branch statement, and focus
instead on decisions and conditions, which are the only ingredients needed to
talk about and compute tests for MC/DC.
A test for the above decision, call it \\(D\\), is an assignment
of a truth value to \\(x\\) \\(y\\), and \\(z\\). There are a total of eigth possible tests
(columns \\(x\\), \\(y\\), and \\(z\\) are the test inputs, while the 
\\(D\\) column is the expected value):

|           | \\(x\\) | \\(y\\) | \\(z\\) | \\(D\\) |
|   :---:   |:---:|:---:|:---:|:---:|
| \\(t_1\\) |  F  |  F  |  F  |  F  |
| \\(t_2\\) |  F  |  F  |  T  |  T  |
| \\(t_3\\) |  F  |  T  |  F  |  F  |
| \\(t_4\\) |  F  |  T  |  T  |  T  |
| \\(t_5\\) |  T  |  F  |  F  |  F  |
| \\(t_6\\) |  T  |  F  |  T  |  F  |
| \\(t_7\\) |  T  |  T  |  F  |  T  |
| \\(t_8\\) |  T  |  T  |  T  |  T  |

With a bit of notation we will write things like
- \\(t_1(x) = F\\)
- \\(t_3(y) = T\\)
- \\(t_1(x \wedge y) = t_1(x) \wedge t_1(y) = F\\)
- \\(t_7(D) = t_7((x \wedge y) \vee (\neg x \wedge z)) = \ldots = T\\)

meaning that we substitute the variables with their values in the test
and perform elementary simplifications like \\(T \wedge F = F\\).
Moreover we will write \\(D[c]\\) to indicate a decision where
condition \\(c\\) is replaced with \\(\neg c\\). For example
\\(D[c_3] = (x \wedge y) \vee (x \wedge z)\\);
recall that \\(c_3\\) is "the second occurrence of x", and that \\(\neg \neg x = x\\).

\\(\\{t_1 \ldots t_8\\}\\) represent the test suite with the maximum cardinality, 
it they perfectly characterize the behavior of the decision: change a single gate in the
Boolean expression, and at least one test will surely fail (unless
you are lucky enough to change the expression into an equivalent one,
but that would require changing more than one gate).
Unfortunately, for obvious practical reasons, one cannot afford to produce
a test suite with exponentially many tests in the number of variables of
the expression.

MC/DC aims to provide a trade-off between producing an effective test suite (a test
suite that catches a high number of bugs) and a compact test suite. In
particular MC/DC test suites contain at most as 2N tests where
N is the number of conditions in the expression (on average however
N+1 tests are usually enough).

MC/DC focuses on providing proof that
**each condition is relevant to determine the value of the decision**.
In particular for each condition \\(c\\) we need to provide a pair of
tests \\((t_1, t_2)\\), called **independence pair** such that:
1. the value of \\(D\\) in the two tests is different, i.e., \\(t_1(D) \not= t_2(D)\\)
1. the value of \\(c\\) in the two tests is different, i.e., \\(t_1(c) \not= t_2(c)\\)
1. \\(c\\) has influence on the outcome of \\(D\\) in \\(t_1\\), i.e., \\(t_1(D) \not= t_1(D[c])\\)
1. \\(c\\) has influence on the outcome of \\(D\\) in \\(t_2\\), i.e., \\(t_2(D) \not= t_2(D[c])\\)

For example, take \\(c_1\\), "the first occurrence of \\(x\\)". We show that \\((t_3, t_7)\\)
respects the four conditions above (recall that \\(D[c_1] = (\neg x \wedge y) \vee (\neg x \wedge z)\\)):
1. \\(t_3(D) = F\\) and \\(t_7(D) = T\\)
1. \\(t_3(c) = F\\) and \\(t_7(c) = T\\)
1. \\(t_3(D) = F\\) and \\(t_3(D[c_1]) = T\\)
1. \\(t_7(D) = T\\) and \\(t_7(D[c_1]) = F\\)

Similarly it is possible to show that \\((t_5, t_7)\\), \\((t_2, t_6)\\), and \\((t_2, t_3)\\)
are valid independence pairs for \\(c_2\\), \\(c_3\\)", and
\\(c_4\\) respectively. Therefore the test suite
\\(\\{t_2, t_3, t_5, t_6, t_7\\}\\) provides MC/DC for the given decision.

## Masking and Unique-cause MC/DC

So far we have described the most general form of MC/DC, also known as **Masking** MC/DC.
The other form of MC/DC in use in avionics is called **Unique-cause** MC/DC. In this
case we also need to find independence pairs, but they are subjected to more restrictive
constraints. In particular for each condition \\(c\\) we need to provide an independence
pair \\(t_1, t_2\\) such that:
1. the value of \\(D\\) in the two tests is different, i.e., \\(t_1(D) \not= t_2(D)\\)
1. the value of \\(c\\) in the two tests is different, i.e., \\(t_1(c) \not= t_2(c)\\)
1. the value of all other conditions does not change, i.e., if \\(c' \not= c\\) then \\(t_1(c') = t_2(c')\\)

It is easy to see that the last constraint cannot be satisfied when there is more than one
occurrence of the same variable in a decision, as in the example that we have used this far.
In fact two occurrences correspond to two different conditions, but it is impossible to
change the value of one occurrence while leaving the other unchanged. Such conditions
are called **strongly coupled** because their truth values are entangled.

In Masking MC/DC more than one condition may change in an independence pair, but
only one will contribute to the decision outcome, while the other one will have
to be **masked** (which justifies the name "masking"). Take for instance
the independence pair for \\(c_1\\), \\((t_2, t_3)\\):
in both tests \\(c_3\\), the second occurrence of \\(x\\), is masked out in the 
subformula \\((\neg x \wedge z)\\) by
the fact that \\(t_2(z) = t_3(z) = F\\).

### Another example

Consider the decision \\(D := (x \wedge y) \vee z\\). The following is a test suite
that provides Unique-cause MC/DC:

|           | \\(x\\) | \\(y\\) | \\(z\\) | \\(D\\) |
|   :---:   |:---:|:---:|:---:|:---:|
| \\(t_1\\) | **F** |   T   | **F** |  F  |
| \\(t_2\\) |   F   |   T   | **T** |  T  |
| \\(t_3\\) | **T** | **T** |   F   |  T  |
| \\(t_4\\) |   T   | **F** |   F   |  F  |

Each column contains two highligthed values: take the rows corresponding
to these values and you get an independence pair. For instance \\((t_1, t_2)\\)
is the independence pair for \\(z\\). In Masking MC/DC it is possible to
be more flexible, as in the following test suite:

|            | \\(x\\) | \\(y\\) | \\(z\\) | \\(D\\) |
|   :---:    |:---:|:---:|:---:|:---:|
| \\(t'_1\\) | **F** |   F   | **F** |  F  |
| \\(t_2\\)  |   F   |   T   | **T** |  T  |
| \\(t_3\\)  | **T** | **T** |   F   |  T  |
| \\(t_4\\)  |   T   | **F** |   F   |  F  |

\\((t'_1, t_2)\\) is an independence pair for \\(z\\) even though both
\\(z\\) and \\(y\\) change their value: however \\(y\\) is masked in
both tests by \\(x\\) (assigned to false), and it is therefore not
relevant in the decision outcome.

Among the two tables, the first one is to be preferred, as it is easier
to review (just need to check that only one condition changes): in the
second table one has to make sure, by looking at the expression,
that \\(y\\) is not relevant. This is time consuming, and more error prone.

## When is it not possible to find MC/DC ?

Not all Boolean expressions admit Consider the following decision

\\((x \wedge y) \vee (x \wedge y \wedge z)\\)

The following holds:
- if either \\(x\\) or \\(y\\) are false, then \\(z\\) is masked out
- if both \\(x\\) and \\(y\\) are true, then \\(z\\) is masked out

Therefore there is no way to show that \\(z\\) affects the outcome of the decision;
in fact \\(z\\) itself is **redundant** in the decision. In other words, the
decision could be simplified to a more compact form

\\((x \wedge y)\\)

from which it is clear that only \\(x\\) and \\(y\\) can affect the decision's outcome.
Decisions with redundant conditions are called **degenerate**.
Degenerate decisions are a sign that something is wrong in the software, and they
must be fixed, otherwise it will not be possible to reach 100% MC/DC coverage. 
In a sense MC/DC is a way to improve the quality of the code: only code
containing non-degenerate decisions can be fully covered with MC/DC.

## Masking vs Unique-cause MC/DC

Masking and Unique-cause MC/DC was extensively studied by Chilenski in a report
titled ["An Investigation of Three Forms of the Modified Condition Decision Coverage (MCDC) Criterion"][chilenski],
which we definitely recommend to the reader. Chilenski calls **Singular Boolean Expressions (SBE)**
those expressions that are not degenerate and that contain only one occurrence per variable,
and **Non-SBE** those that are neither degenerate nor SBEs (in other words, Non-SBEs contain
strongly coupled conditions).
Basically SBEs are the only expressions to which Unique-cause MC/DC could be applied to reach 100% coverage.

In his paper he observes that, considering all possible Boolean expressions over 5 conditions,
SBEs are very small in number compared to Non-SBEs or degenerate: this would suggest that Unique-cause MC/DC 
has very little applicability. However in practice this is not true: by analyzing a large set of
avionics software, Chilenski observed that the majority of decisions are actually SBEs (only 70
out of 20256 decisions were Non-SBEs). Finally, he suggests a hybrid technique, 
Unique-cause+Masking MD/DC,
in which Unique-cause is used to find independence pairs for all conditions but those that are
strongly coupled, and use Masking only for strongly coupled conditions.

## Extension to predicates

So far we have considered expressions with Boolean variables, however avionics software
may contain also predicates such as \\((x < y)\\), \\((y = z)\\), etc. To find
an independence pair in this context requires to computate values for variables
that lie in some numerical domain (integers, floating-point, etc.), such that 
the truth values of the predicates fits the MC/DC constraints.
A further complication here is represented by **weakly coupled conditions**, i.e., conditions
that influence each other such as \\((x < y)\\) and \\((x = y)\\): the two conditions
cannot be both true at the same time (they can be both false though). In Intrepyd
we rely on SMT-solving to efficiently deal with these problems.

## Masking and Unique-cause in the standard

The previous version of the standard, the DO-178B, defines of MC/DC in a
way that is equivalent to the Unique-cause MC/DC given here (pay attention to the last sentence):

> Modified Condition/Decision Coverage - Every point of entry and exit in
> the program has been invoked at least once, every condition in a decision
> in the program has taken all possible outcomes at least once, every
> decision in the program has taken all possible outcomes at least once,
> and each condition in a decision has been shown to independently affect 
> that decision's outcome. A condition is shown to independently affect a
> decision's outcome by varying just that condition while holding fixed all
> other possible conditions. 

While Masking MC/DC was not explicitly mentioned it was often accepted as
a valid criterion to obtain structural coverage. The following extract is
from ["Position Paper CAST-6"][cast6] (unofficial guidelines written by experts):

> According to SC-190/WG-52 Frequently Asked Question #43 (ref. 5), structural coverage
> analysis complements requirements-based tests by:
> 1. Providing “evidence that the code structure was verified to the degree required for the
> applicable software level”;
> 2. Providing “a means to support demonstration of absence of unintended functions”; and
> 3. Establishing “the thoroughness of requirements-based testing”.
>
> Masking MC/DC, as well as unique-cause MC/DC, satisfies all three of these “intents”. 

and also

> Therefore, it is
> proposed that masking MC/DC be considered an acceptable method for meeting MC/DC by
> applicants striving to meet the objectives of DO-178B, Level A. 

With the introduction of DO-178C (released in 2011) the definition of MC/DC has been relaxed to officially allow
Masking as an accepted form of MC/DC:

> [...] A condition is shown to independently affect a
> decision's outcome by: (1) varying just that condition while holding fixed all
> other possible conditions, or (2) varying just that condition while holding
> fixed all other possible conditions that could affect the outcome. 

## Conclusion

In this post we have given a brief introduction to MC/DC. 
[In the second part of this post][part2] we will show how to generate MC/DC test cases
automatically from either Simulink or Lustre designs, using Intrepyd. We 
will also show how to evaluate the coverage of a test suite, and how
to extend a legacy test suite (that offers only partial coverage) to reach 100% MC/DC coverage.

Further readings in addition to the ones mentioned above can be found here:
- [A tutorial on MC/DC][tutorial];
- [What is a "Decision" in MC/DC][cast10].

[intrepyd]:        http://github.com/formalmethods/intrepyd
[do178c]:          https://en.wikipedia.org/wiki/DO-178C
[chilenski]:       http://www.tc.faa.gov/its/worldpac/techrpt/ar01-18.pdf
[cast6]:           https://www.faa.gov/aircraft/air_cert/design_approvals/air_software/cast/cast_papers/media/cast-6.pdf
[cast10]:          https://www.faa.gov/aircraft/air_cert/design_approvals/air_software/cast/cast_papers/media/cast-10.pdf
[tutorial]:        https://ntrs.nasa.gov/archive/nasa/casi.ntrs.nasa.gov/20010057789.pdf
[part2]:           {{ site.baseurl }}{% post_url 2017-12-20-automated-test-generation-of-mcdc-part2 %}