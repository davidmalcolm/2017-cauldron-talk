.. Note on building:

   sphinx 1.6+ is incompatible with hieroglyph:
     https://github.com/nyergler/hieroglyph/issues/124
     https://github.com/nyergler/hieroglyph/issues/127

   As a workaround, I've been building this using a virtualenv
   containing sphinx 1.5.6:

     (in /home/david/nomad-coding):
       virtualenv venv-sphinx-1.5
       source venv-sphinx-1.5/bin/activate
       easy_install sphinx==1.5.6
       easy_install hieroglyph

   Activating the virtualenv:

   $ source /home/david/nomad-coding/venv-sphinx-1.5/bin/activate

   "make slides" then works

=======================================================
Diagnostic and location-tracking improvements for GCC 8
=======================================================

GNU Tools Cauldron 2017

David Malcolm <dmalcolm@redhat.com>

https://dmalcolm.fedorapeople.org/presentations/cauldron-2017/

.. Abstract: I've got a number of proposals for improving diagnostics and
   how we track source locations in GCC, which I'd like to present at
   Cauldron; extending location-tracking to cover:

   (a) all expressions (including constants, and uses of a decl), not
       just compound expressions

   (b) other syntactic elements (e.g. for implementing IDE integration)

   I also want to discuss how we might help advanced users track how GCC
   is optimizing their code via some kind of hybrid of the dump_file and
   diagnostics subsystems.

   I plan for most of the session to be interactive, hence this feels
   like something of a "diagnostics and location-tracking BoF".

.. TODO: when and where?

.. TODO: objectives for the talk?


What this talk is about
=======================

* Problems with location-tracking in our internal representation(s)

  * Possible solutions

    * (with a detour into IDE support)

* Better information for advanced users on what optimizers are doing

  * "optimization remarks"

  * tracking cloning of statements (e.g. inlining, vectorization, etc)

* Discussion of above, and what's doable for gcc 8 (and beyond)


History of source-location tracking in GCC
==========================================

Source locations are encoded as 32-bit values
(:c:type:`location_t` a.k.a :c:type:`source_location`).

Effectively a key into a database (:c:data:`line_table`).

  * early versions of GCC merely tracked filename and line number

  * column numbers were added in 2004

  * tracking of macro expansions in 2011 (Dodji, I believe)

  * I added tracking of *ranges* of source code (rather that just points)
    in GCC 6


``location_t`` vs  ``rich_location``
====================================

* A :c:type:`location_t` is a caret+start+finish, often with caret==start
  e.g.::

    dst = foo * bar;
          ~~~~^~~~~

    dst = foo * bar;
          ^~~

* A :c:type:`location_t` can also represent a macro expansion
  location.

.. nextslide::
   :increment:

* A :c:type:`rich_location` is a bundle of information for
  :c:func:`diagnostic_show_locus`:

  * a primary :c:type:`location_t`

  * zero or more secondary :c:type:`location_t`

  * zero or more fix-it hints

This talk is about :c:type:`location_t`.


Location information in the C and C++ frontends
===============================================

.. code-block:: c

   int test (int first, int second)
   {
     return foo (100, first * 42, second);
   }

.. nextslide::
   :increment:

We capture a location (of sorts) for the FUNCTION_DECL::

    int test (int first, int second)
        ^~~~

.. nextslide::
   :increment:

but we throw away these locations:

* return type::

    int test (int first, int second)
    ^~~

* param locations (FIXME: do we?)::

    int test (int first, int second)
              ^~~~~~~~~  ^~~~~~~~~~

.. nextslide::
   :increment:

We capture a location for the CALL_EXPR::

     return foo (100, first * 42, second);
            ~~~~^~~~~~~~~~~~~~~~~~~~~~~~~

.. nextslide::
   :increment:

We capture locations for compound expressions e.g. the MULT_EXPR::

    return foo (100, first * 42, second)
                     ~~~~~~^~~~

.. nextslide::
   :increment:

...but we *don't* permanently capture locations of constants and
*uses* of decls::

    return foo (100, first * 42, second)
                ^--              ^-----

(see `PR 43486 "Preserve variable-use locations" <https://gcc.gnu.org/bugzilla/show_bug.cgi?id=43486>`_,
filed 2010-03-22)

.. nextslide::
   :increment:

Other locations we discard during parsing:

* locations of attributes of a function

