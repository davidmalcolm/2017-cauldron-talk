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

The C++ FE currently reports::

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

The C FE does better::

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

TODO:
  * what it is
  * what can it do
  * screenshot of dump
    * https://dmalcolm.fedorapeople.org/gcc/2017-07-24/fdump-blt.html

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

What to do about EXPR_LOCATION?
===============================

TODO: add summary slide about next slides


Possible solution: new tree node?
=================================
* wrapper node


Possible solution: embedding location_t in tcc_constant?
========================================================


Possible solution: extrinsic locations ("tloc")
===============================================

(no exprs have location; convert most uses of "tree"
to be "tree_and_loc"/"tnl"/"tloc")


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
