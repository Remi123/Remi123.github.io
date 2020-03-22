---
layout: post
title: Template Meta-Programming Revisited 
---

# TMP for fun, power, madness and vice versa

## Around summer 2019
I've been experimenting with Tmp lately to understand this arcane part of our favorite language. 
I quickly settle for using kvasir.mpl since it had the best performance according to metaben.ch but the documentation was somewhat difficult to understand (at first !) but the vision and the continuation-style was really addicting.

Fast-forward a couple of months of dev and I was able to implement myself some functions using the guts of the library.
One problem in particular (the equivalent of std::unique in TMP) was giving me trouble. I tried to implement a particular fonction to help me with this and to my complete surprise was actually working, but found out that kvasir.mpl's INTERFACE ( not the implementation) wasn't supporting this functionnality. 

## November 2019
I wanted to experiment with this newly found discovery and see where it lead to. If it didn't found something worthwhile, I wouldn't be writing a blog about it. 

## Now let me be clear about some things before I continue.
1. If you want to learn TMP : Learn kvasir.mpl or boost.mp11 or boost.hana first. 
2. The library I wrote isn't ready for production yet : There is some inconsistencies in the interface and the implementation linearise some functions up to 8 times, while kvasir.mpl goes up to 256 in some cases.
3. Since everything here is experimental, treat this as some back-alley deals that someone (me) is trying to get you into. 
4. My goal is simply to throw things at the committee and say "Hey this work". 

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
    static_assert(
        std::is_same< int,
                      te::eval_pipe_< te::input_<int> > // same as int 
        >::value
    ,"eval to true if it's the same as int");
```

> This is cool but boring and other libraries can do the same things easily.

I know, I'm not there yet.

___

With all those information, try to evaluate the resulting type (Click to see the answer) : 
<details>
<summary>`te::eval_pipe_< te::input_< float > , te::input_< int > >`</summary>
Each `te::input` ignore whatever it receive to use the types in it's template parameter instead.
So the second `te::input` receive `float`, but use `int` instead. The answer is thus `int`
</details>
<details>
<summary>
`te::eval_pipe_< te::input_< float > , te::pipe_< te::input_< int > > >` 
</summary>
Under evaluation, te::pipe is a meta-expression that pass the incoming inputs types to the meta-expression `Es...` and return their result. The answer is still `int`
</details>
<details>
<summary>
`te::eval_pipe_< te::input_< float > , te::pipe_< te::input_< int,float > > >` 
</summary>
We need something to hold multiples types and C++ doesn't yet have a good synthactic sugar over that. We cannot return `int,float` by themselves, so we simply returns `te::input_< int,float>`.
</details>

___

> Interesting, albeit readability is still concerning. But still nothing new under the sun.

I know, this is the basic.

Let's introduce more advanced meta-expression :


### te::identity

This simple meta-expression simply continue with what it receive. It serve more purpose than it may seems. `te::eval_pipe_< te::input_<int>, te::identity>` result in `int`.


### te::flatten

This is a necessary evil in case of multiple nested inputs like this `te::eval_pipe_<te::input_< std::string, te::input_<int>, te::input_<float>, char >,te::flatten>`. It only remove the nested `te::input_`s to result into `te::input_<std::string, int, float, char>` 


### te::unwrap

This remove the template template parameter which is useful in many place. `te::eval_pipe_<te::input_< std::tuple<int,float,char>>, te::unwrap>` will result into `te::input<int,float,char> `and this allows some optimization.


### te::quote_< template<typename ... Ts> class F >

Similar to mp11's `mp_quote`, this thing wrap multiple types into a single one defined in the template template parameter. Be cautious that it only accept template that receive only types, no `bool` or `int` or whatever.
`te::eval_pipe_< te::input_<int,float,char>, te::quote_<std::tuple> >` will result into `std::tuple<int,float,char>`
This function have a sister function named `lift_<template<...> class F>` due to the std.type_trait library that need to quote then to get the nested `::type`. 
Rule of thumb : use quote most of the time, use lift if you need to use std::type_traits meta-function


> Ok I get that you can play with template template but other libraries can do it too.
> However, somethings sound familiar here and I can't put my finger on it...

It will be more visible when I introduce the whole mp11-like meta-expression and start to chain them.
```C++
    template<int I> using int_c = std::integral_constant<int,I>;

    struct Local_Meta_Expr : 
        te::pipe_<
                te::transform_< te::multiply_<te::int_c<2> > >,
                te::transform_< te::mkseq, te::quote_std::integer_sequence >
                >{};

    static_assert(
        te::eval_pipe_<
                        te::input_<int_c<1>, int_c<2>>,
                        Local_Meta_Expr,
                        te::is_<std::integer_sequence<0,1>, std::integer_sequence<0,1,2,3>> 
        >, "Compile fine")