* locations of individual tokens like close parens and
  semicolons::

   int test (int first, int second)
            ^         ^           ^
   {
   ^
     return foo (100, first * 42, second);
                ^   ^           ^       ^^
   }
   ^

.. nextslide::
   :increment:

Missing location information limits our ability to implement
"cousins" of a compiler on top of the GCC codebase e.g.:

  * code refactoring tools,
  * code reformatting tools
  * IDE support daemons
  * etc

.. nextslide::
   :increment:

Ultimately, it makes our diagnostics harder to read than they could be.


Why do we lose the location information?
========================================

Leaf nodes in many expressions don't have location information.

Quoting tree.h:

.. code-block:: c++

   /* The source location of this expression.  Non-tree_exp nodes such as
      decls and constants can be shared among multiple locations, so
      return nothing.  */
   #define EXPR_LOCATION(NODE) \
     (CAN_HAVE_LOCATION_P ((NODE)) ? (NODE)->exp.locus : UNKNOWN_LOCATION)

.. nextslide::
   :increment:

Nasty workarounds:

.. code-block:: c++

  #define EXPR_LOC_OR_LOC(NODE, LOCUS) (EXPR_HAS_LOCATION (NODE) \
                         ? (NODE)->exp.locus : (LOCUS))

.. code-block:: c++

  location_t loc = EXPR_LOC_OR_LOC (src, input_location);

.. nextslide::
   :increment:


Workaround in C frontend
========================

.. code-block:: c++

  struct c_expr
  {
    /* The value of the expression.  */
    tree value;

    /* [...snip...] */

    /* The source range of this expression.  This is redundant
       for node values that have locations, but not all node kinds
       have locations (e.g. constants, and references to params, locals,
       etc), so we stash a copy here.  */
    source_range src_range;

    /* [...snip...] */

  };


Workaround in C++ frontend
==========================

.. code-block:: c++

  /* A tree node, together with a location, so that we can track locations
     (and ranges) during parsing.
     The location is redundant for node kinds that have locations,
     but not all node kinds do (e.g. constants, and references to
     params, locals, etc), so we stash a copy here.  */
  class cp_expr
  {
  public:
    cp_expr () :
      m_value (NULL), m_loc (UNKNOWN_LOCATION) {}

    cp_expr (tree value) :
      m_value (value), m_loc (EXPR_LOCATION (m_value)) {}

   cp_expr (tree value, location_t loc):
      m_value (value), m_loc (loc) {}

    /* [...snip...] */
  };


Current state of workarounds in gcc 7
=====================================

=============== ====================================
When            Best source of location_t in gcc 7
=============== ====================================
C frontend      c_expr, vec<location_t> at callsites
C++ frontend    cp_expr
generic tree    EXPR_LOCATION ()
gimple          EXPR_LOCATION ()
gimple-SSA      EXPR_LOCATION ()
RTL             EXPR_LOCATION ()
=============== ====================================


Going back to our example
=========================

.. code-block:: c

   int test (int first, int second)
   {
     return foo (100, first * 42, second);
   }

.. nextslide::
   :increment:

``first * 42`` is a :cpp:enumerator:`MULT_EXPR`, which has a
:c:type:`location_t`:

.. code-block:: c

     return foo (100, first * 42, second);
                      ~~~~~~^~~~

and this compound location is retained past the frontend:

=============== ====================================
When            Location of MULT_EXPR
=============== ====================================
C frontend      Available
C++ frontend    Available
generic tree    Available
gimple          Available
gimple-SSA      Available
RTL             Available
=============== ====================================

.. nextslide::
   :increment:

``100`` is usage of an :c:type:`INTEGER_CST`; the location:

.. code-block:: c

     return foo (100, first * 42, second);
                 ^~~

is tracked via workarounds within the frontend, but doesn't
make it into generic tree:

=============== ====================================
When            Location of INTEGER_CST param
=============== ====================================
C frontend      c_expr, vec<location_t> at callsites
C++ frontend    cp_expr, but not at callsites
generic tree    Not available
gimple          Not available
gimple-SSA      Not available
RTL             Not available
=============== ====================================

.. nextslide::
   :increment:

Similarly ``second`` is a usage of a :c:type:`PARM_DECL`; the location:

.. code-block:: c

     return foo (100, first * 42, second);
                                  ^~~~~~

is tracked via workarounds within the frontend, but doesn't
doesn't survive past the frontend:

