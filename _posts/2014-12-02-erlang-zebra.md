---
layout: post
title:  "斑马谜题的 Erlang 求解"
date:  2014-12-09 17:03:31 
categories: Erlang
---

[斑马谜题](http://en.wikipedia.org/wiki/Zebra_Puzzle)是个典型的
[约束满足问题(CSPs)](http://en.wikipedia.org/wiki/Constraint_satisfaction_problem).
数独, 幻方也属于 CSPs 问题.
Erlang 的列表解析等语言特性让它能够以一种非常贴近自然语言的描述方式来解决这类问题:
不需要考虑过程式的求解步骤, 描述好约束条件, Erlang 解释器就通过域搜索匹配出了答案.
有人还用 Erlang 写了一个对于[有限域约束(Finite Domain Constraints)](http://www.math.unipd.it/~frossi/SchulteCarlsson_CPH_2006.pdf)
的一个[扩展](http://www.erlang.se/publications/xjobb/finite-domain-erlang.ps.gz).

用 Erlang 的列表解析生成所有三阶幻方是如此简单:

```erl
%% magicsquare.erl

-module(magicsquare).
-export([magicsquare/0]).

magicsquare() ->
    [ {A00, A01, A02, A10, A11, A12, A20, A21, A22} ||
      A00 <- lists:seq(1, 9),
      A01 <- lists:seq(1, 9) -- [A00],
      A02 <- lists:seq(1, 9) -- [A00, A01],
      A10 <- lists:seq(1, 9) -- [A00, A01, A02],
      A11 <- lists:seq(1, 9) -- [A00, A01, A02, A10],
      A12 <- lists:seq(1, 9) -- [A00, A01, A02, A10, A11],
      A20 <- lists:seq(1, 9) -- [A00, A01, A02, A10, A11, A12],
      A21 <- lists:seq(1, 9) -- [A00, A01, A02, A10, A11, A12, A20],
      A22 <- lists:seq(1, 9) -- [A00, A01, A02, A10, A11, A12, A20, A21],

      %% 每一行的和都为 15
      A00 + A01 + A02 =:= 15,
      A10 + A11 + A12 =:= 15,
      A20 + A21 + A22 =:= 15,

      %% 每一列的和都为 15
      A00 + A10 + A20 =:= 15,
      A01 + A11 + A21 =:= 15,
      A02 + A12 + A22 =:= 15,

      %% 对角线的和为 15
      A00 + A11 + A22 =:= 15,
      A02 + A11 + A20 =:= 15
    ].
```

既然著名的斑马谜题和幻方是同样类型的问题, 按理说用 Erlang 写个求解程序也不是个难事.
先把斑马问题贴出来:

> Five men of different nationality (England, Spain, Japan, Italy, Norway) live in the
> first five houses on a street. They all each have a profession (painter, diplomat,
> violinist, doctor, sculptor), one animal (dog, zebra, fox, snail, horse), and one favorite
> drink (juice, water, tea, coffee, milk), all different from the others. Each of the houses
> is painted in a color different from all the others (green, red, yellow, blue, white).
> 
> Furthermore:
> 
> 1. The Englishman lives in the red house.
> 2. The Spaniard owns the dog.
> 3. The Japanese is the painter.
> 4. The Italian likes tea.
> 5. The Norwegian lives in the leftmost house.
> 6. The owner of the green house likes coffee.
> 7. The green house is to the right of the white one.
> 8. The sculptor breeds snails.
> 9. The diplomat lives in the yellow house.
> 10. Milk is drunk in the third house.
> 11. The Norwegian's house is next to the blue one.
> 12. The violinist likes juice.
> 13. The fox is in the house next to the doctor's house.
> 14. The horse is in the house next to the diplomat's.
> 
> The problem is thus to infer who owns the zebra and who drinks water.

为方便程序描述问题, 先为 *国籍*, *职业*, *颜色*, *宠物*, *饮料* 这五项的内容映射到 1~5 这几个数字去:

| House: left to right| 1  | 2  | 3  | 4  | 5  |
| ---- | -- | -- | -- | -- | -- |
| Nation | England | Spain | Japan | Italy | Norway |
| Profession | painter| diplomat| violinist| doctor| sculptor |
| Color | green | red| yellow| blue| white |
| Animal | dog | zebra| fox| snail| horse |
| Drink | juice| water| tea| coffee| milk |

然后就可以用 Erlang 描述问题了:

```erl
%% zebra.erl version 1
-module(zebra).
-compile(export_all).

perms([]) -> [[]];
perms(L)  -> [[H|T] || H <- L, T <- perms(L--[H])].

full_perms(N) ->
    perms(lists:seq(1, N)).

zebra() ->
    [ {N, C, P, A, D} ||
            N <- full_perms(5),
            C <- full_perms(5),
            P <- full_perms(5),
            A <- full_perms(5),
            D <- full_perms(5),
            lists:nth(1, N) =:= lists:nth(2, C), % 条件1
            lists:nth(2, N) =:= lists:nth(1, A), % 条件2
            lists:nth(3, N) =:= lists:nth(1, P), % 条件3
            lists:nth(4, N) =:= lists:nth(3, D), % 条件4
            lists:nth(5, N) =:= 1, % 条件5
            lists:nth(1, C) =:= lists:nth(4, D), % 条件6
            lists:nth(5, C) + 1 =:= lists:nth(1, C), % 条件7
            lists:nth(5, P) =:= lists:nth(4, A), % 条件8
            lists:nth(2, P) =:= lists:nth(3, C), % 条件9
            lists:nth(5, D) =:= 3, % 条件10
            (((lists:nth(5, N) + 1) =:= lists:nth(4, C)) orelse ((lists:nth(4, C) + 1) =:= lists:nth(5, N))), % 条件11
            lists:nth(3, P) =:= lists:nth(1, D), % 条件12
            (((lists:nth(3, A) + 1) =:= lists:nth(4, P)) orelse ((lists:nth(4, P) + 1) =:= lists:nth(3, A))), % 条件13
            (((lists:nth(5, A) + 1) =:= lists:nth(2, P)) orelse ((lists:nth(2, P) + 1) =:= lists:nth(5, A))) % 条件14
    ].
```

编译运行:
```console
$ erl
1> c(zebra).
{ok, zebra}.
2> zebra:zebra().
[{[3,4,5,2,1],
  [5,3,1,2,4],
  [5,1,4,2,3],
  [4,5,1,3,2],
  [4,1,2,5,3]}]
```

不错, 寥寥二三十行代码(大部分是在阐述约束条件)可以求解出正确答案了. 可是它运行得非常慢, 在我的机器上大概花了一两个小时! 而且得出的答案还需要人工将数字映射到对应的属性去. 

先来解决运行慢的问题:
还记得[计算机程序的构造和解释](http://book.douban.com/subject/1148282/)[*SICP*](http://mitpress.mit.edu/sicp/)[练习4.39](http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-28.html#%_thm_4.39)这个问题吗? 约束条件的顺序不会影响答案, 但会影响程序执行时搜索范围的大小, 导致不同的约束条件的顺序运行的时间相差巨大. 把约束性越强的条件放在前面, 将显著减少搜索范围, 降低运行时间. 调整下上面的zebra求解代码, 就瞬间解出了答案:

```erl
%% zebra.erl version2
-module(zebra).
-compile(export_all).

perms([]) -> [[]];
perms(L)  -> [[H|T] || H <- L, T <- perms(L--[H])].

full_perms(N) ->
    perms(lists:seq(1, N)).

zebra() ->
    [ {N, C, P, A, D} ||
            N <- full_perms(5),
            lists:nth(5, N) =:= 1, % 条件5
            C <- full_perms(5),
            (((lists:nth(5, N) + 1) =:= lists:nth(4, C)) orelse ((lists:nth(4, C) + 1) =:= lists:nth(5, N))), % 条件11
            lists:nth(1, N) =:= lists:nth(2, C), % 条件1
            lists:nth(5, C) + 1 =:= lists:nth(1, C), % 条件7
            P <- full_perms(5),
            lists:nth(3, N) =:= lists:nth(1, P), % 条件3
            lists:nth(2, P) =:= lists:nth(3, C), % 条件9
            A <- full_perms(5),
            lists:nth(2, N) =:= lists:nth(1, A), % 条件2
            lists:nth(5, P) =:= lists:nth(4, A), % 条件8
            (((lists:nth(3, A) + 1) =:= lists:nth(4, P)) orelse ((lists:nth(4, P) + 1) =:= lists:nth(3, A))), % 条件13
            (((lists:nth(5, A) + 1) =:= lists:nth(2, P)) orelse ((lists:nth(2, P) + 1) =:= lists:nth(5, A))), % 条件14
            D <- full_perms(5),
            lists:nth(5, D) =:= 3, % 条件10
            lists:nth(4, N) =:= lists:nth(3, D), % 条件4
            lists:nth(1, C) =:= lists:nth(4, D), % 条件6
            lists:nth(3, P) =:= lists:nth(1, D) % 条件12
    ].
```

解决了性能问题, 再来完善一下程序, 在程序里面映射好各种属性, 这样就能更好地描述约束条件了. 完整的 Erlang 代码如下:

```erl
%% zebra.erl final version
-module(zebra).
-export([zebra/0, zebra_print/0, who_own_zebra/0]).

%% @spec (List::list(), Ele) -> integer()
%% @doc Returns the position of `Ele' in the `List'. 0 is returned
%%      when `Ele' is not found.
%% @end
pos(List, Ele) ->
     pos(List, Ele, 1).
pos([Ele | _Tail], Ele, Pos) ->
     Pos;
pos([_ | Tail], Ele, Pos) ->
     pos(Tail, Ele, Pos+1);
pos([], _Ele, _) ->
     0.

nation() -> ['England', 'Spain', 'Japan', 'Italy', 'Norway'].
profession() -> [painter, diplomat, violinist, doctor, sculptor].
color() -> [green, red, yellow, blue, white].
animal() -> [dog, zebra, fox, snail, horse].
drink() -> [juice, water, tea, coffee, milk].

data_inx(N, Data) ->
    lists:nth(N, Data()).

data_pos(Name, Data) ->
    pos(Data(), Name).

perms([]) -> [[]];
perms(L)  -> [[H|T] || H <- L, T <- perms(L--[H])].

full_perms(N) ->
    perms(lists:seq(1, N)).

zebra_print() ->
    [io:format("Nation:\t\t~-12s~-12s~-12s~-12s~-12s~n"
                "Color:\t\t~-12s~-12s~-12s~-12s~-12s~n"
                "Profession:\t~-12s~-12s~-12s~-12s~-12s~n"
                "Animal:\t\t~-12s~-12s~-12s~-12s~-12s~n"
                "Drink:\t\t~-12s~-12s~-12s~-12s~-12s~n~n",
               [data_inx(pos(N, 1), fun nation/0),
                data_inx(pos(N, 2), fun nation/0),
                data_inx(pos(N, 3), fun nation/0),
                data_inx(pos(N, 4), fun nation/0),
                data_inx(pos(N, 5), fun nation/0),

                data_inx(pos(C, 1), fun color/0),
                data_inx(pos(C, 2), fun color/0),
                data_inx(pos(C, 3), fun color/0),
                data_inx(pos(C, 4), fun color/0),
                data_inx(pos(C, 5), fun color/0),

                data_inx(pos(P, 1), fun profession/0),
                data_inx(pos(P, 2), fun profession/0),
                data_inx(pos(P, 3), fun profession/0),
                data_inx(pos(P, 4), fun profession/0),
                data_inx(pos(P, 5), fun profession/0),

                data_inx(pos(A, 1), fun animal/0),
                data_inx(pos(A, 2), fun animal/0),
                data_inx(pos(A, 3), fun animal/0),
                data_inx(pos(A, 4), fun animal/0),
                data_inx(pos(A, 5), fun animal/0),

                data_inx(pos(D, 1), fun drink/0),
                data_inx(pos(D, 2), fun drink/0),
                data_inx(pos(D, 3), fun drink/0),
                data_inx(pos(D, 4), fun drink/0),
                data_inx(pos(D, 5), fun drink/0)]) ||
                                    {N, C, P, A, D} <- zebra()
    ].

who_own_zebra() ->
    [data_inx(pos(N, pos(A, data_pos(zebra, fun animal/0))), fun nation/0) ||
        {N, _C, _P, A, _D} <- zebra()
    ].

zebra() ->
    [{N, C, P, A, D} ||
            N <- full_perms(5),

            %% The Norwegian lives in the leftmost house
            lists:nth(data_pos('Norway', fun nation/0), N) =:= 1,

            C <- full_perms(5),

            %% The Norwegian's house is next to the blue one
            (((lists:nth(data_pos('Norway', fun nation/0), N) + 1) =:= lists:nth(data_pos(blue, fun color/0), C)) orelse ((lists:nth(4, C) + 1) =:= lists:nth(5, N))),

            %% The Englishman lives in the red house
            lists:nth(data_pos('England', fun nation/0), N) =:= lists:nth(data_pos(red, fun color/0), C),

            %% The green house is to the right of the white one
            lists:nth(data_pos(white, fun color/0), C) + 1 =:= lists:nth(data_pos(green, fun color/0), C),

            P <- full_perms(5),

            %% The Japanese is the painter
            lists:nth(data_pos('Japan', fun nation/0), N) =:= lists:nth(data_pos(painter, fun profession/0), P),

            A <- full_perms(5),

            %% The Spaniard owns the dog
            lists:nth(data_pos('Spain', fun nation/0), N) =:= lists:nth(data_pos(dog, fun animal/0), A),

            %% The sculptor breeds snails
            lists:nth(data_pos(sculptor, fun profession/0), P) =:= lists:nth(data_pos(snail, fun animal/0), A),

            D <- full_perms(5),

            %% The diplomat lives in the yellow house
            lists:nth(data_pos(diplomat, fun profession/0), P) =:= lists:nth(data_pos(yellow, fun color/0), C),

            %% The fox is in the house next to the doctor's house
            (((lists:nth(data_pos(fox, fun animal/0), A) + 1) =:= lists:nth(data_pos(doctor, fun profession/0), P)) orelse ((lists:nth(data_pos(doctor, fun profession/0), P) + 1) =:= lists:nth(data_pos(fox, fun animal/0), A))),

            %% The horse is in the house next to the diplomat's
            (((lists:nth(data_pos(horse, fun animal/0), A) + 1) =:= lists:nth(data_pos(diplomat, fun profession/0), P)) orelse ((lists:nth(data_pos(diplomat, fun profession/0), P) + 1) =:= lists:nth(data_pos(horse, fun animal/0), A))),

            %% Milk is drunk in the third house
            lists:nth(data_pos(milk, fun drink/0), D) =:= 3,

            %% The Italian likes tea
            lists:nth(data_pos('Italy', fun nation/0), N) =:= lists:nth(data_pos(tea, fun drink/0), D),

            %% The owner of the green house likes coffee
            lists:nth(data_pos(green, fun color/0), C) =:= lists:nth(data_pos(coffee, fun drink/0), D),

            %% The violinist likes juice
            lists:nth(data_pos(violinist, fun profession/0), P) =:= lists:nth(data_pos(juice, fun drink/0), D)
    ].

```

编译运行:

```console
$ erl
>1 c(zebra).
{ok, zebra}
>2 zebra:zebra_print().
Nation:         Norway      Italy       England     Spain       Japan       
Color:          yellow      blue        red         white       green       
Profession:     diplomat    doctor      sculptor    violinist   painter     
Animal:         fox         horse       snail       dog         zebra       
Drink:          water       tea         milk        juice       coffee  

[ok]
>3 zebra:who_own_zebra().
['Japan']
```
