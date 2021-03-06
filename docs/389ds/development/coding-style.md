---
title: "Coding Style"
---

# Coding Style Guidelines
-------------------------

This manual contains coding standards and guidelines for 389 Directory Server contributors.

This project has evolved over a very long time (see [History](../FAQ/history.html)). As a result not all of the existing code conforms to these standards. All new code should adhere to these standards but please do not go back and make wholesale formatting changes to the old code. It just confuses things and is generally a waste of time.

{% include toc.md %}

Why Have A Coding Style?
------------------------

### For easier maintenance

If you're merging changes from a patch it's much easier if everyone is using the same coding style. This isn't the reality for a lot of our code, but we're trying to get to the point where most of it uses the same style.

### Improved readability

Remember, code isn't just for compilers, it's for people, too. If it wasn't for people, we would all be programming in assembly. Coding style and consistency mean that if you go from one part of the code to another you don't spend time having to re-adjust from one style to another. Blocks are consistent and readable and the flow of the code is apparent. Coding style adds value for people who have to read your code after you've been hit by a bus. Remember that.

General Rules
-------------

-   All source and header files should include a copy of the license.
-   C99 is the minimum version we target (no c89 mode).
-   Do not use bare int or long. Use inttypes.h (see Types)
-   Stick to a K&R coding style.
    -   Space after keywords
    -   Curly braces on same line as if/while.
-   We provide slapi_* functions for various tasks - these should be your first reference.
-   Avoid NSPR functions for file, string, memory allocation, etc. if at all possible. They add excess overhead to the platform.
-   All code should be peer reviewed before being checked in.
-   Our code is used internationally, so use simple english in messages.
-   Don't add extra dependencies unless they are needed. Every dependency is a potential "new project" we may need to adopt. If we need a small function, just write that rather than using micro dependencies.
-   Keep messages short, direct, and offer direction. For example:

A bad message is:

    An error occured, please check the error log.

A better message is:

    An error occured processing operation=X: Invalid dn.

A great message is:

    ERROR: Processing add operation=X: DN containing "z" is invalid.

Messages should direct a user or admin where to go to solve the problem, not to just make them aware of it.

Statements
----------

-   if-else statements should have the following form.

        if (<condition>) {     
            /* do some work */     
        }     

        if (<condition>) {     
            /* do some work */     
        } else {     
            /* do some other work */     
        }

-   Balance the braces in the if and else in an if-else statement if either has only one line:

        if (condition) {     
            /*     
             * stuff that takes up more than one     
             * line     
             */     
        } else {     
            /* stuff that only uses one line */     
        }     

-   Avoid last-return-in-else problem. Code should look like this:

        int foo(int bar) {     
            if (something) {     
                /* stuff done here */     
                return 1;            
            }     
        
            return 0;     
        }     

    **NOT** like this:

        int foo(int bar) {     
            if (something) {     
                /* stuff done here */     
                return 1;                 
            } else {     
                return 0;     
            }     
        }     

-   for, while and until statements should take a similar form:

        for (<initialization>; <condition>; <update>) {     
            /* iterate here */     
        }     

        while (<condition>) {     
            /* do some work */     
        }     

-   switch statements:

        switch (<condition>) {     
        case A:     
            /* do work */     
            break;     
        case B:     
            /* do work */     
            break;     
        default:     
            /* do work */     
            break;     
        }     

-   Braces are required on all conditionals and for loops. Code like the following will be rejected at review. The reason for this is it can cause accidental logic mistakes that are hard to detect.

        if (foo)     
            bar();     
        else     
            baz();     


Comments
--------

