# CLP(BNR)
This repository is a re-implementation of CLP(BNR) in Prolog and packaged as an SWI-Prolog module. The original implementation was a component of BNR Prolog (.ca 1988-1996) on Apple Macintosh and Unix that has been lost (at least to the original implementors) so this is an attempt to capture the design and provide a platform for executing the numerous examples found in the [literature][bnrpp]. As it is constrained by the host environment (SWI-Prolog) it can't be 100% compliant with the original implementation(s), but required changes should be minimal.
 
In the process of recreating this version of CLP(BNR) subsequent work in relational interval arithmetic has been used, in particular [Efficient Implementation of Interval Arithmetic Narrowing Using IEEE Arithmetic][ia1] and [Interval Arithmetic: from Principles to Implementation][ia2]. In addition, there is at least one comparable system [CLIP][clip] that is targeted at GNU Prolog, ([download CLIP][cldl]). While earlier implementations typically use a low level  of the relational arithmetic primitives, advances in computing technology enable this Prolog version of CLP(BNR) to achieve acceptable performance while maintaining a certain degree of platform independence (addressed by SWI-Prolog) and facilitating experimentation (no "building" required). [SWI-Prolog][swip] was used due it's long history (.ca 1987) as free, open-source software and for supporting `freeze` which is a key mechanism in implementing constrained logic variables.

## A Brief Introduction

An interval in CLP(BNR) is a logic variable representing a set of real numbers between (and including) a numeric lower bound and upper bound. Intervals are declared using the `::` infix operator, e.g.,

	?- X::real, [A,B]::real(0,1).
	X = _2820::real(-1.7976931348623157e+308,1.7976931348623157e+308),
	A = _2926::real(0.0,1.0),
	B = _3032::real(0.0,1.0).

Intervals are output by the Prolog listener in declaration form with the original variable name replaced with internal variable representation (the "frozen" logic variable). The current bounds values are arguments in a term whose functor is the type, e.g., `real` in the above example. Note that if bounds are not specified, the defaults are large, but finite, platform dependent values. The infinities (`inf` and `-inf`) are supported but must be specified in the declaration. 

To constrain an interval to integer values, type `integer` can be used in place of `real`:

	?- I::integer, [J,K]::integer(-1,1).
	I = _2830::integer(-72057594037927936,72057594037927935),
	J = _2952::integer(-1,1),
	K = _3074::integer(-1,1).

Finally, a `boolean` is an `integer` constrained to have values `0` and `1` (meaning `false` and `true` respectively).

	?- B::boolean.
	B = _2622::integer(0,1).

Interval declarations define the initial bounds of the interval which are narrowed over the course of program execution. Narrowing is controlled by defining constraints in curly brackets, e.g.,

	?- [M,N]::integer(0,8), {M == 3*N}.
	M = _2808::integer(0,6),
	N = _2930::integer(0,2).

	?- [M,N]::integer(0,8), {M == 3*N}, {M>3}.
	M = 6,
	N = 2.

	?- {1==X + 2*Y, Y - 3*X==0}.
	X = _84::real(0.14285714285714268,0.14285714285714302),
	Y = _140::real(0.42857142857142827,0.4285714285714289).

In the first example, both `M` and `N` are narrowed by applying the constraint `{M == 3*M}`. Applying the additional constraint `{M>3}` further narrows the intervals to single values so the original variables are unified with the point (integer) values.
 
In the last example, there are no explicit declarations of `X` and `Y`; in this case the default `real` bounds are used. The constraint causes the bounds to contract almost to a single point value, but not quite. The underlying reason for this is that standard floating point arithmetic is mathematically unsound due to floating point rounding errors and cannot represent all the real numbers in any case, due to the finite bit length of the floating point representation. Therefore all the interval arithmetic primitives in CLP(BNR) round any computed result outwards (lower bound towards -inifinity, upper bound towards infinity) to ensure that any answer is included in the resulting interval, and so is mathematically sound. (This is just a brief, informal description; see the literature on interval arithmetic for a more complete justification and analysis.)

Sound constraints over real intervals (since interval ranges are closed) include `==`, `=<`, and `>=`, while `<>`, `<` and `>` are provided for `integer`'s. A fairly complete set of standard arithmetic operators (`+`, `-`, `*`, `/`, `**`), boolean operators (`and`, `or`, `xor`, `nand`, `nor`) and common functions (`exp`, `log`, `sqrt`, `abs`, `min`, `max` and standard trig and inverse trig functions) provided as interval relations. Note that these are relations so `{X==exp(Y)}` is the same as `{log(X)==Y}`. A few more examples:

	?- {X==cos(X)}.
	X = _44::real(0.7390851332151601,0.7390851332151612).

	?- {X>=0,Y>=0, tan(X)==Y, X**2 + Y**2 == 5 }.
	X = _58::real(1.0966681287054703,1.0966681287054718),
	Y = _106::real(1.9486710896099515,1.9486710896099542).

	?- {Z==exp(5/2)-1, Y==(cos(Z)/Z)**(1/3), X==1+log((Y+3/Z)/Z)}.
	Z = _110::real(11.182493960703471,11.182493960703475),
	Y = _242::real(0.25518872031001844,0.25518872031002077),
	X = _612::real(-2.061634262247237,-2.061634262247228).

