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

## Now let me be clear about some things before I continue.
1. If you want to learn TMP : Learn kvasir.mpl or boost.mp11 or boost.hana first. 
2. The library I wrote isn't ready for production yet : There is some inconsistencies in the interface and the implementation linearise some functions up to 8 times, while kvasir.mpl goes up to 256 in some cases.
3. Since everything here is experimental, treat this as some back-alley deals that someone (me) is trying to get you into. 
4. My goal is simply to throw things at the comittee and say "Hey this work". 

# Let's start
[Link To the Library](https://github.com/Remi123/type_expr)

The goal of every type-based metaprogramming library is to start from a type T and get another type U.

### Meta-Expressions ( typename ... Es)
Every meta-expression in the namespace _type_expr_ respect the following concept which this is an examplar :
```C++
    namespace te = type_expr;
    struct te::exemplar
    {
        template<typename ... Ts>
        struct f { using type = /*Implementation*/};
    };
```
The examplar type can also be templated. f is a variadic subtype that is supposed to receive all incoming inputs types.

There is a couple of fundamentals types that you need to know to better understand my library _type_expr_

### te::input_< typename ... Ts >
Certainly the most important type. It's a meta-expression typelist that simply ignore whatever it receive to use the Ts instead.
Almost all my other meta-expression returns some forms of `te::input_<result_t...>`. An `te::input_<T>` with only one type will be replace by T under evaluation.

### te::pipe_< typename ... Es >
This is the lazy version of the meta-expression magic types. It's only a holder for meta-expression like `te::input_<...>`. The only other type that pipe is allowed to know is `te::input_<...>` to do some syntaxic sugar. It's also a meta-expression that apply Es... to every types it receives throught the subtype f.

### using te::eval_pipe< typename ... Es > = typename te::pipe_< Es... >::template f<>::type;
This is the actual implementation of my evaluation. It's similar to `te::pipe_t` but I didn't want to have such a big difference in behavior under one additional character. This is where all the magic happen. Using those three types, we can do some simple stuff.

```C++
    static_assert(std::is_same<
    te::eval_pipe_< te::input_<int> > // same as int 
    , int >::value,"eval to int");
```

> This is cool but boring and other libraries can do the same things easily.

I know, I'm not there yet.

___

What happen with this :
`te::eval_pipe_< te::input_< float > , te::input_< int > >` 
What is the resulting type here ? Hint : See the definition above
<details>
<summary>Answer</summary>
Each te::input ignore whatever it receive to use the types in it's template parameter instead.
So the second te::input receive float, but use int instead. The answer is thus `int`
</details>
___

`te::eval_pipe_< te::input_< float > , te::pipe_< te::input_< int > > >` 
What is the resulting type here ?
<details>
<summary>Answer</summary>
Under evaluation, te::pipe is a meta-expression that pass the incoming inputs types to the meta-expression Es... and return their result. The answer is still `int`
</details>
___

`te::eval_pipe_< te::input_< float > , te::pipe_< te::input_< int,float > > >` 
What is the resulting type here ?
<details>
<summary>Answer</summary>
We need something to hold multiples types and C++ doesn't yet have a good synthactic sugar over that. We cannot return `int,float` by themselves, so we simply returns `te::input_< int,float>`.
</details>
___

> Interesting, albeit readability is still concerning. But still nothing new under the sun.

I know, this is the basic.

Let's introduce more advanced meta-expression :

`te::flatten`

This is a necessary evil in case of multiple nested inputs like this `te::eval_pipe_<te::input_< std::string, te::input_<int>, te::input_<float>, char >,te::flatten>`. It only remove the nested `te::input_`s to result into `te::input_<std::string, int, float, char>` 
___
`te::unwrap`

This remove the template template parameter which is useful in many place. `te::eval_pipe_<te::input_< std::tuple<int,float,char>>, te::flatten>` will result into `te::input<int,float,char> `and this allows some optimization.
___
`te::quote_< template<typename ... Ts> class F >`

Similar to mp11's `mp_quote`, this thing wrap multiple types into a single one define in the template template. Be cautious that it only accept template that receive only types, no `bool` or `int` or whatever.
`te::eval_pipe_< te::input_<int,float,char>, te::quote_<std::tuple> >` will result into `std::tuple<int,float,char>`
This function have a sister function named `lift_<template<...> class F>` due to the std.type_trait library that need to quote then to get the nested `::type`. 
___

> Ok I get that you can play with template template but other libraries can do it too.
> However, somethings sound familiar here and I can't put my finger on it...

It will be more visible when I introduce the whole mp11-like meta-expression and start to chain them.

`template<int I> using int_c = std::integral_constant<int,I>;
// too long to write obviously

struct Local_Meta_Expr : te::pipe_< 
                te::transform_< te::multiply_< te::int_c<2> > >,
                te::transform_< te::mkseq, te::quote_std_integer_sequence >                
                >{};
static_assert(te::eval_pipe_< 
                te::input_<int_c<1>, int_c<2>>,
                Local_Meta_Expr ,
                te::is_<std::integer_sequence<0,1>, std::integer_sequence<0,1,2,3>> 
                >::value , "Compile");
                `
Ok that's a lot to digest but it show almost everything I want here :

1. `pipe_` is being used here as a lazy view of any operation.
2. `transform_` is more similar to `std::transform()` so I could rename it `te::foreach_`
3. `mkseq` only accept `std::integral_constant<int,N> `(for the moment) and output `std::integral_constant` from 0 to N-1.
4. `quote_std_integer_sequence` is unfortunate, but needed since the difference of signature (`quote_` only accept types and `std::integer_sequence` have the signature `template<typename T, T ... value>`)
5. `is_<typename ... Ts>` is very similar to `std::is_same` but checks if the types it receive is the same as `Ts...` 
6. I'm manipulating multiple types at the same time very easily
7. This is the closest I can get to range-like syntax for manipulating types
8. Local_Meta_Expr inherite all this behavior from pipe. It could have been a typedef or an alias template, but I wanted to show that it doesn't affect the behavior.

> ... Ok now you have my attention. The functionnality can still be done with the other library but you have a better looking syntax.
Ok now let's open Pandora's box.

> ... What do you mean?
Let's create meta-expression out of other meta-expressions. Let's use `te::quote_< te::pipe_ >`.
This would wrap the current inputs with `te::pipe_`, and if the types are themselves meta-expressions, this create a new meta-expressions.

Let's show some of my favorites examples

`template<int N>
struct copy_ : te::eval_pipe_< te::input_<int_c<N>>,
                                te::mkseq,
                                te::transform_<te::input_<te::identity>>, 
                                te::quote_<te::fork_>>
                                {}; // all of this eval to fork_<identity,identity,..., N> `

`te::fork_<typename ... Es>` is the opposite of `te::transform_`. It copies the inputs received to all meta-expression `Es...` which is exactly what we wanted to do in this case.

If this is not mind-blowing enough, let's me introduce you `te::repeat_`

`template <std::size_t N, typename... Es>
struct repeat_ : eval_pipe_<input_<Es...>, copy_<N>, flatten, quote_<pipe_>> {};

static_assert(
    eval_pipe_<input_<int>,
               repeat_<2, lift_<std::add_pointer>, lift_<std::add_const>>,
               is_<int *const *const>>::value,
    "");`
    