=============== ====================================
When            Location of PARM_DECL at callsite
=============== ====================================
C frontend      c_expr, vec<location_t> at callsites
C++ frontend    cp_expr, but not at callsites
generic tree    Not available
gimple          Not available
gimple-SSA      Not available
RTL             Not available
=============== ====================================


Problem: emitting warnings from the middle-end
==============================================

The missing location information means we can't
always emit useful locations for diagnostics in the middle-end.

TODO: example


Concrete example: bad arguments at a callsite
=============================================

.. code-block:: c

   extern int callee (int one, const char *two, float three);

   int caller (int first, int second, float third)
   {
     return callee (first, second, third);
   }

.. nextslide::
   :increment:

GCC 7's C++ FE reports::

  test.c: In function ‘int caller(int, int, float)’:
  test.c:5:38: error: invalid conversion from ‘int’ to ‘const char*’
  [-fpermissive]
   return callee (first, second, third);
                                      ^
  test.c:1:12: note:   initializing argument 2 of ‘int callee(int,
  const char*, float)’
   extern int callee (int one, const char *two, float three);
              ^~~~~~

.. nextslide::
   :increment:

GCC 7's C FE does better::

  test.c: In function ‘caller’:
  test.c:5:25: warning: passing argument 2 of ‘callee’ makes pointer
  from integer without a cast [-Wint-conversion]
     return callee (first, second, third);
                           ^~~~~~
  test.c:1:12: note: expected ‘const char *’ but argument is of type
  ‘int’
   extern int callee (int one, const char *two, float three);
              ^~~~~~

* C FE correctly highlights the bogus arg at the callsite
  (due to the `vec<location_t>` workaround)

* Like the C++ frontend, it doesn't underline the pertinent parameter
  at the decl of the callee.

.. nextslide::
   :increment:

The ideal: highlight both argument and param::

  test.c: In function ‘caller’:
  test.c:5:25: warning: passing argument 2 of ‘callee’ makes pointer
  from integer without a cast [-Wint-conversion]
     return callee (first, second, third);
                           ^~~~~~
  test.c:1:12: note: expected ‘const char *’ but argument is of type
  ‘int’
   extern int callee (int one, const char *two, float three);
                               ^~~~~~~~~~~~~~~


Solutions for gcc 8
===================

* extend the workarounds to cover these cases

* add tracking of the missing locations

* preserving locations of all expressions into the middle-end
  (and beyond?)


Solution: using vec<location_t> * in more places
================================================

Committed gcc 8 patch:

* r251238: "c-family/c/c++: pass optional vec<location_t> to c-format.c"
  (2017-08-18)

  * https://gcc.gnu.org/ml/gcc-patches/2017-08/msg01164.html

.. nextslide::
   :increment:

This takes the C frontend from e.g.::

    printf("hello %i %i %i ", foo, bar, baz);
                     ~^
                     %s

to::

    printf("hello %i %i %i ", foo, bar, baz);
                     ~^            ~~~
                     %s


Solution: use vec<location_t> * in C++ frontend
===============================================

Proposed gcc 8 patch:

* "[PATCH] C++: use an optional vec<location_t> for callsites"
  (2017-08-23)

  *  https://gcc.gnu.org/ml/gcc-patches/2017-08/msg01392.html

.. nextslide::
   :increment:

This fixes the location at the callsite, for C++ frontend warnings,
so that::

  error: invalid conversion from 'int' to 'const char*' [-fpermissive]
     return callee (first, second, third);
                                        ^

becomes::

  error: invalid conversion from 'int' to 'const char*' [-fpermissive]
     return callee (first, second, third);
                           ^~~~~~

Doesn't help for the middle-end.


Solution: on-the-side parse tree ("BLT")
========================================

Patch to C/C++ frontends to retain more information
about what was seen during parsing.

* "[PATCH 00/17] RFC: New source-location representation;
  Language Server Protocol" (2017-07-24)

  * https://gcc.gnu.org/ml/gcc-patches/2017-07/msg01448.html

.. nextslide::
   :increment:

Screenshot of dump:

* ``-fdump-blt``

* `https://dmalcolm.fedorapeople.org/gcc/2017-07-24/fdump-blt.html
  <_static/fdump-blt.html>`_

.. nextslide::
   :increment:

* tree-like hierarchy of nodes

* nodes have source ranges

* each node has an ID, corresponding to non-terminals in the C/C++
  grammars

  * e.g. "struct-declaration", "parameter-list"

  * these are just an enum

