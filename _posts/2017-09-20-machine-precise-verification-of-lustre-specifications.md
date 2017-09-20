---
layout: post
title: "Machine-precise Verification of Lustre specifications"
categories: lustre
author: "Roberto Bruttomesso"
---

## Summary

In this post we show how to use [Intrepyd][intrepyd] to translate and
verify [Lustre][lustre] specifications. The translation is such that the semantic
is **machine-precise**, i.e., we verify properties by taking into account
finite integers and floating-point representations for integers and real
variables, which is something that, to the best of our knowledge, is not
available even in commercial tools. We compare Intrepyd with an existing
tool, [Luke][luke], on the integer part (as Luke does not support reals),
and we report on solving some benchmarks with floating-point arithmetic
from Rockwell-Collins.

### Highlights

- Brief introduction to Lustre
- Use Intrepyd to verify Lustre specifications
- Experimental comparison w.r.t. other available software

### Keywords

Lustre, model-checking, SMT, Z3, BMC

## The Lustre specification language

[Lustre][lustre] is a synchronous dataflow language for programming reactive systems. The language is expressive enough for writing virtually any control software, but its semantic is well defined and conceptually easy to interpret and process automatically (as opposed to a traditional language like C, which is in comparison much harder to deal with). In a typical Model-Based flow, the executable code to be loaded on the actual system can be then automatically generated from the previously analyzed and certified Lustre specification, thus achieving a high degree of confidence on the behavior of the produced sofware. 

Lustre is used mostly in avionics and automotive domains, but it finds applications also in nuclear plants and industrial automation. Airbus, United Technologies, Ansaldo, Rockwell-Collins, Schneider Electric, Siemens are just a few examples of companies that employ Lustre for the specification of safety-critical systems. 

The following example shows a simple integer counter initialized to 0, and incremented by 1 at each clock tic, unless a reset signal comes to restore the value to 0.

{% highlight lustre %}
node counter (reset : bool) returns (c : int)
let
    c = 0 -> if reset then 0 else (pre c) + 1;
tel
{% endhighlight %}

For those who are more familiar with [Simulink][simulink], the above code is equivalent to the following model.

TODO: insert simulink model

TODO: supported data-types e pippone su semantica precisa

We shall not indulge more on the Lustre language, as there are already a number of resources available online in addition to the ones mentioned so far, such as [the PhD thesis of George E. Hagen][hagen], or [a Lustre course by Philipp Ruemmer][luke].

## Translating Lustre to Intrepyd's Python

[Intrepyd github repository][intrepyd]

{% highlight python %}
def main():
    print 'Hello World'
{% endhighlight %}

## Experiments

All

<div>
    <a href="https://plot.ly/~robertobruttomesso/34/?share_key=F5sMHzkyh5MyqWYyGbBEET" target="_blank" title="intrepyd-vs-luke-all" style="display: block; text-align: center;"><img src="https://plot.ly/~robertobruttomesso/34.png?share_key=F5sMHzkyh5MyqWYyGbBEET" alt="intrepyd-vs-luke-all" style="max-width: 100%;width: 700px;"  width="700" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="robertobruttomesso:34" sharekey-plotly="F5sMHzkyh5MyqWYyGbBEET" src="https://plot.ly/embed.js" async></script>
</div>

Invalid

<div>
    <a href="https://plot.ly/~robertobruttomesso/30/?share_key=FwHko4IEqw25dWedsImfWr" target="_blank" title="intrepyd-vs-luke-invalid" style="display: block; text-align: center;"><img src="https://plot.ly/~robertobruttomesso/30.png?share_key=FwHko4IEqw25dWedsImfWr" alt="intrepyd-vs-luke-invalid" style="max-width: 100%;width: 700px;"  width="700" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="robertobruttomesso:30" sharekey-plotly="FwHko4IEqw25dWedsImfWr" src="https://plot.ly/embed.js" async></script>
</div>

Valid

<div>
    <a href="https://plot.ly/~robertobruttomesso/32/?share_key=P9EyQuHh0kzYCBXBzkoJNx" target="_blank" title="intrepyd-vs-luke-valid" style="display: block; text-align: center;"><img src="https://plot.ly/~robertobruttomesso/32.png?share_key=P9EyQuHh0kzYCBXBzkoJNx" alt="intrepyd-vs-luke-valid" style="max-width: 100%;width: 700px;"  width="700" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="robertobruttomesso:32" sharekey-plotly="P9EyQuHh0kzYCBXBzkoJNx" src="https://plot.ly/embed.js" async></script>
</div>

Survival

<div>
    <a href="https://plot.ly/~robertobruttomesso/36/?share_key=TvQ8qC0thyV2VpHKxnq9yW" target="_blank" title="intrepyd-vs-luke-all-survival" style="display: block; text-align: center;"><img src="https://plot.ly/~robertobruttomesso/36.png?share_key=TvQ8qC0thyV2VpHKxnq9yW" alt="intrepyd-vs-luke-all-survival" style="max-width: 100%;width: 600px;"  width="600" onerror="this.onerror=null;this.src='https://plot.ly/404.png';" /></a>
    <script data-plotly="robertobruttomesso:36" sharekey-plotly="TvQ8qC0thyV2VpHKxnq9yW" src="https://plot.ly/embed.js" async></script>
</div>

[intrepyd]: http://github.com/formalmethods/intrepyd
[lustre]:   https://en.wikipedia.org/wiki/Lustre_(programming_language)
[luke]:     https://www.it.uu.se/edu/course/homepage/pins/vt11/lustre
[simulink]: https://www.mathworks.com/products/simulink.html
[hagen]:    http://clc.cs.uiowa.edu/Kind/Papers/Hag-PHD-08.pdf
