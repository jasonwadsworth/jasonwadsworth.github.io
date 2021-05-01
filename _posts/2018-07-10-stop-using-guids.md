---
layout: post
title: Stop Using GUIDs for Identifiers
tags:
  - guid
  - uuid
  - identifiers
---

A number of years ago I started a job at a company that was just starting their move into the cloud. Shortly before I arrived there was a rather heated debate, from what I've heard, about whether identifiers should be integers (auto numbered specifically) or GUIDs. By the time I arrived the decision had been made to use integers. I quickly stepped in and pointed out all the problems this has in distributed systems and we, mostly, switched to GUIDs. That wasn't the right choice.

It occurred to me some years later that the conversation was framed completely wrong. The debate shouldn't have been integers vs. GUIDs. Instead we should have asked "what is the best option for identifiers?" In so doing we would have talked through the same pros and cons of the integer vs. GUID debate but in a completely different manor. In stead of a list of why one is better than the other we should have been talking about what things are important. In so doing we might have realized that what we needed from a GUID was the randomness of the identifier. We needed it to be truly unique, with little to no risk of a clash. GUIDs happen to provide that, but that's a side effect of using them.

Think about this: when you chose a GUID as your identifier do you care about the fact that it's a 16 byte number that it typically represented in some form of hexadecimal? Probably not. What you do care about is that it is "globally unique."

The solution, it turns out, is far simpler than imagined. Leave it to software engineers to make things too complex. So what is the best solution? Believe it or not, it's a string. Sounds crazy, doesn't it? But is it really all that crazy? Let's say I want to have a truly unique value in my string. Instead of something like "Guid.NewGuid()" I just change it to "Guid.NewGuid().ToString()". A pretty simple change that gets me what I want.

I know what you're thinking. If that's the solution then why not just use a GUID? There are a number of reasons, but let's start with the simplest. By using a string you can put whatever you want in there. In some cases you might want some special values that mean something in the system. With a GUID you likely have some set of randomly generated values that mean something. With a string you can have something meaningful in the text. Sometimes the value might need to be a little easier for a human to use, maybe a little shorter than 32 characters. No matter the reason, a string allows you to change the way you generate the uniqueness to suite whatever your needs are.

Make the change to strings, you'll thank me later. When you do, keep [Hyrum's Law](http://www.hyrumslaw.com/) in mind. Don't let your users drive the meaning of your values.