* there's a sparse two-way mapping between these nodes and the
  regular "tree" world

  * can go from a "tree" to find its BLT node, then navigate
    the BLT hierarchy (in a lang-specific way) to locate BLT
    nodes of interest, and hence locations

.. nextslide::
   :increment:

* an additional tree of parse information

  * much more concrete than our "tree" type, but

  * not quite the full concrete parse tree.

  * somewhere between an AST and a CPT (hence "BLT")

    * name ideas?

  * optional ("-fblt" currently)

    * (see later in talk for discussion of memory tradeoffs)

.. nextslide::
   :increment:

BLT is complementary to our existing IR:

  * captures the locations the FEs are currently throwing away

  * doesn't bother "looking inside functions": we already have
    location information there (to avoid bloating representation)

    * could handle the insides of functions if we wanted to

.. nextslide::
   :increment:

* started as a experiment to debug the recursive descent through the
  C and C++ parsers.

* a possible way of supporting IDEs (e.g. via LSP)

  * patchkit has a proof-of-concept of an LSP server

    * anyone want to pick this up and run with it?


Aside: Language Server Protocol
===============================

* JSON-RPC protocol from Microsoft for a compiler to implement
  language services for an IDE

  * https://github.com/Microsoft/language-server-protocol


Demo of LSP
===========

.. notes:

   sudo yum install pygtk2 pygtksourceview
   pip install json-rpc

   cd /home/david/nomad-coding/c64-working-copies/gcc-git-lsp/build/gcc
   ./xgcc -B. -c ../../src/gcc/testsuite/gcc.dg/lsp/test.c -flsp=4000 -fblt -wrapper gdb,--args^C

   cd /home/david/nomad-coding/c64-working-copies/gcc-git-lsp/src
   python gcc/testsuite/gcc.dg/lsp/toy-ide.py

Toy IDE (~150 lines of PyGTK code):

.. image:: /_static/toy-ide-before.png
   :scale: 50%

Click on usage of "struct foo" and click on "Goto Definition"

.. nextslide::
   :increment:

