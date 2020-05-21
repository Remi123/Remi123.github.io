---
layout: post
title: Template Meta-Programming Revisited 
---

## Range-ifying TMP for fun, power, madness and vice versa

Before you start to read an heavy template meta-programming article, I'll get you motivated with a cool example. This will show the similarity with the Std.Range library :

```C++
    // range library to double each int values in a container and calculate the summation 
    
	int accumulator = 0;
	auto fct_actions =
		ranges::actions::transform([](int& i){ return i*=2;} )
	|	ranges::actions::transform( [&accumulator](const int& i)
						{accumulator += i; return i;});

	std::vector<int> vi{1,2,3};
	vi |= fct_actions;
	assert(accumulator == 12);

    //-------------------------------------------------------------------------

	// type_expr library to do the same thing with std::integral_constant types
    template <int N> using int_c = std::integral_constant<int,N>;
    
    using type_expression = 
        te::pipe_<
                te::transform_<te::multiply_<int_c<2>>>
                ,te::fold_left_<te::plus_<>> // Very similar to std::accumulate()
        >; 
    using result_type = te::eval_pipe_<
                                te::input_<int_c<1>,int_c<2>,int_c<3>>, 
                                type_expression
                                >; 
	static_assert( std::is_same_v<int_c<12>, result_type>,"");
```

Intrigued ? Continue reading to discover even more amazing things about my library _type_expr_ . If you are not familiar with meta-programming, just assumes that it works. I'll explain the previous example later.

## Let's start

I've been experimenting with Type-based meta-programming (Tmp) lately to understand this arcane part of our favorite language. 
I quickly settle for using kvasir.mpl since it had the best performance according to metaben.ch but the documentation was somewhat difficult to understand (at first !) but the vision and the continuation-style was really addicting.

One problem in particular (the equivalent of std::unique in TMP) was giving me trouble. I tried to implement a particular function to help me with this and to my complete surprise was actually working, but found out that kvasir.mpl's INTERFACE ( not the implementation) wasn't supporting this functionality. 

Obviously I wanted to expand this idea further and create (yet another) meta-programming library.

After a lot of trial and compiler error novel, I was able to reproduce most of the other type-based meta-programming library's functionalities. I was also able to create something new that require some explanation but if it wasn't something worthwhile, I wouldn't be writing a blog about it.

