# libCW v1.1
A library that contains stuff for the cw/cwx calcs

To use:

Add `#include "libcw.h"` to the top of your source file.

(For CWX add `#define IS_CWX` ***BEFORE*** `#include "libcw.h"`)

I do not reccomend mixing this with some of libc's stuff especially when using the heap, you are able to easily change the addresses of things to account for using libc with this library.

Read `libcw.md` for a more in depth explination on all of the features it provides.

### THIS ONLY WORKS ON CW/CWX CALCULATORS, AND HAS TO BE BUILT USING THE LAPIS TOOLCHAIN
