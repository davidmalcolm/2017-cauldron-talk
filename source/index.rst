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

============================================================
GCC diagnostics and location-tracking improvements for GCC 8
============================================================

GNU Tools Cauldron 2017

David Malcolm <dmalcolm@redhat.com>

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


History of source-location tracking in GCC
==========================================

Source locations are encoded as 32-bit values
(:c:type:`location_t` a.k.a :c:type:`source_location`).

Effectively a key into a database (:c:data:`line_table`).

  * early versions of GCC merely tracked filename and line number

  * column numbers were added in 2004

  * tracking of macro expansions in 2011

  * I added tracking of *ranges* of source code (rather that just points)
    in GCC 6


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

...but we *don't* permaently capture locations of constants and
*uses* of decls::

    return foo (100, first * 42, second)
                ^--              ^-----

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

.. code-block:: c

   /* The source location of this expression.  Non-tree_exp nodes such as
      decls and constants can be shared among multiple locations, so
      return nothing.  */
   #define EXPR_LOCATION(NODE) \
     (CAN_HAVE_LOCATION_P ((NODE)) ? (NODE)->exp.locus : UNKNOWN_LOCATION)

.. nextslide::
   :increment:

Nasty workarounds:

.. code-block:: c

  #define EXPR_LOC_OR_LOC(NODE, LOCUS) (EXPR_HAS_LOCATION (NODE) \
                         ? (NODE)->exp.locus : (LOCUS))

.. code-block:: c

  location_t loc = EXPR_LOC_OR_LOC (src, input_location);

.. nextslide::
   :increment:


Workaround in C frontend
========================

.. code-block:: c

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

.. TODO
   - what's the PR?

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

* add tracking of the missing locations (e.g. param locations within decl)

* more invasive IR changes to preserve locations into the middle-end


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

    * I don't yet have memory-consumption stats

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

.. nextslide::
   :increment:

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

  Before::

    warning: 'return' with a value, in function returning void
       return 42;
              ^~
    note: declared here
     void test_1 (void)
          ^~~~~~

.. nextslide::
   :increment:

After::

    warning: 'return' with a value, in function returning void
       return 42;
              ^~
    note: the return type was declared as 'void' here
     void test_1 (void)
     ^~~~

.. nextslide::
   :increment:

Also in the patch kit: fix-it hints to -Wsuggest-override::

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


What to do about EXPR_LOCATION?
===============================

How to reliably get at locations from middle-end?

TODO: add summary slide about next slides


Possible solution: new tree node?
=================================
* wrapper node

.. note to self:
   working copy: /home/david/coding-3/gcc-git-expr-vs-decl/src


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

Presumably to retain location information we'd need to add
``location_t`` values to SSA_NAME...

.. nextslide::
   :increment:

GIMPLE SSA, status quo:

.. code-block:: c

  _1 = first_2(D) * 42;
  _6 = foo (100, _1, second_4(D));

GIMPLE SSA with idea 2 (unflattened wrappers):

.. code-block:: c

  _w_first_1 = first;
  _w_42_1 = 42;
  _1 = _w_first_1 * _w_42_1;
  _w_second_1 = second;
  _6 = foo (_w_100_1, _1, _w_second_1);

The def-stmts for the wrappers have their ``location_t``.

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

Status:


Possible solution: embedding location_t in tcc_constant?
========================================================

But what about shared constants?


Possible solution: extrinsic locations ("tloc")
===============================================

(no exprs have location; convert most uses of "tree"
to be "tree_and_loc"/"tnl"/"tloc")

.. note to self:
   working copy: /home/david/coding-3/gcc-git-extrinsic-locations/src


Rejected solution: taking BLT much further
==========================================


Other stuff
===========

* overloading to get rid of "error_at_rich_loc" verbosity?


Summary
=======


Next steps
==========


Questions and Discussion
========================

Thanks for listening!