## Let's start
[Link To the Library](https://github.com/Remi123/type_expr)


## Now let me be clear about some things before I continue.
1. If you want to learn TMP : Learn kvasir.mpl or boost.mp11 or boost.hana first. 
2. The library I wrote isn't ready for production yet : There is some inconsistencies in the interface and the implementation linearize some functions up to 8 times, while kvasir.mpl goes up to 256 in some cases.
3. Since everything here is experimental, treat this as some back-alley deals that someone (me) is trying to get you into. 
4. You can consider this library a fork of kvasir.mpl since I use a lot of the same technique internally to make it fast. I also do my fair share of forbidden magic.
5. My goal is simply to throw things at the committee and say "Hey this works". 
6. Developing this come as the cost of your sanity.

## C++20 Concept : Meta-Expression

For those of you that want to know which Concept my library respect ( as of C++20 concept ), here is an exemplar to demonstrate how they are structured, but keep in mind that my library is C++11-compatible so no requires:

```C++
    template<typename ... Types>
    struct meta_function_name // Note: All std.type_trait meta-function respect more or less this structure  
    {
        using type = /* implementation defined */;
    };
    // A meta-function is something like std::add_const<int> where all the arguments are provided in the primary struct and the resulting type is accessed with the underlying type 'typename std::add_const<int>::type' .

    template< typename ... MetaExpressions > //This line is optional or modifiable
    struct meta_expression_name
    {
        template < typename ... Types >
        struct f
        {
            using type = /* implementation defined */;
        };
    };
    // A meta-expression have two variadic entry points which are where you insert the template arguments : the name of the meta-expression (meta_expression_name) and the input types in f. You can see that f is actually a meta-function in concept, but always named f. 
```

This means that each meta-expressions contains another variadic struct named `f` which contains a type named `type` which is the result to the meta-expression. I always consider the template arguments of f as 'inputs types'.

For example, adding a pointer to a type. `typename te::add_pointer::template f<int>::type `would be `int*` for example.
However, and this is key, the meta_expression_name's template argument (named MetaExpressions in the example above ) can also be meta-expression themselves, which I consider to be sub-expression. A good example would be meta-predicate.

Obviously, my library tries to hide all ugly `typename `and `template f` as much as possible. But this allows the meta-expressions to have two kinds of inputs : another meta-expression in the primary template, and "inputs" type in the sub-type `f`. My choice to abstract all of this was to create two specials meta-expressions (`te::input_` and `te::pipe_`)and an type alias( `te::eval_pipe_`).


### te::input_< typename ... Ts >
Certainly the most important type. It's a meta-expression type-list that simply ignore whatever it receives to use the Ts instead.
Almost all my other meta-expression return some form of `te::input_<result_t...>`. An `te::input_<T>` with only one type will be replaced by T under evaluation.

### te::pipe_< typename ... Es >
This is the lazy version of the meta-expression magic types. It's only a holder for meta-expression like `te::input_<...>`. The only other type that pipe is allowed to know is `te::input_<...>` to do some syntactic sugar. It's also a meta-expression that apply `Es...` to every type it receives through the sub-type f.

### using te::eval_pipe< typename ... Es > = typename te::pipe_< Es... >::template f<>::type;
This is the actual implementation of the evaluation. It's similar to `te::pipe_t` but I didn't want to have such a big difference in behavior under one additional character. This is where all the magic happens. You can actually see that the sub-type `f<>` starts with no types. This mean that most to the meta-expressions will start with `te::input_<Ts...>` . Using those three types, we can do some simple stuff.

```C++
    static_assert(
        std::is_same< int,
                      te::eval_pipe_< te::input_<int> > // same as int 
        >::value
    ,"Evaluate to true_type if it's the same as int");

    // Same thing but only using my library
    static_assert(
        te::eval_pipe_<te::input_<int,float>,
                        te::same_as_<int,float>>::value,
    "te::same_as_ is similar to std::is_same, but check if the types receives are the same as the on in the template arguments.");
```

This is very basic and every other library can do a variation of std::is_same. But more interesting thing will be demonstrated later in this post.

___

With all those information, try to evaluate the resulting type (Click to see the answer) : 
<details>
<summary>`te::eval_pipe_< te::input_< float > , te::input_< int > >`</summary>
Each `te::input` ignores whatever it receives to use the types in it's template parameter instead.
So the second `te::input` receives `float`, but use `int` instead. The answer is thus `int`
</details>
<details>
<summary>
`te::eval_pipe_< te::input_< float > , te::pipe_< te::input_< int > > >` 
</summary>
Under evaluation, te::pipe is a meta-expression that pass the incoming inputs types to the meta-expression `Es...` and return their result. The answer is still `int`. This is a little bit strange but it's extremely important that `te::pipe_`, under evaluation, work exactly as if I didn't write it.
</details>
<details>
<summary>
`te::eval_pipe_< te::input_< float > , te::input_< int,float > >` 
</summary>
We need something to hold multiples types and C++ doesn't yet have a good syntactic sugar over that. We cannot return `int,float` by themselves, so we simply returns `te::input_< int,float>`.
</details>
___

It's still basic, but for me those kind of basic stuff must work. 

Let's introduce more advanced meta-expressions.


### te::identity

This simple meta-expression simply continue with what it receive. It serves more purpose than it may seems. 

`te::eval_pipe_< te::input_<int>, te::identity>` result in `int`.


### te::flatten

This is a necessary evil in case of multiple nested inputs like this `te::eval_pipe_<te::input_< std::string, te::input_<int>, te::input_<float,int>, char >,te::flatten>`. It only removes the nested `te::input_`s to result into `te::input_<std::string, int, float,int, char>` 


### te::unwrap

This remove the template template parameter which is useful in many place. `te::eval_pipe_<te::input_< std::tuple<int,float,char>>, te::unwrap>` will result into `te::input<int,float,char> `and this allows some optimization.


### te::quote_< template<typename ... Ts> class F >

Similar to mp11's `mp_quote`, this thing wraps multiple types into a single one defined in the template template parameter. Be cautious that it only accepts template that receive only types, no `bool` or `int` value.
`te::eval_pipe_< te::input_<int,float,char>, te::quote_<std::tuple> >` will result into `std::tuple<int,float,char>`
`te::eval_pipe_< te::input_<int>, te::quote_<std::vector>>` will result into `std::vector<int,std::allocator<int>> // std::vector have a default template parameter`
This function have a sister function named `lift_<template<...> class F>` due to the std.type_trait library that need to quote then to get the nested `::type` to get the result. 
Be wary of templated alias. `quote_<std::add_const_t>` and `lift_<std::add_const>` give the same answer but `quote_<std::add_const>`will result into `std::add_const<T>` because add_const_t is an alias.
Rule of thumb : use quote most of the time, use lift if you need to use std::type_traits meta-function

### te::mkseq

Similar to `std::make_sequence`, it create an `te::input_<std::integral_constant<int,0>, ... std::integral_constant<int, N-1>> ` depending on the integral_constant of N it receive.

### te::transform_<typename ... MetaExpressions>

This is one of my most used meta-expression. It apply the MetaExpressions to each types received. `te::eval_pipe_<te::input_<int,float>, te::transform_<te::quote_<std::add_pointer_t>,te::quote_<add_pointer_t>>>` will result in `input_<int**,float**>`

### te::fork_<typename ... MetaExpressions>

This is one of the most powerful meta-expression. All the received types are copied into each MetaExpression. Two example are needed to illustrate its behavior :
```C++
    te::eval_pipe_<te::input_<int>
                    ,te::fork_< te::quote_<std::add_const_t>
                                , te::quote_<std::add_pointer_t> 
                                >
                    ,te::same_as_<  const int
                                    , int*>
                    >{} = std::true_type{};
    te::eval_pipe_<te::input_<int,float>
                    ,te::fork_< te::second // get the second type. Could be written as get_<1>
								,te::pipe_<second,lift_<std::add_pointer>> // Get the second and add a pointer to it
					// pipe_<Es...> count as only one type, so you can use it to separate your function
								,te::push_back_<char> //push char at the end
                                ,te::push_front_<char*> // push char* at the start
                                ,te::input_<double> //replace int,float with double
                                >
                    ,same_as_<float,float*,te::input_<int,float,char>,te::input_<char*,int,float>,double>
                    >{} = std::true_type{};
```

### te::each_<typename ... MetaExpressions>

This meta-expression would be equivalent to writing `fork_<pipe_<get_<0>,...>,pipe_<get_<1>,...>>,...> ` but doing so require too much copying. There is a more efficient way to do it. The restriction is that you must provide exactly the same number of meta-expressions as you have types as inputs. Otherwise big compilation error.

```C++
	te::eval_pipe_<te::input_<int,float,char>
				,te::each_<identity,te::lift_<std::add_pointer>,lift_<std::add_const>>
				,te::same_as_<int,float*,const char>
	>{} = std::true_type{};
	static_assert(te::eval_pipe_<te::input_<int,float,char>
								,te::each_<te::identity,te::identity,te::identity>
								,te::same_as_<int,float,char>>::value
								,"The expression each_<identity...> is exactly the same as the original input.
								  This will become important later in the bind_<index,Es...> meta-expression");
```

### Let's show a comparison

This is the example at the start of this blog :
```C++
    // range library to double each int values in a container and calculate the summation 
    
	int accumulator = 0;
	auto fct_actions =
		ranges::actions::transform([](int& i){ return i*=2;} )
	|	ranges::actions::transform( [&accumulator](const int& i)
						{accumulator += i; return i;});

	std::vector<int> vi{1,2,3};
	vi |= fct_actions;
	assert(accumulator == 12);

	// type_expr library to do the same thing with std::integral_constant types
    template <int N> using int_c = std::integral_constant<int,N>;
    
    using type_expression = 
        te::pipe_<
                te::transform_<te::multiply_<int_c<2>>>
                ,te::fold_left_<te::plus_<>> // Very similar to std::accumulate()
        >; 
    using result_type = te::eval_pipe_<
                                te::input_<int_c<1>,int_c<2>,int_c<3>>, 
                                type_expression
                                >; 
	static_assert( std::is_same_v<int_c<12>, result_type>,"");
```

This is really the closest I can get to the range library. I took some liberties since there is certainly a better way to double each values in a vector and do the summation.

I mostly wanted to show that each meta-expression inside `eval_pipe_` is considered something akin to an `actions` in range_v3. Any `views` ( the other "concept" in range_v3 ) is not really possible in type meta-programming since everything we interact with is a type, which is by definition very const and so cannot be iterated over, at least not in an imperative way. 

While very cool, some operations are still difficult to express in meta-programming and, up until now, all of this was a by-product of what I wanted to achieve. Let me be clear : I did not "TMP all the things" in the range library. It's very similar due to some similar ideas and concepts, but nothing more.

# Let's open Pandora's box.

Let's create meta-expression out of other meta-expressions. Since most of my meta-expression are respecting the template signature of `template< typename ...>`, even `te::pipe_<typename ... Es>` , we can use `te::quote_<...>` inside `te::eval_pipe_<...>` to create new meta-expression.


```C++
    template<int N>
    struct copy_ : te::eval_pipe_< te::input_<int_c<N>>,
                                    te::mkseq,
                                    te::transform_<te::input_<te::identity>>, 
                                    te::quote_<te::fork_>>
                                    {}; 
    // all of this eval to fork_<identity,identity,... > with N-1 time identity
    // copy_<int 2> inherit from te::fork_<te::identity,te::identity>
    // te::eval_pipe_<te::input_<int>, copy_<5>> result into te::input_<int,int,int,int,int>
```
Technically, what we are doing here is modifying a function to be written as another one.

If this is not mind-blowing enough, let's me introduce you `te::repeat_<int N,typename ... Es>` which repeats the meta-expressions N time.

```C++
    template <std::size_t N, typename... Es>
    struct repeat_ : te::eval_pipe_<te::input_<Es...>, 
                                te::copy_<N>, 
                                te::flatten, 
                                te::quote_<te::pipe_>> {};

    static_assert(
        eval_pipe_<te::input_<int>,
                   te::repeat_< 2, te::lift_<std::add_pointer>, te::lift_<std::add_const>>,
                   te::same_as_<int *const *const>>::value, 
                "The result is int * const * const");
```

If you are familiar with kvasir.mpl documentations, there is a little meta-function named ``kvasir::mpl::call_f ``with the comment literally saying "experimental, may be deprecated or changed". This is because it consider its input as meta-function.  My library is basically an experiment of using this function all over the place.

I'm now able to implement some cool functions like `te::swizzle<int ... I>`

```C++
    template<int ... I>
    struct swizzle : te::fork_<te::get_<I>...>{};

    static_assert(
    te::eval_pipe_<
            te::input_<int, int *, int **, int ***>, 
            te::swizzle_<2, 1, 0, 3, 1>, // Get by index
            te::same_as_<int **, int *, int, int ***, int *>
    >::value, " Swizzling is easy");
```

The last example actually modified a function using variadic pack expansions. This is one of the reasons I wanted to start a library since other libraries made it too difficult to do such things due to the nesting of their meta-functions.

For now, I've already implemented two higher-meta-expression using `te::fork_` , but I'm not surprised one bit. Fork was the first meta-expression that blew my mind. I remember my big meta-functions that wasn't working, replacing it by fork and three little dot and seeing it work perfectly on the first try. I was flabbergasted the whole day. 

The last higher-meta-expression I want to talk about is `te::on_args_<typename ... Es>`. This solved a weird problem in template library where you have something like a `std::tuple<Ts...>` and just want to play with the inner types, then wrap with tuple. 

The problem is that it's technically a series of functions that depend on the input, which is notoriously difficult since you need to add your continuation to the last meta-function which can be nested inside other meta-function. Not in my library

```C++
    template <typename... Es>
struct on_args_ {
  template<typename ... Ts>
  struct f {};
  // Take the template arguments, evaluate the expression, then rewrap with the template template
  template <template <typename... Ts> class F, typename... Ts>
  struct f<F<Ts...>> {
    typedef eval_pipe_<input_<Ts...>, Es..., quote_<F>> type;
  };
};

```

This is my actual solution to Arthur O'Dwyer post about template library released in December 2019 : 

```C++
    // On this challenge, the goal was to unwrap, remove empty class, sort them by
    // size and rewrap them. on_args_<Es...> deals with the unwrap rewrap if the
    // signature of the type accept only types.
    struct Z {};  // EMPTY CLASS

    using MetaFct = te::on_args_<
                       te::remove_if_<te::lift_<std::is_empty>>,
                       te::sort_<te::transform_<te::size>, te::greater_<>> // Get their sizeof and compare if greater.
                       >;

    static_assert(
      te::eval_pipe_<
          te::input_<std::tuple<Z, int[4], Z, int[1], Z, int[2], int[3]>>,
                    MetaFct
                    te::same_as_<std::tuple<int[4], int[3], int[2], int[1]>>
        >::value, "Arthur O'Dwyer");
    // No unwrap, no as_lambda, no custom implementation. Simply on_args. 
```

Since we have abstracted the whole unwrap-wrap, might as well test this function with different types at the same time.

```C++
    static_assert(
      te::eval_pipe_<
          te::input_<
            std::tuple<Z, int[4], Z, int[1], Z, int[2], int[3]>,
            te::ls_<int[2], Z, int[1], Z, int[3]> // just a type_list named ls_
          >,
          te::transform_< MetaFct  >, // The same MetaFct as previous example.
          te::same_as_<
                std::tuple<int[4], int[3], int[2], int[1]>
                , te::ls_<int[3],int[2],int[1]>
              >
        >::value, "Arthur O'Dwyer but with multiple types");   
```

## Meta-Predicate definition

Some of my meta-expressions require some expression that are either unary of binary predicate. Something like 
```C++
    template<typename ... BinaryPredicate>
    struct sort_;
```

This needs a little explaining, but a meta-expression binary-predicate can be multiple expressions that take 2 types and return either `std::integral_constant<bool,false>` or `std::integral_constant<bool,true>`. Nothing else. I'm not converting into them nor getting the value, albeit I'm starting to evaluate if I should.
However, those three littles dot actually means that it's a series of function that end with either std::true_type or std::false_type. I take those BinaryPredicate, put it in another eval_pipe, look at the result and continue from there.

For example, let's say you have` te::sort_<transform_<te::size>, greater_<>>`. The sort meta-expression will give two types at a time to the sub-meta-expressions. The `te::size` will transform 1 type into their size in `std::integral_constant<int,sizeof(T)>` and `te::greater_<>`, without argument, will look at those two integer_constant of their size and "return" either `true_type or false_type` depending if the first is greater than the second.


## Binding with bind_<int index,UnaryFct...> and bind_on_args_<int index,UnaryFct...>

Not sure if anyone could correct me if "bind" is the correct term to use, but my goal for this suite of meta-expressions was to have a range of type, and modify only the selected one depending on the index. 
Two general usecases seems to exist : Only modify a type at index N, and modify a type depending on the other types. Both are implemented using the technique of creating another meta-expression depending on the inputs. Both use a circular signed index using modulo, so you can write -1 to get the last one, -2 to get the one before that, etc.

`bind_<int index,UnaryFct...> ` will internaly give you the type at the index provided. 
`bind_on_args_<int index,UnaryFct...>` will internaly give you all the types that you provided it.

```C++
using namespace te;
static_assert(
    eval_pipe_<input_<int, float, char>, bind_<0, lift_<std::add_pointer>>,
               same_as_<int *, float, char>>::value, "Add pointer to position 0, which is the first");
static_assert(
    eval_pipe_<input_<int, float, char>, bind_<-1, lift_<std::add_pointer>>,
               same_as_<int, float, char *>>::value,"Same thing but on the last position");
static_assert(
    eval_pipe_<input_<i<1>, i<2>, i<3>>, bind_on_args_<-1,first>
			  ,same_as_<i<1>, i<2>, i<1>>>::value,"Put at the last position the type of the first position");
static_assert(
    eval_pipe_<input_<i<1>, i<2>, i<3>>, bind_on_args_<-1, fold_left_<plus_<>>/*Again, this is similar to accumulate*/>
			  ,same_as_<i<1>, i<2>, i<6>>>::value," Put at the last position the sum of all");
```

For the implementation, remember when I said that `te::each_<te::identity...>` is the same as the original input ? My only concern for the `bind_<index,Es...>` meta-expression was to replace the identity at the index position with `pipe_<Es...>`. So I use `te::mkseq` with the int_c of lenght of Ts..., and I do a `te::transform_<te::cond_<same_as<int_c<index>>,input_<pipe_<Es...>>,input_<te::identity>>>`  which is basically tell that for each type, if it's the same as the int_c of the index, replace it with `pipe_<Es...>` and if not, then replace if with `te::identity`. Once this is done, wrap it all in `te::each_` and then evaluate this expression with the original input.
For bind_on_args, the only difference is that you replace `te::pipe_<te::input_<Ts...>,Es...>` at the selected index.


## Error Handling, or something close

Also, I have this feature that is not present on every function since I've recently implemented it, but I actually do some sort of error management. However, it's a little bit unconventional

Let's say you want to unwrap an `int`, which is not possible since `int ` is not a template template.
`te::eval_pipe_<te::input_<int>, te::unwrap> test_error = ?`

Without any error management, you would have a huge compiler message with nested error reporting. I was as tired of seeing this as you do. What I did was, since a lot of my code is centralized into what I call a context, if I know an error is produced in this context, I can just create a type named `te::error_<>` and put whatever message I want inside in the form of a type.

In the case of unwrapping an `int`, I return something like `error_<te::unwrap,te::unspecialized, int>` because I don't have a specialization for unwrapping an int. The _very_ cool thing is that further evaluation of meta-expressions will do _nothing_ once it saw that an `te::error_<>` has been evaluated. You then receive a type `error_` with some custom message explaining what is the problem. It's not perfect, if you have multiple types and one of them is an error type, then there is some possibility that I can continue with that, but it's a cool feature.

Albeit weird, this system allow me to greatly reduce the compiler log size, which is my goal in 99% of the case. Not all functions support it yet since it's fairly new.



## What's Next

I'm very satisfied with this library. In truth, I don't feel it's ready for production since there are some inconsistencies in the meta-expressions calling, but nothing that can't be fixed with a decent naming scheme. For performance concerns, most of the implementation is very similar to kvasir.mpl, which has the best compile-time performance. There is some algorithms where I simply cannot see an improvement. 
I do however feel like I want to improve how I'm doing the implementation in case of if statement.


