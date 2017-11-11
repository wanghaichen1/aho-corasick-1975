**A small, documented, easy to use implementation of the Aho-Corasick algorithm.**
------------------------------------------------------------------------------

This implementation strictly follows acurately and step by step the pseudo-code given in the original paper from Aho and Corasick, exquisitely clear and well written.

See Aho, Alfred V.; Corasick, Margaret J. (June 1975). "[Efficient string matching: An aid to bibliographic search](https://pdfs.semanticscholar.org/3547/ac839d02f6efe3f6f76a8289738a22528442.pdf)".
Communications of the ACM. 18 (6): 333–340.

Compared to the implemenation proposed by Aho and Corasick, this one adds several enhancements:

1. First of all, the implementation does not define any assumption on the size of alphabet used.
   Particularly, the alphanet is not limited to 256 signs.
   For instance, if ACM_SYMBOL is defined as 'long long int', then the number of possible signs would be 18446744073709551616.
   For this to be possible, the assertion "for all a such that g(0, a) = fail do g(0, a) <- 0" at the end of algorithm 2 can not be fulfilled
   because it would require to set g(0, a) for all the values of 'a' in the set of possible values of the alphabet,
   and thus allocate (if not exhaust) a lot of memory.
   Therefore, g(0, a) is kept equal to fail (i.e. a or g(0, a) is kept undefined) for all a not yet defined by keyword registration.
   Nevertheless, for the state machine to work properly, it must behave as if g(0, a) would be equal to 0 whenever g(0, a) = fail.
   Algorithms 1 and 3 (called after algorithm 2) must be adapted accordingly (modifications are tagged with [1], [2] and [3] in the code):

      - [1] g(0, a) = (resp. !=) 0 must be replaced by: g(0, a) = (resp. !=) fail
      - [2] g(state, a) = fail must be replaced by: g(state, a) = fail and state != 0
      - [3] s <- g(state, a) must be replaced by: if g(state, a) != fail then s <- g(state, a) else s <- 0

2. To reduce the memory footprint, this implementation does not stores output keywords associated to states.
   It rather reconstructs matching keywords by traversing the branch of the tree backward (see ACM_get_match).
3. This implementation permits to search for keywords even though all keywords have not been registered yet.
   To achieve this, failure states are reconstructed after every registration of a next keyword
   (see ACM_register_keyword which alternates calls to algorithms 2 and 3.)
4. This implemtation keeps track of the rank of a registered keyword as returned by ACM_get_match().
   This can be used as a unique identifiant of a keyword for a given machine state.
5. If ACM_ASSOCIATED_VALUE is defined at compile time, then values can be associated to registered keywords, and retreived with the found keywords:
      - two arguments are added when ACM_register_keyword is called: a pointer to a previously allocated value, and a pointer to function for
        deallocation of the associated value. This function will be called when the state machine will be release by ACM_release.
      - one argument is added when ACM_get_match is called: the address of a pointer to an associated value.
        The pointer to associated value is modified by ACM_get_match to the address of the value associated to the keyword.

Usage:
-----
First, initialize the finite state machine with a set of keywords to be searched for:

1. Define the type ACM_SYMBOL with the type of symbols that constitute keywords. char or int should match most needs.
   Either define ACM_SYMBOL direcly in the user program or include a user defines aho_corasick_symbol.h file.
   E.g.:
      - \#define ACM_SYMBOL char (simple choice if the algorithm is compiled as a private module).
      - \#include "aho_corasick_symbol.h" (better choice if the algorithm is compiled as an object or library, and not as a private module).
2. Insert "aho_corasick.h"
3. Initialize a state machine: InitialState * M = 0;
4. Add keywords (of type Keyword) to the state machine calling ACM_register_keyword() repeatedly.
      - ACM_KEYWORD_SET can be used to initialize keywords with a single statement.
      - ACM_nb_matches() returns the number of keywords already inserted in the state machine.

Then, search for keywords in an input text:

5. Initialize an internal machine state to M: InternalState s = M;
6. Initialize a keyword with ACM_KEYWORD_INIT before the first use by ACM_get_match.
7. Inject symbols of the text, one at a time by calling ACM_change_state() on s.
8. After each insertion of a symbol, call ACM_nb_matches() on the internal state s to check if the last inserted symbols match a keyword.
9. If matches were found, retrieve them calling ACM_get_match() for each match.
      - ACM_KEYWORD_LENGTH and ACM_KEYWORD_SYMBOLS can be used to access the length and the content of the match found.
10. After the last call to ACM_get_match(), release to keyword by calling ACM_KEYWORD_RELEASE.

Finally:

11. After usage, release the state machine calling ACM_release() on M.

Note on ACM_SYMBOL
------------------
ACM_SYMBOL is the type of letters of the alphabet that constitutes the keywords.

ACM_SYMBOL can be any basic type (char, int, unisigned long long int for instance), or a user defined structure.

If ACM_SYMBOL is a basic type, the default comparator operator is '=='.
This operator can be overwritten by a unser defined function which takes two arguments of type ACM_SYMBOL and returns an int:

  - the first argument is a symbol from a keyword;
  - the second argument is a symbol from the text into which keywords are searched for.
  - the function should return 1 if symbols are considered equal, 0 otherwise.

E.g. the following code will make string matching case insensitive.

    static int nocaseeq (char k, char t)
    { return tolower (k) == tolower (t); }
    #define ACM_SYMBOL_EQ_OPERATOR nocaseeq

Note if ACM_SYMBOL is a structure (does not apply for basic types such as int or char):
---------------------------------------------------------------------------------------
If ACM_SYMBOL is a structure:

  - An equality operator '==' with signature 'int eq (ACM_SYMBOL a, ACM_SYMBOL b)' should be defined
    in the user program and the name of the function should be defined in macro ACM_SYMBOL_EQ_OPERATOR.
    E.g.: #define ACM_SYMBOL_EQ_OPERATOR myequalityoperaor
  - Keywords are passed by value to ACM_register_keyword().
    Therefore, if ACM_SYBOL contains pointers to allocated memory,
      - a copy operator '=' with signature 'ACM_SYMBOL copy (ACM_SYMBOL a)' should be defined in the user program and
        the name of the function should be defined in macro ACM_SYMBOL_COPY_OPERATOR.
      - a destructor operator with signature 'void dtor (ACM_SYMBOL a)' should be defined in the user program and
        the name of the function should be defined in macro ACM_SYMBOL_DTOR_OPERATOR.

Compilation:
------------
The algorithm can be compiled either as an object (aho_corasick.o) or as a private module (if PRIVATE_MODULE is defined in the user program).
If PRIVATE_MODULE is set in the user program, then:
- the implementation of the algorithm will be compiled in the same compilation unit as the user program, therefore without requiring linkage.
  There is no need to compile aho_corasick.c separately, the source will be include in the user program including aho_corasick.h.
- the warning "The Aho-Corasick algorithm is compiled as a private module." is emitted during compilation.

Files:
------

Source code:

- aho_corasick.h defines the interface. It is fully documented.
- aho_corasick.c defines the implementation. It is fully documented.
  The implementation complies to the original algorithm as published by Aho and Corasick in 1975.

Examples:

- aho_corasick_test.c gives an example of use (words.gz should be gunzip'ed before use).
- words.gz is an input file used by the example.
- aho_corasick_symbol.h is an example of a declaration of the ACM_SYMBOL.

Have fun !