```

Ok that's a lot to digest but it show almost everything I want here 


1. `pipe_` is being used here as a lazy view of any operation.
2. `transform_` is more similar to `std::transform()` so I could rename it `te::foreach_`
3. `mkseq` only accept `std::integral_constant<int,N> `(for the moment) and output `std::integral_constant` from 0 to N-1.
4. `quote_std_integer_sequence` is unfortunate, but needed since the difference of signature (`quote_` only accept types wrapper with signature `template<typename ... Ts>`  and `std::integer_sequence` have the signature `template<typename T, T ... value>`)
5. `is_<typename ... Ts>` is very similar to `std::is_same` but checks if the types it receive is the same as `Ts...` 
6. I'm manipulating multiple types at the same time very easily
7. Local_Meta_Expr inherite all this behavior from `te::pipe_<...>`. It could have been a typedef or an alias template, but I wanted to show that it doesn't affect the behavior.
8. This is the closest I can get to range-like syntax for manipulating types

If you squeeze you're eye a little and replace some commas with the pipe operator | , you can see that it's very similar to the range syntax.

> ... Ok now you have my attention. The functionnality can still be done with the other library but you have a better looking syntax.

Ok now let's open Pandora's box.

> ... What do you mean?

Let's create meta-expression out of other meta-expressions. Since most of my meta-expression are respecting the template signature of `template< typename ...>`, even `te::pipe_<typename ... Es>` , we can use `te::quote_<...>` inside `te::eval_pipe_<...>` to create new meta-expression.

Let's show some of my favorites examples

```C++
    template<int N>
    struct copy_ : te::eval_pipe_< te::input_<int_c<N>>,
                                    te::mkseq,
                                    te::transform_<te::input_<te::identity>>, 
                                    te::quote_<te::fork_>>
                                    {}; 
    // all of this eval to fork_<identity,identity,... > with N-time identity
    // copy_<int I> inherit from te::fork_<te::identity,...>
```

`te::fork_<typename ... Es>` is the opposite of `te::transform_`. It copies the inputs received to all meta-expression `Es...` which is exactly what we wanted to do in this case.

Technically, what we are doing here is modifying a function to be written as another one.

If this is not mind-blowing enough, let's me introduce you `te::repeat_`

```C++
    template <std::size_t N, typename... Es>
    struct repeat_ : te::eval_pipe_<te::input_<Es...>, 
                                te::copy_<N>, 
                                te::flatten, 
                                te::quote_<te::pipe_>> {};

    static_assert(
        eval_pipe_<te::input_<int>,
                   te::repeat_< 2, te::lift_<std::add_pointer>, te::lift_<std::add_const>>,
                   te::is_<int *const *const>>::value,
        "");
```

`te::repeat_` is the first meta-expression that I can confidently say that no other library goes as far. I'm not saying it's impossible to implement, but they will fight their own interface due to the nesting of their meta-functions.
If you are familiar with kvasir.mpl documentations, there is a little meta-function named ``kvasir::mpl::call_f ``with the comment literally saying "/ \brief experimental, may be depricated or changed". This is because it consider it's input as meta-function.  My library is basically an experiment of using this function all over the place.

> The problem is not that I've tried making it work, the problem is that I've succeeded

I'm now able to implement some cool function like `te::swizzle<int ... I>`

```C++
    template<int ... I>
    struct swizzle : te::fork_<te::get_<I>...>{};

    static_assert(
    te::eval_pipe_<
            te::input_<int, int *, int **, int ***>, 
            te::swizzle_<2, 1, 0, 3, 1>,                    // Get by index
            te::is_<int **, int *, int, int ***, int *>
    >::value, " Swizzling is easy");
