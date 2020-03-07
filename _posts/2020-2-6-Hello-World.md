---
layout: post
title: Template Meta-Programming Revisited 
---

# TMP for fun, power, madness and vice versa

## Around summer 2019
I've been experimenting with Tmp lately to understand this arcane part of our favorite language. 
I quickly settle for using kvasir.mpl since it had the best performance according to metaben.ch but the documentation was somewhat difficult to understand (at first !) but the vision and the continuation-style was really addicting.

Fast-forward a couple of months of dev and I was able to implement myself some functions using the guts of the library.
One problem in particular (the equivalent of std::unique in TMP) was giving me trouble. I tried to implement a particular fonction to help me with this and to my complete surprise was actually working, but found out that kvasir.mpl's INTERFACE, not the implementation, wasn't supporting this functionnality. 

## November 2019
I wanted to experiment with this newly found discovery and see where it lead to. If it didn't found something worthwhile, I wouldn't be writing a blog about it. 

##Now let me be clear about some things before I continue.
1. If you want to learn TMP : Learn kvasir.mpl or boost.mp11 or boost.hana first. 
2. The library I wrote isn't ready for production yet : There is some inconsistencies in the interface and the implementation linearise some functions up to 8 times, while kvasir.mpl goes up to 256 in some cases.
3. Since everything here is experimental, treat this as some back-alley deals that someone (me) is trying to get you into. 
4. My goal is simply to throw things at the comittee and say "Hey this work". 

# Let's start
[Link To the Library](https://github.com/Remi123/type_expr)

The goal of every type-based metaprogramming library is to start from a type T and get another type U.

### Meta-Expressions ( typename ... Es)
Every meta-expression in the namespace _type_expr_ respect the following concept which this is an examplar :
`namespace te = type_expr;
struct te::exemplar
{
    template<typename ... Ts>
    struct f { using type = /*Implementation*/};
};
`
The examplar type can also be templated. f is a variadic subtype that is supposed to receive all incoming inputs types.

There is a couple of fundamentals types that you need to know to better understand my library _type_expr_

### te::input_< typename ... Ts >
Certainly the most important type. It's a meta-expression typelist that simply ignore whatever it receive to use the Ts instead.
Almost all my other meta-expression returns some forms of `te::input_<result_t...>`. An `te::input_<T>` with only one type will be replace by T under evaluation.

### te::pipe_< typename ... Es >
This is the lazy version of the meta-expression magic types. It's only a holder for meta-expression like `te::input_<...>`. The only other type that pipe is allowed to know is `te::input_<...>` to do some syntaxic sugar. It's also a meta-expression that apply Es... to every types it receives throught the subtype f.

### using te::eval_pipe< typename ... Es > = typename te::pipe_< Es... >::template f<>::type;
This is the actual implementation of my evaluation. It's similar to `te::pipe_t` but I didn't want to have such a big difference in behavior under one additional character. This is where all the magic happen. Using those three types, we can do some simple stuff.

`static_assert(std::is_same<
te::eval_pipe_< te::input_<int> > // same as int 
, int >::value,"eval to int");`

> This is cool but boring and other libraries can do the same things easily.
I know, I'm not there yet.

What happen with this :
`te::eval_pipe_< te::input_< float > , te::input_< int > >` what is the resulting type here ? Hint : See the definition above
<details>
<summary>Answer</summary>
Each te::input ignore whatever it receive to use the types in it's template parameter instead.
So the second te::input receive float, but use int instead. The answer is thus `int`
</details>

`te::eval_pipe_< te::input_< float > , te::pipe_< te::input_< int > > >` what is the resulting type here ?
<details>
<summary>Answer</summary>
Under evaluation, te::pipe is a meta-expression that pass the incoming inputs types to the meta-expression Es... and return their result. The answer is still `int`
</details>

`te::eval_pipe_< te::input_< float > , te::pipe_< te::input_< int,float > > >` what is the resulting type here ?
<details>
<summary>Answer</summary>
We need something to hold multiples types and C++ doesn't yet have a good synthactic sugar over that. We cannot return `int,float` by themselves, so we simply returns `te::input_< int,float>`.
</details>

> Interesting, albeit readability is still concerning. But still nothing new under the sun.
I know, this is the basic.

Let's introduce more advanced meta-expression :

### te::flatten
This is a necessary evil in case of multiple nested inputs like this `te::eval_pipe_<te::input_< te::input_<int>, te::input_<float>, char >,te::flatten>`. It only remove the nested `te::input_`s to result into `te::input_<int,float,char>` 

### te::unwrap
This remove the template template parameter which is useful in many place. `te::eval_pipe_<te::input_< std::tuple<int,float,char>>, te::flatten>` will result into `te::input<int,float,char> `and this allows some optimization.

### te::quote_< template<typename ... Ts> class F >
Similar to mp11's `mp_quote`, this thing wrap multiple types into a single one define in the template template. Be cautious that it only accept template that receive only types, no `bool` or `int` or whatever.
`te::eval_pipe_< te::input_<int,float,char>, te::quote_<std::tuple> >` will result into `std::tuple<int,float,char>
