---
layout: post
title:  "Learning Zig"
date:   2025-11-09 17:45:00 +0100
categories: valheim modding 
---


I've decided to learn zig. I've grown a bit tired of the wasteful nature of development and want to strip things back. At least for personal projects. But before I endeavor to make something "real" or useful I figure I'll start with something basic. First and foremost I need to better grasp the syntax and semantics of the language. Secondly I need to get a feel for the zig way of doing things. Perhaps there is no such thing. Perhaps there is nothing like an established best practice. That would be fine too and for personal projects I'm sure I'll deviate from such norms anyway, at least the ones I come to disagree with. Personally I am mostly interested in making efficient software that solves some problem for its users. Of both those things the second is vastly more important. Hopefully zig I can scratch both itches. I like the idea of not wasting cpu cycles on what essentially becomes shuffling data between memory addresses, however I do realize that this is commonly not the main inefficiency. In my experience the main efficiency and also performance culprit is bad choice of algorithms. That is often caused by no one caring enough to fix the issue and also libraries and frameworks often encouraging poor practices. That is often fine, but when I build stuff for me I want it to avoid being wasteful.


That's the main reason for picking zig. I could also have picked up e.g. rust (and I have done some small hacks with it), but I didn't enjoy the development experience. That was somewhat of a surprise to me as I do appreciate the safety guarantees it provide. Perhaps I need to take another stab at it at some point.

In order to learn a new programing language I find that it's good to pick something familiar and then recreate it with the tools the language provide. As I am mostly a backend web developer I figure I'd tackle the tried and true be all end all of problems. A to-do app. It is such a simple thing to build, but in order to do it I'll get familiar with how to setup a basic web app powered by zig. In addition I can easily extend the problem to e.g. talking to a database, handling authentication, collaborative editing, generating a burn down chart. Plenty of directions to go in if I want to OR I finish it up and pick a more real problem!

Anyway, all that said I set up a basic project with `zig init`. It generates a basic project with some code illustrating how to write tests, basic syntax and a simple print line. It  does point you towards using a general purpose memory allocator. My understanding so far is that zig is very explicit in it's memory management. There is no hidden allocations. This is quite a different approach to more mainstream memory managed languages. I believe this will take some getting used to, but so far I enjoy the control it offers. I could for instance make a large memory request at startup of the application and then "just" ensure to do all processing within those bounds. Alternatively I could create a new memory allocation for each http request or maybe one for each session. We'll see if I can figure out what a good strategy is.

I decided to ask my helpful LLM to teach me zig. I figured it is at least more competent at it than me. I asked it to guide me along, but that I wanted to write the code myself (typing in the code is the best way to learn IMO, just reading code and copy pasting isn't good for retention I find). This might have been a mistake. Turns out the LLM is trained on older versions of zig and likely there isn't that much zig out there to train on to begin with (in comparison with the billions of lines of JavaScript available). After a bit of nudging it managed to guide me to a very basic hello world where it essentially is just a tcp server that responds to a well-formed get request. Next up I'll dig into the official docs a bit and see if there is a built in http server. I don't think I want to implement a whole webserver from scratch starting out for my to-do app.

## Learnings so far
- **Allocators**: Zig requires explicit memory management through allocators (no GC)
- **RAII Pattern**: Used `defer` for cleanup, similar to Go's `defer` but more explicit
- **Error Handling**: `!void` return types, `try` keyword for error propagation
- **API Instability**: Zig 0.15.1 has different APIs than online tutorials
- **Type Mismatches**: Compiler errors were very helpful in guiding fixes