```

The last example actually modified a function using variadic pack expensions. This is one of the reason I wanted to start a library since others libraries made it too difficult to do such things.

For now, I've already implemented two higher-meta-expression using `te::fork_` , but I'm not surprise one bit. Odin Holmes actuallly did a presentation on this ( in Boost(not yet!).tmp, it's named `boost::tmp::tee_`) and fork was the first meta-expression that blew my mind. I remember my big meta-functions that wasn't working, replacing it by fork and three little dot and seeing it work perfectly on the first try.

The last higher-meta-expression I want to talk about is `te::on_args_<typename ... Es>`. This solve a weird problem in template library where you have something like a `std::tuple<Ts...>` and just want to play with the inner types, then rewrap with tuple. The problem is that it's technically a series of functions that depend on the input, which is notoriously difficult since you have add your continuation to the last meta-function which can be nested inside other meta-function. Not in my library, albeit I admit the implementation is a little bit frankenstein. However, this is my actual solution to Arthur O'Dwyer post about template library released in December 2019 : 

```C++
    // On this challenge, the goal was to unwrap, remove empty class, sort them by
    // size and rewrap them. on_args_<Es...> deals with the unwrap rewrap if the
    // signature of the type accept only types.
    struct Z {};  // EMPTY CLASS
    static_assert(
      te::eval_pipe_<
          te::input_<std::tuple<Z, int[4], Z, int[1], Z, int[2], int[3]>>,
          te::on_args_<te::remove_if_<te::lift_<std::is_empty>>,
                       te::sort_<te::transform_<te::size>, te::greater_<>>
                       >,
          te::is_<std::tuple<int[4], int[3], int[2], int[1]>>
        >::value, "Arthur O'Dwyer");
    // No unwrap, no as_lambda, no custom implementation. Simply on_args. 
```

Since we have abstracted the whole unwrap-wrap, might as well test this function with different types

```C++
    static_assert(
      te::eval_pipe_<
          te::input_<
            std::tuple<Z, int[4], Z, int[1], Z, int[2], int[3]>,
            te::ls_<int[2], Z, int[1], Z, int[3]>
          >,
          te::transform_<
              te::on_args_<te::remove_if_<te::lift_<std::is_empty>>,
                           te::sort_<te::transform_<te::size>, te::greater_<>>
                            >
                        >,
          is_<std::tuple<int[4], int[3], int[2], int[1]>
              ,te::ls_<int[3],int[2],int[1]>>
        >::value, "Arthur O'Dwyer but with multiple types");   
```

>That's amazing, do you have any other surprise.

Well, I have this feature that is not present on every feature since I've recently implemented it, but I actually do error management. However, it's a little bit ... unconventional.

Let's say you want to unwrap an `int`, which is not possible since `int ` is not a template template.
`te::eval_pipe_<te::input_<int>, te::unwrap> test_error = int{}`. Without any error management, you would have a huge compiler message with nested error reporing. I was as tired of seeing this as you do. What I did was, since a lot of my code is centralized into what I call a context, if I know an error is produced in this context, I can just create a type named `te::error_<>` and put whatever message I wanted inside in the form of a type.

In the case of unwrapping an `int`, I return something like `error_<te::unwrap,te::unspecialized, int>` because I don't have a specialization for unwrapping an int. The _very_ cool thing is that further evaluation of meta-expression will do _nothing_ once it saw that an `te::error_<>` has been evaluated. It's not perfect, if you have multiple types and one of them is an error type, then there is some possibility that I can continue with that, but it's a cool feature.

Albeit weird, this system allow me to greatly reduce the compiler log size, which is my goal in 99% of the case. Not all function support it yet since it's fairly new.

>Any other remarks

Yes, actually I want to refine my definition of a meta-predicate
Some of my meta-expression require some expression that are either unary of binary predicate. Something like `
    template<typename ... BinaryPred>
    struct sort_;`
This need a little explaining, but a meta-expression binary-predicate can be multiple expression that take 2 types and return either `std::integral_constant<bool,false>` or `std::integral_constant<bool,true>`. Nothing else. I'm not converting into them, nor anything else.

Let's say you have` te::sort_<transform_<te::size>, greater_<>>`. The sort meta-expression will give two types at a time to the sub-meta-expressions. The `te::size` will transform 1 type into their size in `std::integral_constant<int,Size>` and `te::greater_<>`, without argument, will look at those two types receive and "return" either `true_type or false_type` depending if the first is greater than the second.

## What's Next

I'm very satisfied with this library. In
