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


There's room for improvement
============================

The C and C++ frontends discard a lot of source location information:

* locations of *uses* of decls and of constants

* locations of the individual parameters within function decls

* locations of attributes of a function

* locations of individual tokens like close parens and
  semicolons

This limits our ability to implement "cousins" of a
compiler on top of the GCC codebase (e.g. code refactoring tools,
code reformatting tools, IDE support daemons, etc), since much of the
useful location information is discarded at parse time.

.. makes our diagnostics harder to read that they could be

Leaf nodes in many expressions don't have location information
==============================================================

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

Current state of workarounds
============================

=============== ====================================
When            Best source of location_t
=============== ====================================
C frontend      c_expr, vec<location_t> at callsites
C++ frontend    cp_expr
generic tree    EXPR_LOCATION ()
gimple          EXPR_LOCATION ()
gimple-SSA      EXPR_LOCATION ()
RTL             EXPR_LOCATION ()
=============== ====================================

.. nextslide::
   :increment:

Simple example:

.. code-block:: c

   void test (int src)
   {
     dst = src * 42;
     return dst;
   }

Within this line:

.. code-block:: c

  dst = src * 42;


.. nextslide::
   :increment:

``src * 42`` is a :cpp:enumerator:`MULT_EXPR`, which has a
:c:type:`location_t`:

.. code-block:: c

  dst = src * 42;
        ~~~~^~~~

and this compound location is retained.

.. nextslide::
   :increment:

``src`` is a usage of a :c:type:`PARM_DECL`; the location:

.. code-block:: c

  dst = src * 42;
        ^~~

doesn't survive past the frontend.

.. nextslide::
   :increment:

Similarly, ``42`` is usage of an :c:type:`INTEGER_CST`; the location:

.. code-block:: c

  dst = src * 42;
              ^~

doesn't survive past the frontend.

.. TODO
   - example of bad params at callsite
   - not every expression has a location
   - missing stuff from middle-end
   - what's the PR?



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
  test.c:5:38: error: invalid conversion from ‘int’ to ‘const char*’ [-fpermissive]
   return callee (first, second, third);
                                      ^
  test.c:1:12: note:   initializing argument 2 of ‘int callee(int, const char*, float)’
   extern int callee (int one, const char *two, float three);
              ^~~~~~

.. nextslide::
   :increment:

The C FE does better::

  test.c: In function ‘caller’:
  test.c:5:25: warning: passing argument 2 of ‘callee’ makes pointer from integer without a cast [-Wint-conversion]
     return callee (first, second, third);
                           ^~~~~~
  test.c:1:12: note: expected ‘const char *’ but argument is of type ‘int’
   extern int callee (int one, const char *two, float three);
              ^~~~~~

Note how the C FE has correctly highlighted the bogus arg at the callsite
(this is due to the `vec<location_t>` passed around when the call is created)

Like the C++ frontend, it doesn't underline the pertinent parameter
at the decl of the callee.

.. nextslide::
   :increment:

The ideal: highlight both argument and param::

  test.c: In function ‘caller’:
  test.c:5:25: warning: passing argument 2 of ‘callee’ makes pointer from integer without a cast [-Wint-conversion]
     return callee (first, second, third);
                           ^~~~~~
  test.c:1:12: note: expected ‘const char *’ but argument is of type ‘int’
   extern int callee (int one, const char *two, float three);
                               ^~~~~~~~~~~~~~~

The current solution
====================

TODO:

* passing around :c:type:`location_t` values where they seem useful

* falling back to the :c:data:`input_location` global

* :c:type:`c_expr` and :c:type:`cp_expr`

* :c:type:`vec<location_t>` approach for callsites


Problem: emitting warnings from the middle-end
==============================================


Solution: using vec<location_t> * in more places
================================================

Recent patch to c-format.c


Possible solution: use vec<location_t> * in C++ frontend?
=========================================================


Possible solution: new tree node?
=================================
* wrapper node


Possible solution: embedding location_t in tcc_constant?
========================================================


Possible solution: extrinsic locations ("tloc")
===============================================

(no exprs have location; convert most uses of "tree"
to be "tree_and_loc"/"tnl"/"tloc")


Possible solution: on-the-side parse tree ("BLT")
=================================================




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


Extra material
==============
