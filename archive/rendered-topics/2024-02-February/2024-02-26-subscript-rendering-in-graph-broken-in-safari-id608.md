# Subscript rendering in graph broken in Safari

sjors | 2024-02-26 17:48:55 UTC | #1

While reading @sipa's post on cluster mempool I noticed a glitch:

[quote="sipa, post:1, topic:202"]
Consider the transaction graph below
[/quote]

Strangely in Safari on macOS 13.6.4 and 14.3.1 the subscripts numbers  are missing from the graph. They seem to get bunched in the top-left corner:

![subscripts|690x420](upload://cSNreuchXWJzi095wLRWD8khoYz.png)

They do render correctly in Chrome.

Bringing this up in a meta topic to not distract the conversation there.

-------------------------

ajtowns | 2024-02-27 05:56:12 UTC | #2

Do you get the same behaviour on a gist or in the [live editor](https://mermaid.live/)? Here's the source for easy cutting and pasting:

```
graph BT
   t5["t<sub>5</sub>"] --> t2["t<sub>2</sub>"];
   t6["t<sub>6</sub>"] --> t2;
   t6 --> t3["t<sub>3</sub>"] --> t1["t<sub>1</sub>"];
   t7["t<sub>7</sub>"] --> t4["t<sub>4</sub>"];
```

If it's buggy there, probably file [an issue](https://github.com/mermaid-js/mermaid/issues) upstream? If it's fixed upstream, could probably manually update the component until discourse gets around to an official update.

-------------------------

sjors | 2024-02-27 12:02:48 UTC | #3

I get the same in the live editor.
![t|507x500](upload://t0XGChc9cDmCHK4t7I8vvFMZb7J.png)

I'm not familiar enough with Graphviz (?) and how the Delving forum (and Mermaid?) works to file a bug report.

-------------------------

