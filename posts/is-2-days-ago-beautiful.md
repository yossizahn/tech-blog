---
title: Is `2.days.ago` a beautiful way of expressing a relative date?
description: Is `2.days.ago` a beautiful way of expressing a date?
date: 2021-03-14
tags:
  - coding
layout: layouts/post.njk
---

A common scenario: You need to calculate a date as a value relative to the current date.

Using the popular (albeit [somewhat deprecated](https://momentjs.com/docs/#/-project-status/)) [moment.js](https://momentjs.com/), this can be expressed as follows:

```javascript
const twoDaysAgo = moment().subtract(2, 'days')
```
Nice. Readable. Sufficient.

I came across this curious Ruby snippet recently:
```ruby
user = User.create(
  email: 'user@example.com',
  trial_ends_at: 30.days.from_now
)
```
Ruby on Rails users may find this code somewhat familiar. For me this came as a big surprise.

After the initial double take, I tried understanding how this code actually works.
*There is no way that Ruby-core offers `days` as a method on integers. That would just be too ludicrous to imagine.*

Turns out, that Ruby allows (encourages?) extending native types, *and* this seems to be a popular practice. Whereas in JS land - where I live - it is generally frowned upon, in Ruby (on Rails?) it seems to be the norm to extend native types. There is even a utility library specifically for this purpose named [Active Support](https://guides.rubyonrails.org/active_support_core_extensions.html).

So... the `days` extension-method of integers, returns a `Duration` object. And duration objects apparently have an `ago` method, which returns a `Date` which is offset `$duration` amount of time into the past. Exhilarating? Beautiful?

Listen to what DHH (David Heinemeier Hansson, the creator of RoR) [has to say](https://rubyonrails.org/doctrine/#optimize-for-programmer-happiness:~:text=But%20both%20decisions%20still%2C%20to%20this,Rails%20after%2012%20years%20of%20service.) about this style of coding:

> ... both decisions still, to this day, make me smile. I relish getting to write `people.third` in a test case or the console. No, that’s not logical. It’s not efficient. It may even be pathological. But it continues to make me smile, thus fulfilling the principle and enriching my life, helping to justify my continued involvement with Rails after 12 years of service.

Okay... `¯\_(ツ)_/¯`

In my opinion this style of coding is not only bad, it is also *less readable and in a certain sense uglier*. "Why?" you may ask. Well in my mind, code is not prose. Code was never intended to be prose. And code that masquerades as prose requires more mental energy to translate it back into the code that it really is.

Disagree? You're welcome... Let me know in the comments. (Oops... no comments yet...)
