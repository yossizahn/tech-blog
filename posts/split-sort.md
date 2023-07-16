---
title: “Split sort” with some bit twiddling
description: “Split sort” with some bit twiddling
date: 2023-07-16
tags:
  - bit twiddling
  - programming
layout: layouts/post.njk
---

## What is a “split sort”
It all started with an innocent question posted on a small chat group. The poster wanted to sort a list of numbers in ascending order but with a twist: the ascension should start from a specific cutoff point, after reaching the highest number, the list should go back and list from the lowest number up to the cutoff point.
I will call this “split sort” for lack of a better term.

Some code to demonstrate:
```js
const list = [1,10,6,8,9,4,3,5,2,7]

const sortFn = (cutoff) => (a, b) => { ... }

list.sort(sortFn(5)) // [5,6,7,8,9,10,1,2,3,4]
```

The question is how to implement `sortFn`.

## The simple solution
After some consideration most people would come up with code similar to this:
```js
const sortFn = (cutoff) => (a, b) => {
  if (a >= cutoff && b >= cutoff || a < cutoff && b < cutoff) {
    return a - b
  } else {
    return b - a
  }
}
```
*In words:*
> If both numbers are either before of after the cutoff point -> sort ascending, otherwise, i.e. the 2 numbers live on differing sides of the cutoff point -> sort descending.

This is not too bad. Fairly elegant.
But it it got me thinking: The normal numerical sort can be done with a mathematical expression with no conditionals, can “split sort” also be done without using any conditionals? 🤔

## The “clever” solution
After some thought I came up with this solution I would like to share.

```js
const sortFn = (cutoff) =>
  (a, b) =>
    (a - cutoff ^ 1 << 31) - (b - cutoff ^ 1 << 31)
```

## How does it work?
I first started out by trying to simplify the problem. I came up with the following:

Let us deduct `cutoff` from each number in the list. This will give us a list where all numbers above `cutoff` are positive and all numbers below are negative.
This allows us to rephrase the problem as follows:
> Sort the list in ascending order, but consider all negatives to be of higher value than positives.

This rephrase lit up a light bulb in my head 💡.

## 2's complement
 As developers we all have run into the situation where a signed integer overflows and suddenly your very large positive number becomes a very low negative number 🤕.
