# php-memprof

php-memprof profiles memory usage of PHP scripts, and especially can tell which
function has allocated every single byte of memory currently allocated.

This is different from measuring the memory usage before and after a
function call:

``` php
<?php

// script 1

function a() {
    $data = file_get_contents("huge-file");
}

a();

$profile = memprof_dump_array();

```

In script 1, a before/after approach would designate file_get_contents() as huge
memory consumer, while the memory it allocates is actually freed quickly after
it returns. When dumping the memory usage after a() returns, the memprof
approach would show that file_get_contents() is a small memory consumer since
the memory it allocated has been freed at the time memprof_dump_array() is
called.


``` php
<?php

// script 2

function a() {
    global $cache;
    $cache = file_get_contents("huge-file");
}

a();

$profile = memprof_dump_array();
```

In script 2, the allocated memory remains allocated after file_get_contents()
and a() return, and when memprof_dump_array() is called. This time a() and
file_get_contents() are shown as huge memory consumers.

## Dependencies

 * [Judy Libray][3] (e.g. libjudy-dev or judy package)
 * C Library with [malloc hooks][1] (optional; allows to track persistent allocations too)

## Install

    phpize
    ./configure
    make
    make install

## Loading the extension

The extension must be loaded as a zend_extension.

It can be loaded on the command line, just for one script:

    php -dzend_extension=/absolute/path/to/memprof.so script.php

Or permanently, in php.ini:

    zend_extension=/absolute/path/to/memprof.so

## Usage

The memory usage can be dumped by calling the memprof_dump_array() or
memprof_dump_callgrind() functions.

Both tell which functions allocated all the currently allocated memory.

### memprof_dump_callgrind

The memprof_dump_callgrind function dumps the current memory usage to a stream
in callgrind format. The file can then be read with tools such as
[KCacheGrind][2]. Xdebug lists other tools capable of reading
callgrind files: http://xdebug.org/docs/profiler, although they may only work
with xdebug output files.

``` php
<?php
memprof_dump_callgrind(fopen("output", "w"));
?>
```

Here is a KcacheGrind screenshot:

![KCacheGrind screenshot](http://img820.imageshack.us/img820/5530/screenshot3kve.png)

### memprof_dump_pprof

The memprof_dump_pprof function dumps the current memory usage to a stream in
[pprof][4] format.

``` php
<?php
memprof_dump_callgrind(fopen("profile.pprof", "w"));
?>
```

The file can be visualized using [google-perftools][5]'s [``pprof``][4] tool.

Display annotated call-graph in web browser or in ``gv``:

```
$ pprof --web profile.pprof
$ pprof --gv profile.pprof
```

![pprof call-graph screenshot](http://img707.imageshack.us/img707/7697/screenshot3go.png)

Output one line per function, sorted by own memory usage:

```
$ pprof --text profile.pprof
```

### memprof_dump_array

``` php
<?php
$dump = memprof_dump_array();
?>
```

The dump exposes the following informations:

 * Inclusive and exlusive memory usage of functions (counting only the memory
   that has is still in use when memprof_dump_array is called)
 * Inclusive and exlusive blocks count of functions (number of allocated;
   counting only the blocks that are still in use when memprof_dump_array is
   called)
 * The data is presented in call stacks. This way, if a function is called from
   multiple places, it is possible to see which call path caused it to leak the
   most memory

Example output:

    Array
    (
      [memory_size] => 11578
      [blocks_count] => 236
      [memory_size_inclusive] => 10497691
      [blocks_count_inclusive] => 244
      [calls] => 1
      [called_functions] => Array
        (
          [main] => Array
            (
              [memory_size] => 288
              [blocks_count] => 3
              [memory_size_inclusive] => 10486113
              [blocks_count_inclusive] => 8
              [calls] => 1
              [called_functions] => Array
                (
                  [a] => Array
                    (
                      [memory_size] => 4
                      [blocks_count] => 1
                      [memory_size_inclusive] => 10485825
                      [blocks_count_inclusive] => 5
                      [calls] => 1
                      [called_functions] => Array
                        (
                          [b] => Array
                            (
                              [memory_size] => 10485821
                              [blocks_count] => 4
                              [memory_size_inclusive] => 10485821
                              [blocks_count_inclusive] => 4
                              [calls] => 1
                              [called_functions] => Array
                                (
                                  [str_repeat] => Array
                                    (
                                      [memory_size] => 0
                                      [blocks_count] => 0
                                      [memory_size_inclusive] => 0
                                      [blocks_count_inclusive] => 0
                                      [calls] => 1
                                      [called_functions] => Array
                                        (
                                        )
                                    )
                                )
                            )
                        )
                    )
                  [memprof_dump_array] => Array
                    (
                      [memory_size] => 0
                      [blocks_count] => 0
                      [memory_size_inclusive] => 0
                      [blocks_count_inclusive] => 0
                      [calls] => 1
                      [called_functions] => Array
                        (
                        )
                    )
                )
            )
        )
    )

## Todo

 * Support for tracking persistent (non-zend-alloc) allocations when libc
   doesn't have malloc hooks
 * Make it work without disabling the Zend memory allocator; or implement
   memory_limit

[1]: https://www.gnu.org/software/libc/manual/html_node/Hooks-for-Malloc.html#Hooks-for-Malloc
[2]: http://kcachegrind.sourceforge.net/html/Home.html
[3]: http://judy.sourceforge.net/index.html
[4]: https://google-perftools.googlecode.com/svn/trunk/doc/heapprofile.html
[5]: https://google-perftools.googlecode.com/

