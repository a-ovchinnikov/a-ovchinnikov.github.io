---
layout: post
title:  "Toolbox: a Quick Rising Assembler"
date:   2018-11-10 01:00:00 +0400
categories: auxiliary_tools
---
[*Brought to you by 'Long time ago in a galaxy far away'.*
](#here  "This is a rather old text which I still find amusing and thus I decided to publish it with some minimal brush-up.")

The other day I discovered that to be able to experiment with instruction
set trimming for my rustic 8-bit CPU I need a tunable assembler. I thought a
little about my options which appeared to be the following: to search for
existing tool, to modify an existing tool or to build my own tool from scratch.
Since I was not in a big hurry I picked the third option and started building
rudimentary assembler.  Here I'll document this endeavour.

The idea of assembler is quite simple on its face: take a stream of mnemonics
corresponding to processor's instructions more or less directly, return
a chunk of binary data which consists of processor's instructions and is ready
to run. In other words take something like

```assembly
        lda $4020
        inc
	add 10
```

and return something like

>34 40 20 2a 43 10

So to build an assembler one would at least need some mapping from mnemonics to
instructions. Also it would be nice to know if an instruction takes an
argument. And from the snippet above it could be deduced that some inctructions
require more data than others: it is highly likely that instructions that try
to address memory will require arguments wide enough to cover memory bus width
while instructions writing to or reading from registers will be limited
to register width. Sounds trivial so far. In the simpliest case[^1] it is easy
to imagine a dictionary with mnemonics as keys and dictionaries or tuples
holding all necessary data:

```python
instructions = {"lda": {"value": "42",  # hex value represented as a string for
                                        # conveniece.
			"size": 2,      # instruction size in bytes.
                       }
                ...
               }
```

Note that this example implicitly uses Python -- it is much easier to do text
processing in high-level languages. The general principle stays the same no
matter which tool one uses, but some tools are easier to use than others.

The next thing that comes to mind is addresses. Not the ones supplied directly,
but rather the ones which are defined by labels:

```assembly
        jmp start
start:  inc
```

_start_ is a label and it must be handled properly i.e. its position respective
to some point in memory like the beginning of the program must be determined
and the label must be substituted with it whenever it appears as an argument.
The code above immediately shows a problem: the label is used before it appears
in the program. Thus to be able to produce runnable binary an assembler must
first produce intermediate result to compute addresses of labels and then use
both these intermediate and labels-to-addresses mapping to build the binary.
Depending on how intermediates look it could be either two-pass assembler when
assembler first collects all labels and computes all addresses and then walks
along the program once again and uses precomputed addresses or one pass when a
separate table is kept for storing all locations at which label's value must be
inserted. When there are no more instructions to be processed this second table
is used to fill gaps in the code. Both approaches are pretty straightforward.

To make it a charm one needs one more component to a minimal assembler. The
question I have been omitting so far is how eaxctly input program should
be parsed. Now it is time to deal with it. There is as usual more than one way
to do it. First of all I must mention the venerable ad-hoc approach which was
used now and then in the olden days. Just mash together some string splitting
with substring search, add regular expressions to taste and here it is: fast to
hack in, brittle, hard to maintain and expand. I definitely won't recommend it
unless you want to experiment with some artificial obstacles like mandatory
field sizes a-la FORTRAN-77.

The other option is to do it right: write a parser. It will take
care of input and return a nice list of objects corresponding to each line's
content. A nice and thorough example could be found
[here](http://lisperator.net/pltut/).  The only problem with such apprach is
that it feels a bit too fundamental, especially given the fact that this
problem has been solved multiple times. So if you are not looking for an excuse
for diving into a comprehensive guide to parsing (like
[this](https://dickgrune.com/Books/PTAPG_1st_Edition/ "Parsing Techniques - A Practical Guide by Dick Grune"))
and reemerging half a year later with a lovingly-crafted industrial-grade
parse-em-all parser you would be better off if you search for existing solutions
and spend a few hours assessing your finds. Actually I did exactly this: I have
played a little bit with my homemade parser and then decided that I would
rather settle for some intermediate, not necessarily industrial-strength
solution. And after some consideration I decided to continue my experiments
with
 [pyparsing](http://infohost.nmt.edu/~shipman/soft/pyparsing/web/index.html).

The strength of the library (and, alas, its weakness, but that is a subject for
a different discussion) lies in how seamlessly one could embed parser
definition in her code. No code discontinuity, no messy EBNFs with 'E's
understood slightly differently than in the other nice library. Actually the
initial results impressed me so much that I will not resist the urge and
publish the entire parser definition for  simple assembler right here, right
now. Lo and behold:

```python
from pyparsing import alphas, alphanums, delimitedList, Combine, Group, LineEnd
from pyparsing import Literal, Optional, OneOrMore, SkipTo, Word, ZeroOrMore

COMMENT_SYMBOL = Literal(';').suppress()
LABEL_END = Literal(":").suppress()
comment = Combine(COMMENT_SYMBOL + SkipTo(LineEnd()))("comment")
identifier = Word(alphas + "_", alphanums + "_")
mnemonic = Word(alphas, alphanums)("mnemonic")
args = delimitedList(Word(alphanums+'_'), ',')("args")
label = Combine(identifier + LABEL_END)("label")
codeline = (Optional(label)
            + mnemonic
            + Optional(~LineEnd() + args)
            + Optional(comment))("codeline")
line = codeline | comment
```

Nice thing about parsing with pyparsing is that even if you don't have previous
exposure to it you will nevertheless be able to quickly get an idea of what is
going on. Yet what I believe is even more important is the fact that these
definitions stay reader-friendly. When I turned back to my own parser after a
couple of months of other activities I had to spend quite some time
refamiliarizing myself with my implementation. With pyparsing it took me about
a minute to read and understand what this piece is about. The only thing left
now is to apply `line` parser to some assembly code and deal with result.
`line` is a parser which is capable of parsing program lines consiting of parts
described above.  To get parse results now is trivial:

```python
parse_results = [line.parseString(x) for x in program]
```

`parse_results` for parser decribed above can have attributes _label_,
_mnemonic_, _args_ and _comment_. Actually an attempt to get any named field
will result in `parse_results` returning a string which will be empty in
case nothing was successfully matched.

Having a list of parsed elements it is trivial to take a list of instructions,
compute proper addreses for all labels and then substitute mnemonics for
operation codes, addresses for labels and keep constants intact:

```python
def assemble(program):
    return second_pass(*first_pass(parse(program)))

def parse(t):
    return [line.parseString(x) for x in t.split('\n')]

def first_pass(code):
    lt = LabelTable()
    for line in code:
        if line.codeline and line.codeline.label:
            lt.add_entry(line.codeline.label)
        if line.codeline and line.codeline.mnemonic:
            lt.advance_address(line.codeline.mnemonic)
    return code, lt.table

def second_pass(code, address_table):
    out = []
    for line in code:
        cmd = mnemotable.get(line.mnemonic)
        if cmd is not None:
            out.append(cmd['value'])
            for arg in line.args:
                arg = address_table.get(arg, arg)
                out.append(_shape(str(arg).zfill((cmd['size']-1)*2)))
    return " ".join(out)

def _shape(x):
    return x[:2] + (' ' if len(x) == 4 else '') + x[2:]
```

You can see that the code is crude and straightfoward, the most complex part of
it is code for formatting results.  `LabelTable` is an auxiliary class which
stores label locations and advances memory location. It is trivial to come up
with some implementation for it, the one I used I placed in Appendix in order
to save space.

Calling `assemble()` with a program now will result in a string of
space-delimited pairs of characters representing hexadecimal values, so the job
is done, right?  Not yet. Assembler as described is as useless as it is simple.
The other most important part that was omitted from the previous discussion is
the question how one can abstract some common patterns in the code? Consider,
for example assemblers for MIPS that have pseudoinstructions which look like
regular ones to the user, but consist of several CPU instructions under the
hood. Arguably the most important part of a DIY assembler built for reasearch
purposes is an ability to handle such pseudinstructions and preferably some
other abstractions.  For instance it would be nice to have say an ability to
use libraries which get included at assembly time. Such libraries could provide
fixed-point arithmetics routines an could depend on libraries actually
implementing basic arithmetics (in case it is needed). Thus I found myself in
desperate need of a macroprocessor. Once again I could have used existing tools
like _m4_, but decided to roll my own instead.

At first macroprocessors might seem like scary magic. That's not true,
macroprocessors in essence are rather unsophisticated devices, especially ones
that I was interested in.  In fact all I needed was to be able to include
some pieces of code and to substitute some higher-level entries with actual
code. Both this activities are quite similar. Let me start
with includes first as they seem the lowest hanging fruit. 

Macroprocessor should run before assembler, thus it is its job to expand
entries like `INCLUDE foo.asm` when it sees one. This could be achieved by
following the next algorithm: push all the lines in a program to a stack, pop a
line, if it does not start with 'include' then push it to output queue, if it
does then push lines from file to include on top of program stack, repeat while
there is nothing left on program stack. Of course errors must be handled
properly and it also helps to keep a list of already included files at hand
since even several such files are capable of breeding include loops. It could
be tempting to use this mechanism for directly substituting code, but it is
better not to do so since it would require overcomplicated and error-prone loop
detection mechanism.  It is much beter to rely on macro definitions for this
job.

Macro definition could look like this:

```assembler
MACRO foo bar
      lda bar
ENDMACRO
```

First of all, it starts with word `MACRO` which is reserved specifically for
this purpose. One would have to strike it out of her lexicone when working with
this macroprocessor. Then goes macro name followed by named arguments to
macro. Then, starting on the next line goes macro body -- the code which would
get substituted later when macro names appears in code. Finally `ENDMACRO`
finishes macro definition.

When processing input macroprocessor maintains a table of known macro names.
When `MACRO` appears in input stream macro procesor processes everything till
the end of line to create an entry for this table consisting of macro name and
macro arguments, then reads everything till `ENDMACRO` and stores obtained code in
the table. When reading in other tokens macroprocessor checks if token matches
to any known macro name and if yes expands this macro and pushes expansion
result on top of input. This allows one to use macros inside macros which is
quite handy when building on top of a minimalistic architecture.

Macro substitution is slightly more involved than includes: macro body could
contain labels, so each time a macro gets expanded all known labels must be
mangled to avoid surprise jumps during run time. To achieve this labels must be
known to the macroprocessor which, in turn, means, that macro code must be
partially parsed to obtain all label names in use. When label names are known
mangling could be done as trivially as substituting label name substrings with
label names extended with line number on which a macro appeared. Arguments
could be expanded this way as well.

There is more to macro processing than these simple tricks including, but not
limited to expression evaluation and  conditional expansion which can help with
combating recursive expansions (I don't deal with it in my simple demonstration
in any other way than by issuing this stern warning to be careful when working
with your experimental setups). I am stopping here since any other augmentations
above the most necessary feature set would mean a transition to a more complex
system than I intend to work with.

To build a macroprocessor one first needs to be able to parse two special cases
of `INCLUDE` and `MACRO`. This could be easily achieved by expanding parser
definition like this[^2]:

```python
endmacro = Literal('ENDMACRO')('macroend')
filename = Word(alphanums + '_.')("filename")
include = (Literal('INCLUDE').suppress() + filename)('include')
macrostart = (Literal('MACRO').suppress() + identifier('macroname')
            + Optional(args))('macrostart')
mline = endmacro | include | macrostart | codeline | comment
```

Now the parser from above could be used in a preprocessor:

```python
def preprocess(program):
    def process_include(res, included, linelist):
        fname = res.include.filename
        if fname in included:
            return linelist
        if os.path.exists(fname) and os.path.isfile(fname):
            with open(fname, 'r') as f:
                to_include = f.readlines()
            to_include.extend(linelist)
            included.append(fname)
        else:
            raise Exception("%s: file not found" % fname)
        return to_include

    def process_macro(res, macrotable, linelist):
        margs = res.macrostart.args.asList() if res.macrostart.args else []
        mbody, mlabels = [], []
        while True:
            nextline = linelist.pop(0)
            tmp = mline.parseString(nextline)
            if tmp.macroend:
                break
            if tmp.label:
                mlabels.append(tmp.label)
            mbody.append(nextline)
        macrotable[res.macrostart.macroname] = {'body': "\n".join(mbody),
            "args": margs, "labels": mlabels}
        reurn macrotable, linelist

    def process_codeline(res, macrotable, linelist, out, ctr):
        if res.codeline.mnemonic not in macrotable:
            out.append(line)
            return linelist
        if res.codeline.label:
            out.append(res.codeline.label + ': ' +'nop')
        macro = macrotable[res.codeline.mnemonic]
        body = mangle_labels(macro['body'], macro['labels'], ctr)
        body = expand_args(body, macro['args'],
            list(res.codeline.args)).split('\n')
        body.extend(linelist)
        return body

    def mangle_labels(body, labels, modifier):
        newlabels = dict((x, x+str(modifier)) for x in labels)
        for label, newlabel in newlabels.items():
            body = body.replace(label, newlabel)
        return body

    def expand_args(body, abstract_args, actual_args):
        newargs = {}
        if abstract_args:
            newargs = dict(zip(abstract_args, actual_args))
        for abstract_arg, actual_arg in newargs.items():
            body = body.replace(abstract_arg, actual_arg)
        return body

    linelist = program.split('\n')
    included, out, macrotable = [], [], {}
    ctr = 0
    while linelist:
        line = linelist.pop(0).rstrip('\n')
        ctr += 1
        res = mline.parseString(line)
        if res.include:
            linelist = process_include(res, included, linelist)
        elif res.macrostart:
            macrotable, linelist = process_macro(res, macrotable, linelist)
        elif res.comment:
            out.append(line)
        elif res.codeline:
            linelist = process_codeline(res, macrotable, linelist, out, ctr)
        else:
            raise Exception("Impossible!")
    return "\n".join(out)
```

Assembling with the code above now looks like `assemble(preprocess(program))`.

To sum it all up, the resulting tool is straightforward, simplistic, does not
handle many corner cases and turns a blind eye to most of possible errors. It
is just enough to go on exploring instruction sets.  The only means
of abstraction available is rudimentary preprocessing, but my observations
indicate that this is enough for experiments. More complex tools either won't
be used in fullness or indicate a rather complex and feature rich system which
is out of scope now. In case basic features described in this post are not
enough to reach ones goals a much deeper intro
into assemblers should be consulted, for example [this one](http://www.davidsalomon.name/assem.advertis/AssemAd.html
"Assemblers and Loaders by David Salomon").

Appendix
--------

_LabelTable_ implementation:

```python
class LabelTable(object):

    def __init__(self, mnemotable=mnemotable):
        self.mt = mnemotable
        self.table = {}
        self.current_address = 0

    def add_entry(self, entry):
        if entry in self.table:
            raise Exception("Duplicate label definition")
        address = _shape(hex(self.current_address)[2:].zfill(4))
        self.table[entry] = address

    def advance_address(self, l):
        self.current_address += self.mt[l]['size']
```

-----------------------------------

[^1]: There is a special case of mnemonics that look like they have arguments, but in fact they don't (consider register exchange commands). I am intentionally omitting them now. Also I am omitting the case of complex memory addressing schemes, since I believe that if a CPU supports such features then it is not a simple one anymore and by induction its instruction set is not simple as well. So quite naturally it gets excluded from the discussion.
[^2]: `MACRO` and `ENDMACRO` could be treated as open and close brackets for a piece of code, but in my opinion that would overcomplicate the basic tool.
