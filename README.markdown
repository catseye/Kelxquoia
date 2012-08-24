Kelxquoia
=========

Kelxquoia is an esoteric programming language designed by Chris Pressey on
December 23, 2010.  It is a self-modifying, in fact self-*destroying*,
language which combines grid-rewriting with remotely fungeoid playfield
traversal.

Program State
-------------

There is an instruction pointer (IP) with a location and a direction in an
unbounded, two-dimensional Cartesian grid of symbols.  As the IP passes over
symbols, it executes them and erases them from the playfield (overwrites
them with blanks). The program ends when the IP travels off into space, never
to return.

There is a stack. The stack may contain objects of two types, grids and rows.
Instructions expect certain types of objects to be present on the stack; if
the types are incorrect, or insufficient elements are on the stack, the
instruction has no effect.  (That means that the state of the stack, too, is
unchanged by the instruction.)

Instructions
------------

If the IP passes over any symbol _x_, and there is a `'` symbol to the right
of the IP's line of travel, a row is popped from the stack, the symbol _x_ is
appended (to the rightmost position) of that row, and the new row is pushed
back onto the stack. The symbol _x_ is erased, but the `'` is not.

The instruction `-` pushes an empty row onto the stack.

The instruction `+` pushes an empty grid onto the stack.

The instruction `*` pops a row, pops a grid, appends the row to the bottom of
the grid, then pushes the new grid.  Note that the left-hand edges of every
row in a grid line up, but their right-hand edges need not do that.  When
considered as a grid, all unused squares in the grid are considered to contain
blanks.

The instruction `?` pops a row from the stack, appends a wildcard to it, and
pushes the new row back onto the stack.

The instruction `/` pops a grid called the _replacement_, then pops a grid
called the _pattern_, then replaces all non-overlapping occurrences of the
pattern in the playfield with the replacement.  All overlapping occurrences
are untouched, to avoid ambiguity. A few things to note:

*   The replacement must not be any larger than the pattern.  If it is, the
    two grids remain popped but the instruction otherwise has no effect.
*   If the replacement is smaller than the pattern, it is padded with blanks
    (to the bottom and to the right.)
*   There can be at most one wildcard in the pattern. If there are no
    wildcards in the pattern, there should be none in the replacement.  If
    either of these conditions does not hold, the two grids remain popped
    but the instruction otherwise has no effect.
*   Any symbol in the playfield may match the wildcard in the pattern.
    Where-ever there is a wildcard in the replacement, the symbol that matched
    the wildcard in the pattern is substituted instead.
*   If the executed `/` instruction itself is replaced by something during
    this process, whatever it was replaced with will not be erased.
*   If the pattern consists only of blanks, or only wildcards, the
    implementation may choose to halt the program rather than filling the
    entire playfield with replacement. 

The instructions `>`, `<`, `^`, and `v` cause the IP to begin travelling east,
west, north, and south, respectively.

The instruction `!` clears the stack.

Executing any other symbol as an instruction has no effect.

The symbol `$` indicates the initial position of the IP.  The initial
direction of the IP is always east. If there are multiple `$` symbols in the
source code, it is an error and the program is not executed.

Computational Class
-------------------

The author believes Kelxquoia to be Turing-complete because:

*   Repeated use of the `/` instruction can be used to implement
    non-overlapping grid-rewriting, which can simulate an arbitrary Turing
    machine; see below.
*   Assuming all instructions that have been erased after execution can be
    restored, the IP can travel in a loop, to indefinitely apply the `/`
    rewrites.
*   Instructions that have been erased after execution can be restored by
    further rewrites of a certain, fixed form.
*   To effect a conditional halt, a critical direction-changing instruction
    can be selectively not restored to the playfield. 

### Simulating a Turing Machine with non-Overlapping Grid-Rewriting ###

In this section we show how a Turing machine can be simulated with
grid-rewriting where we can match only non-overlapping instances of a pattern.
(The addition of wildcards does not add any computational power, but does add
some expressivity, as it can reduce the number of rewrites that need to be
made.)  The tape symbols of our machine will be the digits `0`-`9`, and the
states of the finite control will be labeled with the lowercase letters
`a`-`z`. In addition we will use the `.` symbol to depict whitespace, for
clarity.

We can depict each tape cell with the following 2x2 grid of symbols:

    0T
    T+

We arrange the tape cells horizontally in the playfield, and we make sure
there is only one such tape:

    0T0T1T2T3T0T0T
    T+T+T+T+T+T+T+

We have rewriting rules that extend the tape in either direction, as needed:

    ?T..  ->  ?T0T
    T+..      T+T+

    ..?T  ->  0T?T
    ..T+      T+T+

We situate the tape head directly below the cell of the tape we wish to
affect, and again we make sure there is only one such head.  The tape head
looks like the following 2x2 grid:

    aH
    HN

The upper-left corner contains the finite control label, and the lower-right
corner contains a "movement indicator".  When the head is ready to move, the
movement indicator is changed, and we have the following two rewrite rules
implement the move:

    ?H..  ->  ..?H
    HR..      ..HN

    ..?H  ->  ?H..
    ..HL      HN..

Now, for each transition rule in the Turing machine, we can compose a
grid-rewriting rule which implements it.  For example, "In state `f`, if
there is a `5` on the tape, write a `7` and move left" corresponds to the
rule:

    5T  ->  7T
    T+      T+
    fH      fH
    HN      HL

From this description it should be fairly evident that this both simulates a
Turing machine (not quite arbitrary, but certainly with sufficient symbols
and states to be universal) and that the non-overlapping condition poses no
obstacle (if the tape and head are arranged as described, these patterns will
never match overlapping instances of the playfield).

Examples
--------

The following example replaces the words `WOW` and `POP`, at the bottom of
the program, with `BOB` and `MOM`:

    $+-W*-P*+-B*-M*/
       '  '   '  '
    WOW
    POP

The state of the playfield at the end of the program would be:

    $
       '  '   '  '
    BOB
    MOM

The following example rewrites some stuff then restores the instructions that
did the rewriting:

     +-0 0*+-1*/+-?*-R*- *+-?*-R*-?*/
     RRRRRRRRRRRRRRRRRRR RRRRRRRRRRRR
    $+-0 0*+-1*/+-?*-R*- *+-?*-R*-?*/
       ' '   '       '  '      '   

     00 00 00 00

The following example extends the previous example to an infinite loop:

     >+-0 0*+-1*/+-?*-R*- *+-?*-R*-?*/v
     RRRRRRRRRRRRRRRRRRRR RRRRRRRRRRRRR
    $>+-0 0*+-1*/+-?*-R*- *+-?*-R*-?*/v
        ' '   '       '  '      '   
                 '         '  '     
     ^      /*?-*P-*?-+*?-*P-* -+     <
     P      PPPPPPPPPPPPPPPPPP PP     P
     ^      /*?-*P-*?-+*?-*P-* -+     <

     00 00 00 00

Acknowledgements
----------------

The contents of this document were taken from the Kelxquoia article on the
esolangs.org wiki, which is in the public domain.