IDE makes request to cc1::

  cc1: note: received http request: ‘POST /jsonrpc HTTP/1.\x0d\x0aHost
  : localhost:400\x0d\x0aConnection: keep-aliv\x0d\x0aAccept-Encoding:
  gzip, deflat\x0d\x0aAccept: */\x0d\x0aUser-Agent: python-requests/2.
  13.\x0d\x0acontent-type: application/jso\x0d\x0aContent-Length: 18\x
  0d\x0a\x0d\x0a{"jsonrpc": "2.0", "params": {"position": {"line": 8,
  "character": 17}, "textDocument": {"uri": "../../src/gcc/testsuite/g
  cc.dg/lsp/test.c"}}, "method": "textDocument/definition", "id": 8}’

i.e. this JSON-RPC request::

  {"jsonrpc": "2.0",
   "params": {"position": {"line": 8, "character": 17},
              "textDocument": {"uri": "../../src/gcc/testsuite/gcc.dg/lsp/test.c"}},
   "method": "textDocument/definition",
   "id": 8}

.. nextslide::
   :increment:

cc1 uses BLT to locate what's at the file-line-column

cc1 uses BLT to locate the definition of it

.. nextslide::
   :increment:

cc1 issues RPC response::

  [{"range": {"end": {"character": 1, "line": 6},
              "start": {"character": 0, "line": 2}},
     uri": "../../src/gcc/testsuite/gcc.dg/lsp/test.c"}]

like this::

  cc1: note: sending http response: ‘HTTP/1.1 200 O\x0d\x0aContent-Len
  gth:13\x0d\x0a\x0d\x0a[{"range": {"end": {"character": 1, "line": 6}
  , "start": {"character": 0, "line": 2}}, "uri": "../../src/gcc/tests
  uite/gcc.dg/lsp/test.c"}]’

.. nextslide::
   :increment:

IDE highlights the returned region of source:

.. image:: /_static/toy-ide-after.png
   :scale: 50%

Aside: Language Server Protocol
===============================

* What the patch kit implements:

  * One method call out of dozens ("where is this struct declared?"),
    messily

* What the patch kit doesn't implement:

  * all of the other method calls

  * change-monitoring/editing

  * the idea of where the "truth" of each source file is
    (filesystem vs memory)

  * coping with more than one source file and language at a time

  * probably a whole bunch of other stuff


Back to more mundane uses of BLT...
===================================


Using BLT to improve our diagnostics
====================================

Before::

  test.c: In function ‘caller’:
  test.c:5:25: warning: passing argument 2 of ‘callee’ makes pointer
  from integer without a cast [-Wint-conversion]
     return callee (first, second, third);
                           ^~~~~~
  test.c:1:12: note: expected ‘const char *’ but argument is of type ‘int’
   extern int callee (int one, const char *two, float three);
              ^~~~~~

.. nextslide::
   :increment:

With BLT capturing the param locations::

  test.c: In function ‘caller’:
  test.c:5:25: warning: passing argument 2 of ‘callee’ makes pointer
  from integer without a cast [-Wint-conversion]
     return callee (first, second, third);
                           ^~~~~~
  test.c:1:12: note: expected ‘const char *’ but argument is of type ‘int’
   extern int callee (int one, const char *two, float three);
                               ^~~~~~~~~~~~~~~

.. nextslide::
   :increment:

Also in the patch kit:

* Highlighting the return type in the function defn
  when compaining about mismatches, e.g.:

  Before:

.. code-block:: console

    warning: 'return' with a value, in function returning void
       return 42;
              ^~
    note: declared here
     void test_1 (void)
          ^~~~~~

.. nextslide::
   :increment:

After:

.. code-block:: console

    warning: 'return' with a value, in function returning void
       return 42;
              ^~
    note: the return type was declared as 'void' here
     void test_1 (void)
     ^~~~

.. nextslide::
   :increment:

Also in the patch kit: fix-it hints to -Wsuggest-override:

.. code-block:: diff

       test.cc:16:15: warning: ‘virtual void B::f()’ can be marked
       override [-Wsuggest-override]
         virtual void f();
                      ^
                          override
       --- test.cc
       +++ test.cc
       @@ -13,5 +13,5 @@
        {
          B();
          virtual ~B();
       -  virtual void f();
       +  virtual void f() override;
        };

.. nextslide::
   :increment:

Ideas for other uses of this infrastructure (not yet done):

* C++: highlight the "const" token (or suggest a fix-it hint)
  when you have a missing "const" on the *definition* of a member
  function that was declared as "const" (I make this mistake
  all the time).

* C++: add a fix-it hint to -Wsuggest-final-methods

* highlight bogus attributes

* add fix-it hints suggesting missing attributes

* ...etc, plus those "cousins of a compiler" ideas mentioned above.

* other ideas?


.. nextslide::
   :increment:

.. code-block:: c++

  class blt_node
  {
    /* [... lots of methods/accessors ...]  */
  private:
    enum blt_kind m_kind;
    blt_node *m_parent;
    blt_node *m_first_child;
    blt_node *m_last_child;
    blt_node *m_prev_sibling;
    blt_node *m_next_sibling;
    location_t m_start;
    location_t m_finish;
    tree m_tree;
  };

.. nextslide::
   :increment:

Current, unoptimized content (x86_64 host):

.. code-block:: console

  (gdb) p sizeof(blt_node)
  $1 = 64

* easy win: reorder fields for better packing

* currently has lots of pointers, supporting editing

  * could save some of these, making them immutable after construction

  * or pointer compression (indexes rather than pointers)

* etc

BLT Design Questions
====================

* do we support editability? (can save memory if we don't)

* how much should we capture?

  * EVERYTHING?  (inside function bodies?  individual tokens?)

  * or just some subset

    * I favor capturing the things that we're currently missing,
      wherever it allows improvements to our diagnostics
      ("pragmatic approach"?)

.. nextslide::
   :increment:

* what's the lifetime of the BLT nodes?  when do we delete them?

  * maybe a :c:type:`blt_context` containing an obstack?

* do we store BLT information in LTO?  (I'm thinking "no")

* do we store BLT information in PCH?  (I'm not sure)

.. nextslide::
   :increment:

Should we store token pointers rather than location_t?

.. code-block:: c++

  class blt_node
  {
    /* [...]  */
  #if 1
    location_t m_start;
    location_t m_finish;
  #else
    // C++: tokens are currently released after lexing...
    cp_token *m_first_token;
    cp_token *m_last_token;
    // C: tokens are released *during* lexing
  #endif
    /* [...]  */

  };


GCC 8 plan for BLT
==================

  * I hope to come up with a subset of this work (before
    close of stage1) that improves diagnostics, with no
    noticeable effect on compile time/memory

  * mothballing the LSP work for now (any volunteers?)


What to do about EXPR_LOCATION?
===============================

How to reliably get at locations from middle-end?

Possible solutions (see next slides):

* add wrapper tree nodes?

* embedding location_t in tcc_constant?

* extrinsic locations? ("tloc" vs "tree")

* other ideas?


Possible solution: new tree node?
=================================
* wrapper node

  * should it be a new kind of tree node, or should we
    use ``NOP_EXPR`` or ``CONVERT_EXPR``?

  * my current working copy adds a new node kind (``DECL_USAGE_EXPR``)

Status:

  * work-in-progress

    * examples in the above slides work, but...

    * ...much of testsuite fails, and:

    * ...doesn't yet bootstrap

.. note to self:
   working copy: /home/david/coding-3/gcc-git-expr-vs-decl/src

.. nextslide::
   :increment:

Lots of issues:

* what about folding?

* a new tree code?  what about the hundreds of

  .. code-block:: c

     switch (TREE_CODE (node))

* impact on memory usage?  (not yet known; still trying to get it to work)

* how do the gimple representations interact with SSA and with optimization?

.. nextslide::
   :increment:

.. code-block:: c

     return foo (100, first * 42, second);

GENERIC, status quo:

.. blockdiag::

  diagram {

    orientation = portrait;

    class has_location;
    class no_location  [color = yellow, style = dotted];

    CALL_EXPR <- 100, MULT_EXPR, second;
    MULT_EXPR <- first, 42;

    CALL_EXPR[class="has_location"]
    MULT_EXPR[class="has_location"]
    100[class="no_location"]
    first[class="no_location"]
    42[class="no_location"]
    second[class="no_location"]
  }

.. nextslide::
   :increment:

.. code-block:: c

     return foo (100, first * 42, second);

GENERIC, with wrapper nodes:

.. blockdiag::

  diagram {

    orientation = portrait;

    class has_location;
    class no_location  [color = yellow, style = dotted];
    class wrapper [color = pink, label="WRAPPER"];

    CALL_EXPR <- w_100, MULT_EXPR, w_second;
    w_100 <- 100;
    MULT_EXPR <- w_first, w_42;
    w_first <- first;
    w_42 <- 42;
    w_second <- second;

    CALL_EXPR[class="has_location"]
    MULT_EXPR[class="has_location"]

    w_100[class="wrapper"]
    w_first[class="wrapper"]
    w_42[class="wrapper"]
    w_second[class="wrapper"]

    100[class="no_location"]
    first[class="no_location"]
    42[class="no_location"]
    second[class="no_location"]
  }

.. nextslide::
   :increment:

.. code-block:: c

     return foo (100, first * 42, second);

GIMPLE, status quo:

.. code-block:: c

  _1 = first * 42;
  D.1799 = foo (100, _1, second);
  return D.1799;

.. nextslide::
   :increment:

GIMPLE, status quo:

.. code-block:: c

  _1 = first * 42;
  D.1799 = foo (100, _1, second);
  return D.1799;

GIMPLE idea 1: don't flatten the wrappers:

.. code-block:: c

  _1 = wrapper_1(first) * wrapper_2(42); // using the wrapper nodes
  D.1799 = foo (wrapper_3(100), _1, wrapper_4(second));
  return D.1799;

.. nextslide::
   :increment:

GIMPLE, status quo:

.. code-block:: c

  _1 = first * 42;
  D.1799 = foo (100, _1, second);
  return D.1799;

GIMPLE idea 2, turning wrappers into temporaries:

.. code-block:: c

  _w_first = first;  // this stmt retains the location of the usage of "first"
  _w_42 = 42; // likewise for the usage of "42"
  _1 = _w_first * _w_42;
  _w_second = second; // likewise
  D.1799 = foo (_w_100, _1, _w_second);
  return D.1799;

.. nextslide::
   :increment:

GIMPLE SSA, status quo:

.. code-block:: c

  _1 = first_2(D) * 42;
  _6 = foo (100, _1, second_4(D));

GIMPLE SSA with idea 1 (unflattened wrappers):

.. code-block:: c

  _1 = wrapper_1(first) * wrapper_2(42);
  _6 = foo (wrapper_3(100), _1, wrapper_4(second));

.. Presumably to retain location information we'd need to add
   ``location_t`` values to SSA_NAME...

.. nextslide::
   :increment:

GIMPLE SSA, status quo:

.. code-block:: c

  _1 = first_2(D) * 42;
  _6 = foo (100, _1, second_4(D));

GIMPLE SSA with idea 2 (flattened wrappers):

.. code-block:: c

  _w_first_1 = first;
  _w_42_1 = 42;
  _1 = _w_first_1 * _w_42_1;
  _w_second_1 = second;
  _6 = foo (_w_100_1, _1, _w_second_1);

The def-stmts for the wrappers have their ``location_t``.

.. nextslide::
   :increment:



Possible solution: embedding location_t in tcc_constant?
========================================================

* I have a patch to do this.

* ...but what about shared constants?

* could be used with the wrapper nodes, or the wrapper nodes could
  make this redundant


Possible solution: extrinsic locations ("tloc")
===============================================

* *no* exprs have locations

  * ``EXPR_LOCATION`` goes away

* *uses* of nodes have locations, rather than the nodes
  themselves

* convert most uses of "tree" to be
  "tree_and_loc"/"tnl"/"tloc"

  * "tloc" has same number of chars as "tree"

.. nextslide::
   :increment:

.. code-block:: c++

  struct tloc
  {
    tree node;
    location_t loc;

    tloc () : node (NULL), loc (UNKNOWN_LOCATION) {}
    tloc (tree node_) : node (node_), loc (UNKNOWN_LOCATION) {}
    tloc (tree node_, location_t loc_) : node (node_), loc (loc_) {}

    /* FIXME: for now.  */
    operator tree () const { return node; }
    //operator const_tree () const { return node; }
    tree operator -> () const { return node; }
  };

.. nextslide::
   :increment:

Status:

  * doesn't even compile yet

    * e.g. issues with :c:type:`tree` vs :c:type:`const_tree`

.. note to self:
   working copy: /home/david/coding-3/gcc-git-extrinsic-locations/src

Unlikely solution: extrinsic locations ("tree")
===============================================

"tree" is currently a typedef to a pointer:

.. code-block:: c++

  typedef union tree_node *tree;

What if it was instead something like:

.. code-block:: c++

  struct tree
  {
    union tree_node *node;
    location_t loc;
  };


Missing locations: plan for GCC 8
=================================

* fix existing workarounds to work better

* the BLT work discussed earlier

* continue investigating wrapper nodes

* any other possible solutions I haven't thought of?


Optimization Remarks
====================

How do advanced users ask for more information on what GCC's optimizers
are doing?

e.g.

* "Why isn't this loop being vectorized?"

* "Did this function get inlined?  Why?"

etc

.. nextslide::
   :increment:

Current UI:

  * turn on dump flags

  * examine ``foo.c.SOMETHING``

    * where "SOMETHING" is undocumented, and changes from revision to
      revision of the compiler (e.g. "foo.c.029t.einline")

  * no easy way to parse (both for humans and scripts)

  * what is important, and what isn't?

    * e.g. "only tell me about the hot loops"

.. nextslide::
   :increment:

Possible UI:

* a simple way to enable sending optimization information through the
  diagnostic subsystem, e.g.::

  -Rvectorization -fdiagnostics-hotness-threshold=500

* easy-to-read output e.g.::

    foo.c:23:2: remark: unable to vectorize this loop...
    [-Rvectorization, hotness=1000]
     for (i = 0; i < n; i++)
         ^
    foo.c:24:4: note: ...due to this read
       a[i] = b[i] * some_global;
                     ^~~~~~~~~~~

.. nextslide::
   :increment:

Taking another leaf from clang::

  -fsave-optimization-record -foptimization-record-file=foo.yaml

(perhaps with a compatible format?  they have viewers)

.. nextslide::
   :increment:

What should the internal API look like?

e.g.:

.. code-block:: c++

  if (!some_test_for_validity_of_optimization_p (...))
    {
      if (failed_vectorization_remark (loop_stmt))
        inform (read_stmt->location, "...due to this read");
      return false;
    }


.. code-block:: c++

  extern bool remark (location_t, int, const char *, ...)
    ATTRIBUTE_GCC_DIAG(3,4);

  extern bool remark (gimple *, int, const char *, ...)
    ATTRIBUTE_GCC_DIAG(3,4);

  // or pass in gcov_type or a frequency?


How to take this forward?
=========================

Is it acceptable to build up a parallel API:

.. code-block:: c++

  if (dump_file && (dump_flags & TDF_DETAILS))
    {
      fprintf (dump_file,
               "can't frobnicate this stmt:\n");
      print_gimple_stmt (dump_file, stmt, 0, 0);
    }
  remark (loop_stmt, OPT_remark_foo,
          "can't frobnicate this stmt");


Tracking cloned statements
==========================

The idea (this is a work-in-progress):

.. code-block:: console

  bar.c:12:5: remark: unable to vectorize loop [-Rvectorization,
  hotness=1000]
    for (i = 0; i < n; i++)
        ^
  foo.c:23:2: note: ...in inlined copy of 'init' here [einline]
    init (&something);
    ~~~~~^~~~~~~~~~~~

i.e. stash away extra info in the :c:type:`location_t` to describe
where the code we're optimizing came from (injecting the ``note`` above).

.. nextslide::
   :increment:

Class hierarchy for describing code cloning events:

.. code-block:: c++

  /* Base class for describing a code-cloning event.  */

  struct GTY(()) cloning_info
  {
    enum location_clone_kind m_kind;
    opt_pass GTY((skip)) *m_pass;

    /* Hook for adding a note to a diagnostic.  */
    virtual void describe (diagnostic_context *dc) = 0;

    /* [...snip...] */
  };

.. nextslide::
   :increment:

Example subclass: inlining:

.. code-block:: c++

  struct inlining_info : public cloning_info
  {
    inlining_info (opt_pass *pass, source_location call_loc, tree fndecl)
    : cloning_info (LOCATION_CLONE_INLINING, pass),
      m_call_loc (call_loc), m_fndecl (fndecl) {}

    void describe (diagnostic_context *dc) FINAL OVERRIDE;

    location_t m_call_loc;
    tree m_fndecl;
  };

.. nextslide::
   :increment:

Example subclass: loop peeling (hand-waving here):

.. code-block:: console

  bar.c:12:5: remark: unable to vectorize loop [-Rvectorization,
  hotness=1000]
    for (i = 0; i < n; i++)
        ^
  bar.c:12:5: note: ...in peeled epilog copy of loop, for elements
  after last multiple of 8 [slp]

.. code-block:: c++

  struct peeled_loop_info : public cloning_info
  {
    /* TODO.  */

    void describe (diagnostic_context *dc) FINAL OVERRIDE;

  };

.. nextslide::
   :increment:

RAII way to register a "cloning event"

e.g. within tree-inline.c:expand_call_inline:

.. code-block:: c++

   auto_pop_cloning_info ci (new inlining_info (current_pass,
                                                call_stmt->location,
                                                fn);

Pushes/pops the cloning event:

  * all cloned gimple statements get a new :c:type:`location_t` that
    refers to the cloning event, until the RAII object goes out of scope.


Other stuff
===========

Overloading to get rid of "error_at_rich_loc" verbosity?

Adding a fix-it hint currently involves changing e.g.:

.. code-block:: c++

      error_at (token->location,
                "unknown type name %qE; did you mean %qs?",
                token->value, hint);

to:

.. code-block:: c++

      gcc_rich_location richloc (token->location);
      richloc.add_fixit_replace (hint);
      error_at_rich_loc (&richloc,
                         "unknown type name %qE; did you mean %qs?",
                         token->value, hint);

.. nextslide::
   :increment:

Could use overloading of ``error_at`` to simplify it to just from this:

.. code-block:: c++

      error_at (token->location,
                "unknown type name %qE; did you mean %qs?",
                token->value, hint);

to:

.. code-block:: c++

      gcc_rich_location richloc (token->location);
      richloc.add_fixit_replace (hint);
      error_at (&richloc,
                "unknown type name %qE; did you mean %qs?",
                token->value, hint);


Summary
=======

* Problems with location-tracking in our internal representation(s)

  * Possible solutions

    * better workarounds

    * BLT

    * experiments with preserving locations into middle-end

  * (detour into LSP, for IDE support)

* Better information for advanced users on what optimizers are doing

  * "optimization remarks"

  * tracking cloning of statements (e.g. inlining, vectorization, etc)


Next steps
==========

Try to get some/all of this in good enough shape for GCC 8 before
stage 1 closes.


Questions and Discussion
========================

Thanks for listening!

(thanks to Red Hat for funding this work)

URL for these slides: https://dmalcolm.fedorapeople.org/presentations/cauldron-2017/
