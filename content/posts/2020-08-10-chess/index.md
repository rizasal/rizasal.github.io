---
layout: article
title:  "Representing a Chess Board"
date:   2020-08-10
image: "images/chess_1.png"
description: "A more efficient way to reperensent a Chess Board using FEN strings" 
tags: ["chess"]
categories: random
---
Making a game of chess is a daunting task. Especially if it is to be online and multiplayer.
It is not feasible to send the entire board object around and we need a way to send individual piece movements.

One of the ways to do that is the Forsynth-Edwards Notation aka FEN string. A beautiful way to represent the board in a one line ASCII string. It is widely used to record chess games and start from different initial positions
or even the middle of a game.


For those who are more interested in programming chess rather than playing it, The columns are represented by alphabets (a-h) and is called a 'File'. The rows are represented by digits (1-8) and is called a 'Rank'.

{{< figure src="/chess/chess_1.png" title="Starting Position of the Chess Board" >}}


Below is the entire FEN for the starting position of the chess board.


> rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1


Cool right. No large objects to pass around the internet. Yay.
Let me start by explaining what each of these alphabets and digits stand for.


The lowercase letters represent the black pieces and the uppercase ones represent the white pieces.

```
R - rook, Q - Queen, K - King, B - Bishop, N - Knight, P - pawn
```

The grammar for the FEN notation is as follows. There's 6 parts to the FEN string, each separated by spaces.

```
<Piece Placement> <Side to move> <Castling ability> 
<En passant target square> <Halfmove clock> <Fullmove counter>
```
## Piece Placement

 Let's look at the first part of the initial position string again.

```text
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR - Piece Placement Part
```

The first portion of this string represents the 8th row of the chess board, aka rank8 , the next portion being the row with all pawns which is rank 7 and so on with each portion separated by a '/'. The digit 8 represents a completely empty row.

```text
<Piece Placement> ::= <rank8>'/'<rank7>'/'...'/'<rank1>
```
### Representing A Rank aka Row
```text
<rank> ::= [Any digit btw 1-7] <Any piece> [Any digit btw 1-7]
or
<rank> ::= '8'
```
Any of the three fields in the first definition of rank can be left empty. The digits stand for empty spaces. So a 2 would mean two consecutive squares are left blank. If the rank starts with a piece it means that the piece is placed at the first column aka File. A digit, say n, followed by a piece, say p, denotes n blank columns and then a black pawn('p' stands for pawn).


The digit piece combo can be repeated as many times.
/4pp2/ means 4 empty columns, two black pawns and then 2 empty spaces
If the rank is 8, then it stands for a completely empty row with no pieces.
```text
8/8/8/4pp2/8/8/8/8
```
Would result in

{{< figure src="/chess/chess_2.png" title="Top 3 rows are represented by 8/8/8. The next row(rank 5) is represented by 4pp2" >}}

<!-- {% include image.html url="/assets/images/chess_2.png" description="" %} -->

## Side To Move

The next part in the FEN string is the side  to move part could be one of either 'w' (white) or 'b' (black) indicating whose turn is next.
```text
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w
```
## Castling ability

*The Castling Rights specify whether both sides are principally able to castle king- or queen side, now or later during the game - whether the involved pieces have already moved or in case of the rooks, were captured*

Castling ability represents the [castling rights](https://medium.com/r/?url=https%3A%2F%2Fwww.chessprogramming.org%2FCastling_Rights), which does not actually mean whether a castle is possible right now or not but indicates whether the player lost the castling rights during the game so far. 
```text
K - King side White Castling, Q - Queen side white castling,
k - King side black Castling , q - queen side black castling.
```
If the castling right is lost, it is represented by a '-'.
```text
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq
```
## En-Passant

En passant is a special pawn capture move that make programming chess especially difficult, making us require a special field for it in our FEN string.

{{<figure src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/3b/En_passant.gif/512px-En_passant.gif" link="https://commons.wikimedia.org/wiki/File:En_passant.gif" attr="Calusarul / CC BY-SA" attrlink="https://commons.wikimedia.org/wiki/File:En_passant.gif" width="512" title="En-passant Move">}}

<!-- <a title="" href="">
<img style="display:block; margin: auto; max-width: 70%;" width="512" alt="En passant" src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/3b/En_passant.gif/512px-En_passant.gif"></a>
<br> -->
```text
<En passant target square> ::= '-' | <epsquare>
<epsquare>   ::= <fileLetter> <eprank>
<fileLetter> ::= 'a' | 'b' | 'c' | 'd' | 'e' | 'f' | 'g' | 'h'
<eprank>     ::= '3' | '6' (The two ranks where en-passant is possible)
```

The en-passant part of the FEN string  is set to '-' if the previous move is not a double push of a pawn (Movement of two squares from the starting position). If it is, regardless of whether en-passant is possible or not, it is set to the rank just below (above for black) the pawn that was double pushed, indicating that an en passant move to that square is possible.

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq -
```
## Half Move Clock
Unless there is a pawn move or a capture, this counter is incremented. When it hits fifty, the game is called a draw ([Fifty Move Rule](https://en.wikipedia.org/wiki/Fifty-move_rule)).
```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0
```
## Full Move Counter
It starts at 1, and is incremented each time Black moves.
```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
```

Thus we have it, A complete chess board representation in FEN. 