e.g. adding 1 to the maximum value that an `int32` can hold (2147483647) will return -2147483648. This is due to the way modern computers store signed numbers using the [Two’s complement](https://en.wikipedia.org/wiki/Two's_complement)) system.

Let us look at the bit layout of signed integers using the 2’s complement system. For simplicity we will look at a single byte number (I am assuming the reader is familiar with the basic binary counting system so I won’t explaining how bit patterns represent values):
* Zero value: `00000000`
* Lowest positive number: `00000001`. The zeroed out leftmost bit (the “sign bit”) is a flag that tells us that this is a positive number. The seven other bits represent the value which is `1` in this case.
* Highest positive number: `01111111` . The zeroed out leftmost bit (the “sign bit”) is a flag that tells us that this is still a positive number, the seven remaining bits represent the number `127`, the highest number a signed byte can represent.
* Lowest negative number: `10000000`. The first bit is set to `1` to signify a negative number. The other bits are set to zero, this represents the lowest negative number a signed byte can represent: `-128` (the lowest negative is always equal to the highest positive + `1`). Each successive bit pattern represents the next negative number until we climb back up to zero. The fact the zero lives on the positive side of the fence allows 1 more negative magnitude than positive.

### What does all this mean for us?
We all know that negative numbers have a lower value than positive numbers, and so does the computer’s processor. But… the exact same bits can also represent an _unsigned number_ and then... the previously low negative numbers will suddenly become high value positive numbers 😲. The computer’s CPU also knows that. The CPU knows how to do signed math and unsigned math.

In other words: (when peering through “unsigned glasses 😎”) negative numbers begin from where positive numbers end…
### Casting attempts
That leads us to the following thought:
> How about if we convince the processor to look at our numbers as unsigned integers just for the purpose of determining the order?

Attempting this is not really possible in Javascript. The language doesn’t give us the facilities to tell the processor how to interpret a number.
Would it work in another language? Say C#?

#### Attempt 1:
```csharp
var arr = Enumerable.Range(1, 10)
    .Select(_ => new Random().Next(10))
    .ToList();

Comparison<int> comparison(int startFrom)
{
	return (a, b) =>
	{
		var aPrime = (uint)(a - startFrom);
		var bPrime = (uint)(b - startFrom);
		return aPrime - bPrime;
// ❗CS0266 Cannot implicitly convert type 'uint' to 'int'.
// An explicit conversion exists (are you missing a cast?)
	};
}

arr.Sort(comparison(5));
```

Uh oh. `Comparison` needs to return an `int`. Obviously 🤦. It needs to have a way of returning `-1`.
Ok, let’s cast the result back to an `int`:

#### Attempt 2:
```csharp
Comparison<int> comparison(int startFrom)
{
	return (a, b) =>
	{
		var aPrime = (uint)(a - startFrom);
		var bPrime = (uint)(b - startFrom);
		return (int)(aPrime - bPrime);
	};
}
```
This looks like it should work. I still doesn’t. After some ~~thought~~ debugging the obvious answer presents itself: `a` and `b` are _unsigned_. If `a - b` results in a negative number we have an integer underflow…
Ok, let’s recast back to an `int` after interpreting the bits as a `uint`.
#### Attempt 3:
```csharp
Comparison<int> comparison(int startFrom)
{
	return (a, b) =>
	{
		var aPrime = (int)(uint)(a - startFrom);
		var bPrime = (int)(uint)(b - startFrom);
		return aPrime - bPrime;
	};
}
```

Nope. A large positive `uint` (which is what happens to our negative numbers) will revert back to negative because of that pesky sign bit.

How about if we recast to a `long`? That should work.. 🤔
#### Attempt 4:
```csharp
Comparison<int> comparison(int startFrom)
{
	return (a, b) =>
	{
		var aPrime = (long)(uint)(a - startFrom);
		var bPrime = (long)(uint)(b - startFrom);
		return (int)(aPrime - bPrime);
	};
}
```
Ah, but the sort function needs to return an `int`. Now that the result of the subtraction is a `long` , if we cast the answer back to an `int` we loose the sign bit.
In short: I couldn’t get this path to work without adding a conditional which defeats the whole point of the exercise.

## Transposing
Here’s a different idea:
If we flip the sign bit of the original numbers, we can transpose a negative number into the positive realm and vice versa for positive numbers. Note: This is _transposition_ not _negation_ i.e. the number moves 2147483648 positions up or down the scale, without preserving its _magnitude_ (1 becomes -2147483647).


### Javascript’s `Number`s
In Javascript _all_ numbers are represented as 64 bit floating point numbers. However, when doing binary bitwise operations the number is first coerced to a 32 bit integer.
The coercion will preserve the valence (“sign”) but truncate the the value to 32 bits.
After the operation, the resulting bits are converted back into 64 bit floating point numbers again preserving the valence.
All mathematical operations operate on the resulting 64 bit number.

### The solution
Let us exploit this to solve our problem. To recap: We want to sort numbers in ascending order, but considering all negative numbers as higher valued than positive numbers.

To do this I came up with the above method. Let us step through the code:
* `a - cutoff` - deduct the value of `cutoff` from the original number.
* `1 << 31` - create a 32 bit pattern (mask) where only the leftmost bit is set to one (remember: bitwise operations operate on 32 bits).
* Take the right and left values and smush them together (that’s a technical term 😉) using the XOR operator. This has the effect of flipping the sign bit while leaving the other bits alone.
* Do this to both `a` and `b`
* Do a mathematical operation (minus) on the numbers, this will coerce the bit pattern back into a number, _respecting the sign bit_
Having flipped the sign bit, this will transpose a negative number from the negative realm to the positive realm and vice versa for positive numbers.
* Seeing as the previously negative numbers now have a higher value than the previous positive numbers, we can go ahead and use the regular `a - b`

### In C#
Here is a slightly messier C# version:
```csharp
Comparison<int> comparison(int startFrom)
{
	return (a, b) =>
	{
		return (int)((long)(a - startFrom ^ 1 << 31) -
      (long)(b - startFrom ^ 1 << 31) >> 1);
	};
}
```

The end :)
