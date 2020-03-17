---
title: How "Mazes for programmers" book made programming fun again 🎉
published: true
description: Stop reading your latest Kubernetes book and dive into a world of random maze generation!
tags: ruby, algorithms, motivation, books
---

Typical programming books I read include words like "scalability", "containers", "serverless" etc. However, recently I've completed one that had none of them and still it was a highly valuable read for me.

![My own twisty little passages](https://user-images.githubusercontent.com/5732023/76292085-c8becb00-62ae-11ea-8e05-fec241c9421f.jpg)*My own twisty little passages*

The book I'm talking about is ["Mazes for programmers"]([https://pragprog.com/book/jbmaze/mazes-for-programmers](https://pragprog.com/book/jbmaze/mazes-for-programmers))  by Jamis Buck and, as the title suggests, it's about random maze generation. I've ran into this title totally randomly - while browsing followers of my [ruby gem](https://github.com/Bajena/ams_lazy_relationships/) on GitHub. I've noticed that one of the followers had a repository called [Mazes for Programmers](https://github.com/jeremybytes/mazes-for-programmers/) that included the maze generation code and a link to the book. It drew my attention, because I recalled the times when playing games with randomly generated maps (especially Diablo II ❤️) attracted me to start programming. Being hit by nostalgia and curiosity, I decided to buy the book (paperback naturally) and learn how to create such beauties.

![Diablo](https://user-images.githubusercontent.com/5732023/76288603-67472e00-62a7-11ea-9705-9d1453ba3ce1.png)
*Good ol' Diablo II*

BTW: If you're interested in how Diablo II's mazes were built have a look at this [article](https://diablo.gamepedia.com/Area_Size_(Diablo_II)).
Actually whole Diablo wiki is great, have a look at the whole wiki ;D
## Contents of the book
Jamis Buck grouped the chapters into 4 parts with progressively growing concept difficulty level:

### Part one
In the first part he introduced basic maze generation (binary tree, recursive backtracker, hunt-and-kill algorithms) and solving (Dijkstra's algorithm) techniques and provided an implementation of a basic grid system. It's worth mentioning that the code snippets in the book are written in Ruby language which was perfect for a Rubyist like me :)

![Zdjęcie__11_03_2020_o_07_35__2](https://user-images.githubusercontent.com/5732023/76389568-cd958480-636b-11ea-8463-095c5d313090.png)

### Part two
The second part shows how to "free" yourself from a rectangular grid cage. It explains how to build mazes in radial, hexagonal and triangular shapes. It required me to refresh some of high school maths, but the effects are really cool!

![Zdjęcie__11_03_2020_o_07_48](https://user-images.githubusercontent.com/5732023/76389993-da66a800-636c-11ea-9ec3-2a2541ee86e0.png)

### Part three
In this chapter Jamis teaches six more maze-generation algorithms. If you've studied Computer Science you may be familiar with some of the names in this chapter as it explains how to use Kruskal's, Prim's and recursive division algorithms.

### Part four
I feel like this book would already be complete without chapter four, but the author decided to go a level higher and introduced mind-bending concepts like 3D and 4D mazes, labyrinths on Möbius strip surface and rendering a maze on a sphere. Crazy 😅

![image](https://user-images.githubusercontent.com/5732023/76391009-cb80f500-636e-11ea-8c94-6abb7bc870e2.png)
*A Möbius strip maze!*

## The takeaways
Even though "Mazes for Programmers" didn't help me a lot with my day-to-day work I can surely say that it had a great value to me as a programmer.

Above all it simply brought back **fun** into programming, because I learned and create things not because I had to, but because I could. And I suppose that the reason why most of us started programming was curiosity and the great feeling of making computers do whatever stuff we imagine.
It's definitely a read for people who already are burnt out or starting to feel like burning out in their job.

Another nice takeaway from the book is that I could refresh some of the computer science basics (graph algorithms, geometry) that I haven't touched for a few years already. Some of the concepts were completely new to me, but Buck has an ability to explain even more difficult topics in a very clear way, skipping most of the information nonessential for understanding. Maybe if my university teachers had such ability too I wouldn't be doing web development now (NASA, here I come 🚀).

Now the last but not least - I think it's worth exploring even such exotic topics like random maze generation, because doing it may open a whole new world of ideas and problems to solve. And by solving problems you're becoming a better developer, no matter if you're a web/game/space shuttle developer.

In my case the book motivated me to create a simple game in which players (red & blue dots) have to get through a labyrinth to reach a treasure (the golden dot). I decided to write it in Ruby, so I picked the [ruby2d](https://www.ruby2d.com/) library. I don't think I'd ever decide to learn how to program graphics in Ruby if not for the book. Thanks Jamis!

![screencast 2020-03-11 08-44-52](https://user-images.githubusercontent.com/5732023/76393921-d8084c00-6374-11ea-84ee-0dcfd89d413c.gif)
*It's a-maze game! You can find the full code on [GitHub](https://github.com/Bajena/a_maze_game).*

## OK, I know everything about mazes already, what's next?
It turns out that the author wrote another book - this time about ray tracers. Normally I'd be scared by the amount and complexity of maths, but I blindly bought it, because I'm sure the concepts are going to be explained clearly. When I finish the challenge I'll surely share the results with you :)

![image](https://user-images.githubusercontent.com/5732023/76395229-20286e00-6377-11ea-8d9d-9459174a49cb.png)
*The Ray Tracer Challenge*
