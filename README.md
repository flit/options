# options

C++ command line option library.

The original options library was written by Brad Appleton back in the 90s. It was
updated to a more modern C++ style with STL and Doxygen comments by Chris Reed.

## Synopsis

```cpp
#include <options.h>

char cmdname[], *optv[];
Options opts(cmdname, optv);
```

## Description

The `Options` constructor expects a command-name (usually `argv[0]`) and
a pointer to an array of strings.  The last element in this array _must_
be `NULL`. Each non-`NULL` string in the array must have the following format:

- The 1st character must be the option-name (`c` for a `-c` option).

- The 2nd character must be one of `|`, `?`, `:`, `*`, or `+`.
 - `|` -- indicates that the option takes _no_ argument
 - `?` -- indicates that the option takes an _optional_ argument
 - `:` -- indicates that the option takes a _required_ argument
 - `*` -- indicates that the option takes 0 or more arguments
 - `+` -- indicates that the option takes 1 or more arguments

- The remainder of the string must be the long-option name.

If desired, the long-option name may be followed by one or more
spaces and then by the name of the option value. This name will
be used when printing usage messages. If the option-value-name
is not given then the string `<value>` will be used in usage
messages.

One may use a space to indicate that a particular option does not
have a corresponding long-option.  For example, "`c: `" (or "`c:`")
means the `-c` option takes a value and has _no_ corresponding long-option.

To specify a long-option that has no corresponding single-character
option is a bit trickier: `Options::operator()` still needs an "option-
character" to return when that option is matched. One may use a whitespace
character or a non-printable character as the single-character option
in such a case. (hence "` |hello`" would only match "`--hello`").

### Exceptions to the above

If the 1st character of the string is `-`, then the rest of the
string must correspond to the above format, and the option is
considered to be a hidden-option. This means it will be parsed
when actually matching options from the command-line, but will
_not_ show-up if a usage message is printed using the `usage()` member
function. Such an example might be "`-h|hidden`". If you want to
use any "dummy" options (options that are not parsed, but that
to show up in the usage message), you can specify them along with
any positional parameters to the `usage()` member function.

If the 2nd character of the string is '`\0`' then it is assumed
that there is no corresponding long-option and that the option
takes no argument (hence "`f`", and "`f| `" are equivalent).

```cpp
const char * optv[] = {
    "c:count   <number>",
    "s?str     <string>",
    "x",
    " |hello",
    "g+groups  <newsgroup>",
    NULL
} ;
```

`optv[]` now corresponds to the following:

           usage: cmdname [-c|--count <number>] [-s|--str [<string>]]
                          [-x] [--hello] [-g|--groups <newsgroup> ...]

Long-option names are matched case-insensitive and only a unique prefix
of the name needs to be specified.

Option-name characters are case-sensitive!

## Caveat

Because of the way in which multi-valued options and options with optional
values are handled, it is _not_ possible to supply a value to an option in
a separate argument (different `argv[]` element) if the value is _optional_
and begins with a `-`. What this means is that if an option `-s` takes an
optional value value and you wish to supply a value of `-foo` then you must
specify this on the command-line as `-s-foo` instead of `-s -foo` because
`-s -foo` will be considered to be two separate sets of options.

A multi-valued option is terminated by another option or by the end-of
options. The following are all equivalent (if `-l` is a multi-valued
option and `-x` is an option that takes no value):

    cmdname -x -l item1 item2 item3 -- arg1 arg2 arg3
    cmdname -x -litem1 -litem2 -litem3 -- arg1 arg2 arg3
    cmdname -l item1 item2 item3 -x arg1 arg2 arg3


## Full example

```cpp
#include <options.h>

static const char * optv[] = {
  "H|help",
  "c:count   <number>",
  "s?str     <string>",
  "x",
  " |hello",
  "g+groups  <newsgroup>",
  NULL
} ;

main(int argc, char * argv[]) {
  int  optchar;
  const char * optarg;
  const char * str = "default_string";
  int  count = 0, xflag = 0, hello = 0;
  int  errors = 0, ngroups = 0;

  Options  opts(*argv, optv);
  OptArgvIter  iter(--argc, ++argv);

  while( optchar = opts(iter, optarg) ) {
     switch (optchar) {
     case 'H' :
        opts.usage(cout, "files ...");
        exit(0);
        break;
     case 'g' :
        ++ngroups; break;  //! the groupname is in "optarg"
     case 's' :
        str = optarg; break;
     case 'x' :
        ++xflag; break;
     case ' ' :
        ++hello; break;
     case 'c' :
        if (optarg == NULL)  ++errors;
        else  count = (int) atol(optarg);
        break;
     default :  ++errors; break;
     } //!switch
  }

  if (errors || (iter.index() == argc)) {
     if (! errors) {
        cerr << opts.name() << ": no filenames given." << endl ;
     }
     opts.usage(cerr, "files ...");
     exit(1);
  }

  cout << "xflag=" << ((xflag) ? "ON"  : "OFF") << endl
       << "hello=" << ((hello) ? "YES" : "NO") << endl
       << "count=" << count << endl
       << "str=\"" << ((str) ? str : "No value given!") << "\"" << endl
       << "ngroups=" << ngroups << endl ;

  if (iter.index() < argc) {
     cout << "files=" ;
     for (int i = iter.index() ; i < argc ; i++) {
        cout << "\"" << argv[i] << "\" " ;
     }
     cout << endl ;
  }
}
```

