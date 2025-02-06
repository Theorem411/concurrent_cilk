# Errata

In order to successfully build this repo on a `ubuntu:16.04` based Docker image, I
've made the following changes to this codebase and its submodules. 

## Zero-cost Intrinsics: `__notify_intrinsic` and `__notify_zc_intrinsic`
I disabled uses of "zero-cost intrinsic", i.e. `__notify_intrinsic`, `__notify_zc_intrinsic` to avoid linker error. 

File changed:
+ `concurrent_cilk/runtime/rts-common.h`:
  ```c
  /* Compilers that build the Cilk runtime are assumed to know about
      zero-cost intrinsics.  For those that don't, comment out the
      following definition: */
  -#define ENABLE_NOTIFY_ZC_INTRINSIC
  +// #define ENABLE_NOTIFY_ZC_INTRINSIC
  ```

"Compilers that build the cilk runtime ..." in this case is the Intel Cilkplus branch of [clang](https://github.com/cilkplus/llvm). 

## `CCILK_TOTAL_PAUSE_EVENTS` stats dump segfaulted
Due to unknown reason (under investigation), statistic dump (`__cilkrts_dump_stats_to_stderr`) encounters segmentation fault when printing `CCILK_TOTAL_PAUSE_EVENTS` using `__cilkrts_get_total_pause_count()`. I disabled it for now but we might want to fix this. 

File changed: 
+ `concurrent_cilk/runtime/scheduler.c`:
  ```sh
  --- a/runtime/scheduler.c
  +++ b/runtime/scheduler.c
  @@ -152,7 +152,9 @@ void __cilkrts_dump_stats_to_stderr(global_state_t *g)
      fprintf(stderr, "CILKPLUS_TOTALSTACKS: %ld\n", g->stacks);
  #ifdef CILK_IVARS
      fprintf(stderr, "CONCURRENTCILK_WORKERS_BLOCKED: %d\n", g->workers_blocked);
  +  #if 0 /** DEBUG: UNDO ME! Unknown Segfault in __cilkrts_get_total_pause_count, debug later */
      fprintf(stderr, "CCILK_TOTAL_PAUSE_EVENTS: %llu\n", __cilkrts_get_total_pause_count());
  +  #endif
      fprintf(stderr, "CCILK_EXTRA_STACKS_ADDED: %lu\n", g->total_extra_stacks);
  #endif
  #ifdef CILK_PROFILE
  ```

## External Library Dependencies: `libevent` 
The `deps/libevent` that comes with the original [concurrent_cilk](https://github.com/iu-parfunc/concurrent_cilk.git) repo cannot build `libevent_pthreads.so`. 

I fixed this by downloading [`libevent-2.1.12-stable`](https://libevent.org/) and built the library from source.

**TODO:** this version of `libevent` might be too recent. I'll try reverting to `libevent-2.0.22-stable` released in 2014. 

## Trivia
### `cilk_tests` submodule cmake build
The original did not add `-L${CONCURRENT_CILK_INSTALL_LIB_DIR}` to clang compile options, so `-lcilkrts` couldn't find `libcilkrts.so`. I manually added them to `CMAKE_C_FLAGS` and `CMAKE_CXX_FLAGS`. Below are `git diff` output: 
```sh
$ git diff --cached -- cmake/Modules/CilkTestsCommon.cmake
@@ -74,6 +74,8 @@ set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${BASE_C_FLAGS} ${EXTRA_RELE
 set(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} ${BASE_C_FLAGS} ${EXTRA_RELEASE_C_FLAGS}")
 set(CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL} ${BASE_C_FLAGS}")
 set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${BASE_C_FLAGS} ${EXTRA_DEBUG_FLAGS}")
+# added 
+set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -L/workspace/concurrent_cilk/install/lib")
 
 #C++ Additions
 set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${BASE_CXX_FLAGS}")
@@ -81,3 +83,5 @@ set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${BASE_CXX_FLAGS} ${EXTR
 set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} ${BASE_CXX_FLAGS} ${EXTRA_RELEASE_CXX_FLAGS}")
 set(CMAKE_CXX_FLAGS_MINSIZEREL "${CMAKE_CXX_FLAGS_MINSIZEREL} ${BASE_CXX_FLAGS}")
 set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${BASE_CXX_FLAGS} ${EXTRA_DEBUG_FLAGS}")
+# added 
+set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -L/workspace/concurrent_cilk/install/lib")
\ No newline at end of file
```

# Unresolved Issues

Currently `ivars_parfib.exe` (using channels) has frequent assertion error when Fibonacci input is larger than 20. E.g.: 
```bash
$ CILK_NWORKERS=2 ./ivars_parfib.exe 20
/workspace/concurrent_cilk/runtime/ivar_full_blocking.c:124: cilk assertion failed: 0
```
`concurrent_cilk/runtime/ivar_full_blocking.c:124` is from their "read-from-channel" API, and the assertion failed because scheduler context-switch didn't happen (see `<FAILED HERE>`): 
<details>
<summary> ivar_full_blocking.c:124 </summary>
  
  ```c
  inline CILK_API(ivar_payload_t)
  __cilkrts_ivar_read(__cilkrts_ivar *ivar)
  {
    __cilkrts_worker *w, *replacement;
    unsigned short exit = 0;
    uintptr_t val;
    jmp_buf ctx; 
    uintptr_t volatile peek;
    ivar_payload_t my_payload;

    cons_t my_waitlist_cell, *cur_cell;

    CILK_ASSERT(ivar);

    //fast path -- already got a value.
    //----------------------------------
    if (IVAR_READY(*ivar)) {
      dbgprint(IVAR, "ivar %p FULL -fast path\n", ivar);
      return UNTAG(*ivar);
    }

    //slow path -- operation must block until a value is available.
    //----------------------------------
    val = (uintptr_t) __cilkrts_pause_fiber(&ctx);

    if (! val) {
      w = __cilkrts_get_tls_worker_fast();
      my_waitlist_cell.car = w;
      my_payload = (((ivar_payload_t) &my_waitlist_cell) << IVAR_SHIFT) | CILK_IVAR_PAUSED;
      replacement = __cilkrts_commit_pause(w, &ctx); 

      do {
        peek = *ivar;
        switch (peek & IVAR_MASK) {
          case CILK_IVAR_EMPTY: 
            my_waitlist_cell.cdr = NULL;          
            exit = cas(ivar, 0, my_payload);
            if (! exit) { 
              dbgprint(IVAR, "ivar %p failed cas on EMPTY ivar - going around again\n", ivar);
            } else {
              dbgprint(IVAR, "ivar %p EMPTY. Filled with replacement %p\n", ivar, replacement);
            }
            break;
          case CILK_IVAR_PAUSED:
            cur_cell = (cons_t *)(peek >> IVAR_SHIFT);
            CILK_ASSERT(cur_cell); // Never empty because it starts as a singleton
            my_waitlist_cell.cdr = cur_cell; // If we bump them with CAS, then they are our neighor.
            exit = cas(ivar, peek, my_payload);
            if (exit) dbgprint(IVAR,"ivar %p PAUSED. added worker %p to waitlist %p\n",ivar,w,
                              my_waitlist_cell.cdr);
            break;
          case CILK_IVAR_FULL:
            dbgprint(IVAR, "ivar %p FILLED while reading\n", ivar);
            //nevermind...someone filled it. 
            __cilkrts_roll_back_pause(w, replacement);
            return UNTAG(*ivar);
            break; //go around again
          default: 
            __cilkrts_bug("[read] Cilk IVar %p in corrupted state 0x%x. Aborting program.\n", ivar, *ivar&IVAR_MASK);
        }
      } while (!exit);

      //thread local array operation, no lock needed
      __cilkrts_register_paused_worker_for_stealing(w);
      __cilkrts_run_replacement_fiber(replacement);
      CILK_ASSERT(0); //no return. heads to scheduler. <FAILED HERE>
    }

    return UNTAG(*ivar);
  }
  ```
</details>