It is often the case that the fixed point iteration that executes as a consequence of applying constraints is unable to generate meaningful solutions without applying additional constraints. This may be due to multiple distinct solutions within the interval range, or because the interval iteration is insufficiently "clever". A simple example:

	?- {2==X*X}.
	X = _10074::real(-1.4142135623730954,1.4142135623730954).
but 

	?- {2==X*X, X>=0}.
	X = _2426::real(1.4142135623730947,1.4142135623730954).

In more complicated examples, it may not obvious what the additional constraints might be. In these cases a higher level search technique can be used, e.g., enumerating integer values or applying "branch and bound" algorithms over real intervals. A general predicate called `solve` is provided for this purpose. Additional application specific techniques can also be implemented. Some examples:

	?- {2==X*X},solve(X).
	X = _7534::real(-1.4142135623730954,-1.4142135623730947) 
	X = _7534::real(1.4142135623730947,1.4142135623730954).

	?- X::real(0,1), {0 == 35*X**256 - 14*X**17 + X}, solve(X).
	X = _68::real(0.0,2.225074172065353e-308) 
	X = _68::real(0.847943660827315,0.8479436608273155) 
	X = _68::real(0.9958424942004978,0.9958424942004983).

	?- {Y**3+X**3==2*X*Y, X**2+Y**2==1},solve([X,Y]).
	Y = _92::real(0.3910520007735322,0.3910520069182414),
	X = _226::real(-0.9203685852369246,-0.9203685826261211) 
	Y = _86::real(-0.9203685842547779,-0.9203685836401),
	X = _220::real(0.3910520030850841,0.39105200453177025) 
	Y = _86::real(0.8931356147216993,0.8931356366029135),
	X = _220::real(0.4497874327167326,0.44978747616590165) 
	Y = _86::real(0.44978742979729985,0.4497875096555091),
	X = _220::real(0.8931355978561679,0.8931356380731539) 
	false.

Note that all of these examples produce multiple solutions that are produce by backtracking in the listener (just like any other top level query).

Here is the current set of operators and functions supported in this version:

	== is <> =< >= < >                    %% equalities and inequalities
	+ - * /                               %% basic arithmetic
	**                                    %% includes real exponent, odd/even integer
	abs                                   %% absolute value
	sqrt                                  %% square root (needed?)
	min max                               %% min/max (arity 2)
	<= =>                                 %% inclusion
	and or nand nor xor ->                %% boolean
	- ~                                   %% unary negate and not
	exp log                               %% exp/ln
	sin asin cos acos tan atan            %% trig functions


Further explanation and examples can be found at [BNR Prolog Papers][bnrpp].

[ia1]:   http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.44.6767
[ia2]:   http://fab.cba.mit.edu/classes/S62.12/docs/Hickey_interval.pdf
[clip]:  https://scholar.lib.vt.edu/ejournals/JFLP/jflp-mirror/articles/2001/S01-02/JFLP-A01-07.pdf
[cldl]:  http://interval.sourceforge.net/interval/index.html
[swip]:  http://www.swi-prolog.org
[bnrpp]: https://ridgeworks.github.io/BNRProlog-Papers
	

## Getting Started

The SWI-Prolog package declaration is:

	:- module(clpBNR,          % SWI module declaration
		[
		op(700, xfy, ::),
		(::)/2,                % declare interval
		{}/1,                  % add constraint
		interval/1,            % filter for constrained var
		list/1,                % for compatability
		range/2,               % for compatability
		lower_bound/1,         % narrow interval to point equal to lower bound
		upper_bound/1,         % narrow interval to point equal to upper bound
					   
		% constraint operators
		op(200, fy, ~),        % boolean 'not'
		op(500, yfx, and),     % boolean 'and'
		op(500, yfx, or),      % boolean 'or'
		op(500, yfx, nand),    % boolean 'nand'
		op(500, yfx, nor),     % boolean 'nor'
		op(500, yfx, xor),     % boolean 'xor'
		op(700, xfx, <>),      % integer
		op(700, xfx, <=),      % set inclusion
		op(700, xfx, =>),      % set inclusion
					   
		% utilities
		print_interval/1,      % print interval as a list of bounds, for compaability, 
		solve/1, solve/2,      % search intervals using split
		enumerate/1,           % specialized search on integers
		clpStatistics/0,       % reset
		clpStatistic/1,        % get selected
		clpStatistics/1        % get all defined in a list
		]).


To load CLP(BNR) on SWI-Prolog, consult `clpBNR.pl` (in `src/` directory) which will automatically include `ia_primitives.pl` and `ia_utilities.pl`.
