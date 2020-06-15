---
layout: post
title: How we use Dynamic Programming to find the best price for our customers.
date: 2016-09-18
type: post
published: true
status: publish
comments: true
categories:
- Technology
tags:
- programming
meta:
  _edit_last: '1'
author:
  login: jake
  email: jakewins@gmail.com
  display_name: jake
  first_name: ''
  last_name: ''
redirect_to: 'https://tech.davis-hansson.com/p/price-wars.html'
---

A friend of mine, I won't share his name for privacy reasons<sup>1</sup>, told me he had an interesting realization one day.
Thinking back to the projects he'd done for big banks before his current job, each and every one of his software algorithms had
eventually ended up in The Guardian under a headline like "Giant Bank Inc. indicted for massive automated customer fraud system, AGAIN!".

<!--more-->

For instance, one of his assignments was to write an algorithm that would re-order a customers transactions, so that they would incur
as many penalties as possible.

You might've heard of this before, but breifly, imagine you do the following things, in the following order:

- Notice your bank account has a $0 balance
- Deposit $100
- Buy a $5 coffee
- Buy a $1 chewing gum
- Buy $20 worth of gas

<b>End balance: $74, Bank fees: $0</b>

What the banks systems would do was to look at these, and try to come up with a way to re-order them. In this case, if they pretend
that the deposit got delayed somehow, and happened at the very end, then each of the three transactions would incur an overdraft fee:

- Notice your bank account has a $0 balance
- Buy a $5 coffee ($20 overdraft fee)
- Buy a $1 chewing gum ($20 overdraft fee)
- Buy $20 worth of gas ($20 overdraft fee)
- Deposit $100

<b>End balance: $14, Bank fees: $60</b>

## Morality aside

I don't know which algorithm they used for this - but it struck me the other day that we solved the exact opposite problem at EquipmentShare (ES), where I work.

ES provides a platform for people who own various giant machines, helping them optimize their utilization.
One of the things we do is allow owners to rent equipment that they themselves are not currently using.

One of my first assignments at ES was to write an algorithm that'd take a price list of daily, weekly and monthly rental rates, as well as a period
that the user needed the equipment, and figure out what combination of days, weeks and months from the price list would give the customer the lowest possible price.

## An algorithm for finding the best combination of things

The rest of this post will show you a generic algorithm for solving this kind of problem, and many, many more like it.
The algorithm is called "Dynamic Programming".

Lets make this a bit more concrete, and say that we have a price list where each option covers some set of days and costs some
dollar amount:

    PriceOption = namedtuple('PriceOption', 'name days_covered cost')

    pricelist = (
        PriceOption(name='Daily rate', days_covered=1, cost=10),
        PriceOption(name='Weekly rate', days_covered= 7, cost=45),
        PriceOption(name='Monthly rate', days_covered=28, cost=120),
    )

If I rent for a day, I'd want to pay the daily rate - $10. But what if I want to rent for a week and a day?

I could either pay the daily rate eight times, $80, or I could pay for two weeks, $45 + $45 = $90, or I could pay the weekly rate, plus one day, so $45 + $10 = $55. This gets more
complicated as the time period increases.
Our algorithms job is to always find the best price for the customer.

One straight-forward way to do it is to come up with every possible combination, and then choose the cheapest one.

For renting one day day, the possible (sensible) options are:

- Pay for one day ($10)
- Pay for a week ($45)
- Pay for a month ($120)

For renting two days, it expands:

- Pay for one day ($10) and;
  - Pay for one more day (+$10 = $20)
  - Pay for a week (+$45 = $55)
  - Pay for a month (+$120 = $130)
- Pay for a week ($45)
- Pay for a month ($120)

We can express this as a recursive function, something like:


    def combinations(pricelist, number_of_days):
        for price_option in pricelist:
            days_left = number_of_days - price_option.days_covered

            if days_left > 0:
                # This price option alone is not enough, it only covers
                # part of the rental period. Imagine the user is renting
                # for two days, and the option we're looking at only 
                # pays for one.
                for combo, cost in combinations(pricelist, days_left):
                    combo = [price_option.name] + combo
                    cost = price_option.cost + cost
                    yield combo, cost
            else:
                yield ([price_option.name], price_option.cost)


