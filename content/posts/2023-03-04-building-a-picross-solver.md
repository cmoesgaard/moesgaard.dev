---
title: Building a Picross solver
date: 2023-03-04T16:43:29.671Z
---
[Picross puzzles](https://en.wikipedia.org/wiki/Nonogram) have long been a favorite passtime of mine.

I know that the puzzles don't necessarily guarantee a unique solution, or even a solution that can be reasonably found using logic alone.

## Formats
I need a format for specifying the puzzle, to be read by my solver. I would like to avoid defining my own format if possible, and I figured someone probably defined a suitable format already.

![](https://imgs.xkcd.com/comics/standards.png)

[Yup](https://webpbn.com/export.cgi).

```
catalogue "webpbn.com #1"
title "Dancer"
by "Jan Wolter"
copyright "© 2004 Jan Wolter"
license CC-BY-3.0
width 5
height 10

rows
2
2,1
1,1
3
1,1
1,1
2
1,1
1,2
2

columns
2,1
2,1,3
7
1,3
2,1

goal "01100011010010101110101001010000110010100101111000"
```