-   C-style comments are preferred (/\* \*/ instead of //)
    -   Each function should be preceded with a block comment describing what the function is supposed to do.
    -   Block comments should be preceded by a blank line to set it apart. Line up the \* characters in the block comment.

        /*     
         * A block comment.     
         */     

Indentation
-----------

-   There is no limit on characters per line, but excessive wide text should be avoided. This is a very subjective topic, so if you want a hard number, less than 120 chars.
-   Avoid spliting strings across lines: If a message is long, leave it on one line, or make the message clearer and simpler.
-   Indentation is 4 spaces.
-   No tabs, use spaces. Your editor should take care of most of this but in patches tabs stick out like sore thumbs.
-   When wrapping lines, try to break it:
    -   after a comma
    -   before an operator
    -   align the new line with the beginning of the expression at the same level in the previous line
    -   if all else fails, just indent 8 spaces.

        Function(longArgument1, longArgument2, longArgument3,      
                 longArgument4, longArgument5);     

Variable Declarations
---------------------

-   One declaration per line is preferred.

        int32_t foo;     
        int32_t bar;     

    instead of

        int foo, bar;     

-   Initialize at declaration time when possible.
-   Declare variables at the beginning of a block to maintain C99 compatibility.
-   No tabs, use spaces. Your editor should take care of most of this but in patches tabs stick out like sore thumbs.
-   Avoid the use of memset. You can either use {0} for struct or arrays on the stack, or use calloc. memset has signifigant performance impact on systems.

    Slapi_PBlock pb = {0};
    uint64_t arr[10] = {0};
    struct foo *f = slapi_ch_calloc(sizeof(foo));

-   Avoid variable reuse. Free and allocate a new one, or in a loop, declare inside the loop block. Don't reuse structs as this allow corruption to pass through iterations, and breaks dynamic analysis tools.

Types
-----

- Avoid nested struct definitions. This is inefficent for memory allocation and hard to follow the struct contents. IE:

    struct x {
        struct y {
            member z;
        }
    ...
    }


- Avoid void * in all possible cases. Unless you are writing a low level datastructure (you shouldn't be), you should not need this! Having concrete types lets the compiler find mistakes for us - We are only human after all.
- Avoid the use of variable size types. This includes int, long, long long, double etc. Please use concrete types from inttypes.h. (ie int32_t, int_fast32_t, uint64_t)
- Use 64 bit types.
- Only use 32 bit types to maintain compatability with existing / external apis.
- Arrays are indexed by size_t.

Configuration
-------------

This is a description of how we should create configuration options for our server.

- Configuration options should be positive. Consider the following examples:

    feature: on/off
    feature-enable: on/off
    feature-disable: on/off

The first example is the best. We have a feature and a clear english "on/off". We know that feature: on means on, and feature: off means off. The second is the same, but has the word -enable. This is redundant,
as the on-off already conveys this. The third will be REJECTED on review, because it's confusing. feature-disable: on is counter intuitive, meaning it's disabled. This is hard for a native english speaker, let alone a second language speaker.

- Configuration names should be self documenting. For example:

    cachememorysize: 100
    cache: 100

In our example we have a cache memory size. This is a good name as we know this is about the cache, memory and size: We can infer that it's the amount of memory the cache will occupy, and given context by the value as it has a quantifier in it.

The second is bad, because we have no context to what "cache" is related to, be it a size or otherwise. This would require looking documentation.

Remember: DOCUMENTATION exists as an artifact of POOR DESIGN CHOICES. Every piece of documentation represents a possible design flaw in our system.

- Configuration names should be complete english, not abbreviations. For example:

    cachememsz
    cachememorysize

For a non-native speaker the contraction of memory size to memsz would be confusing.

Command line tools
------------------

This describes how we should create command line tools

- Group common activities. Rather than 100 utilites, we should have 1 utility that shows what is possible. These should be grouped together. This enables discoverability. An admin may know about "ns-activate.pl" but not about the password policy tools. However, contrast to dsconf, and when we type dsconf --help, we see a list of *all* of these possibilities. We have enabled the admin to find more tools and help about their system by collating tools. To take this further we can nest this for example:

    dsconf --help
    ...
        plugins ...
        indices ...
        backend ...

    dsconf backend --help
        create
        set_readonly
        set_readwrite
        delete
        backup
        restore

- Command line tools should be memorable. If you have to read the man page every single time, the tool is flawed. You should be able to easily and fluently type the action you want to achieve, and in a way that is friendly.

- Forgiving: Tools should not cause damage if used incorrectly. If a tool has the wrong arguments, or incorrect or mismatched types it should validate these and give guidance.

- Command line tools should be easy to type. Consider:

    dsadm start instance
    dsadm --start --instance=instance

One of these two is easier to type, and more "natural" to use than the other (hint, it's the shorter one).