Aside: If you're not familiar with Pythons "yield" syntax, it's basically a way to create a function that returns ("yields") a stream of results, instead
of returning a single result, that's why the `for combo, cost in combinations(..)` loop works, it'll loop through each item the recursive
call yields.

This function will look at each option in the price list, see if it covers the number of days left to cover; if it does, it'll simply `yield` that option.
If it does not, it'll do a recursive call to find all possible ways to cover the remaining days and yield each of those combined with the current option.

For `days=2`, the combinations that come out look like:

    (['Daily rate', 'Daily rate'], 20)
    (['Daily rate', 'Weekly rate'], 55)
    (['Daily rate', 'Monthly rate'], 130)
    (['Weekly rate'], 45)
    (['Monthly rate'], 120)

Note how that list is the same as the bullet points I drew up above the code example, so it seems like our function is working.

## Finding the cheapest variant

To figure out what the cheapest option is, all we do is go through the list and pick the cheapest one!
This can be built into the function call, so rather than return all possible combinations, we write our function to return only the cheapest one.

Here is our refactored function:

    def cheapest(pricelist, number_of_days):
        cheapest_cost = None
        cheapest_combo = None

        for price_option in pricelist:
            days_left = number_of_days - price_option.days_covered
            cost = price_option.cost

            if days_left > 0:
                combo, combo_cost = cheapest(pricelist, days_left)
                cost = cost + combo_cost
                combo = [price_option.name] + combo_cost

            if cheapest_cost is None or cheapest_cost > cost:
                cheapest_cost = cost
                cheapest_combo = combo

        return cheapest_combo, cheapest_cost

Doing this means we save a lot of memory space - we won't end up holding every possible combination in RAM at the end, just the cheapest one.
However, under the hood, it's still exploring every possible combination and, as you can imagine, the number of combinations grows exponentially with
the number of days we're trying to find a good price for. 

So, we won't run out of memory - but the universe may end before our calculation completes.

A 10-day rental has 33 alternatives, 30 day rental has 3253 alternatives and at 60 days it takes my laptop
several minutes to come up with the result.
I've not dared venture past 60.

This is where the "dynamic" part of the algorithm comes in.

## Dynamic programming

Having an algorithm that finds the cheapest price is great - but EquipmentShare does multi-year rentals, and asking a customer to stay on the line
for four million years while we calculate the best price is a hard sell.

One important thing to note is that our algorithm is super wasteful - when figuring out, say, the cheapest option for a 7-day rental, it will calculate
the cheapest price for a 2-day rental (for instance) over and over and over. 
 
What if we add a cache? Whenever we've calculated the cheapest way to combine prices to cover a period, we jot that down somewhere. 
Next time the algorithm asks for the cheapest 1-day or 2-day combo, or whatever it may be, we just return our saved value.

This is called "memoization" - remember the result of a function call, given some set of arguments, and python has a built-in decorator for it:


    from functools import lru_cache

    @lru_cache()
    def cheapest(pricelist, number_of_days):
        .. # This stays the same as the original implementation of cheapest(..)

The difference this makes is insane, calculating the cheapest combination for a multi-year rental went from "will finish around the time the sun dies" to
a few microseconds, good times.

## Wrap-up

If you find yourself with a problem that requires choosing a combination of things to minimize (or maximize! ahem.) some cost, this algo is a good
one to keep in mind. 

EquipmentShare is still a young startup, but if we keep growing as we are currently our optimization algorithms will be
managing the majority of the United States heavy equipment in the next few years.
If you are interested in problems like the one in this post, geospatial systems and IoT, and you want to produce software that can have a 
tremendous positive impact on how we use the planets limited resources, we are hiring - [devjobs@equipmentshare.com](mailto:devjobs@equipmentshare.com).


PPS: If you think this algorithm is cool, you should have a look at IDP, a version that lets you not only trade memory/speed by setting the cache size,
but also accuracy, letting you say things like "Give me the best answer you can come up with in 30 seconds using no more than 100MB of RAM". That's part
of the secret sauce in how the worlds best database, [Neo4j](http://neo4j.com) cracks the exponential complexity of query planning for graph data.

<sup>1</sup> It was Alistair, hi Alistair!

