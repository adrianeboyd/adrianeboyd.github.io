---
layout: post
title: Using valgrind with cython
tags: [cython,spacy,valgrind]
categories: cython
---

## Identifying memory leaks in cython with valgrind

How to use valgrind to track down memory leaks in cython. This example walks
through the process for a bug in [spaCy](https://github.com/spaCy) reported in
[issue #3618](https://github.com/explosion/spaCy/issues/3618) and fixed in
[PR #4486](https://github.com/explosion/spaCy/pull/4486).

1. Create a minimal script `minimal.py` that runs the code where you suspect a
   memory leak:

```python
import spacy
nlp = spacy.load('en')
doc = nlp("This is a sentence.")
```

2. 2. Download the valgrind suppressions file from CPython and uncomment the
lines related to `PyObject_Free` and `PyObject_Realloc` as instructed in the
header:
[valgrind-python.supp](https://github.com/python/cpython/blob/master/Misc/valgrind-python.supp)

3. Run valgrind with `--leak-check=full` to get detailed logs about where the
memory related to the leaks is allocated:

```bash
valgrind --tool=memcheck --leak-check=full \
--suppressions=valgrind-python.supp --log-file=minimal.valgrind.log \
python minimal.py
```

(Side note: setting `PYTHONMALLOC=malloc` (for python3.6+) lets valgrind
provide a more detailed analysis of python’s memory allocation, but I didn’t
need it to find this kind of cython-specific memory leak.)

Inspect the saved log file. The end of the file provides a summary:

```bash
==10207== LEAK SUMMARY:
==10207==    definitely lost: 3,936 bytes in 16 blocks
==10207==    indirectly lost: 0 bytes in 0 blocks
==10207==      possibly lost: 149,361 bytes in 94 blocks
==10207==    still reachable: 2,667,208 bytes in 1,535 blocks
==10207==         suppressed: 32 bytes in 1 blocks
```

The `definitely lost` bytes indicate memory leaks. If a memory leak is small
and happens once on initialization, it may not be a major problem. If you add a
loop to the minimal python script and notice that that amount of memory lost is
increasing as you increase the number of iterations, then you clearly have a
problematic memory leak.

Modify `minimal.py` so that the minimal example is executed 10 times:

```python
import spacy
nlp = spacy.load('en')
for i in range(10):
    doc = nlp("This is a sentence.")
```

When `doc = nlp("This is a sentence.")` is executed 10 times, the summary looks
like this:

```bash
==29544== LEAK SUMMARY:
==29544==    definitely lost: 31,000 bytes in 105 blocks
==29544==    indirectly lost: 0 bytes in 0 blocks
==29544==      possibly lost: 148,289 bytes in 92 blocks
==29544==    still reachable: 2,667,504 bytes in 1,536 blocks
==29544==         suppressed: 32 bytes in 1 blocks
```

Search for `definitely lost` in the log file to find more information about
where the allocations for the memory leaks occurred, e.g.:

```bash
==10207== 1,024 bytes in 2 blocks are definitely lost in loss record 667 of 878
==10207==    at 0x4837B65: calloc (vg_replace_malloc.c:752)
==10207==    by 0x20641C1A: __pyx_f_5spacy_6syntax_13_parser_model_resize_activations(__pyx_t_5spacy_6syntax_13_parser_model_ActivationsC*, __pyx_t_5spacy_6syntax_13_parser_model_SizesC) (_parser_model.cpp:6096)
==10207==    by 0x206450F7: __pyx_f_5spacy_6syntax_13_parser_model_predict_states(__pyx_t_5spacy_6syntax_13_parser_model_ActivationsC*, __pyx_t_5spacy_6syntax_6_state_StateC**, __pyx_t_5spacy_6syntax_13_parser_model_WeightsC const*, __pyx_t_5spacy_6syntax_13_parser_model_SizesC) (_parser_model.cpp:6254)
```

The third line indicates that the leaking memory was allocated on line 6096 of `_parser_model.cpp`:


```bash
/* "spacy/syntax/_parser_model.pyx":72
*         A.token_ids = <int*>calloc(n.states * n.feats, sizeof(A.token_ids[0]))
*         A.scores = <float*>calloc(n.states * n.classes, sizeof(A.scores[0]))
*         A.unmaxed = <float*>calloc(n.states * n.hiddens * n.pieces, sizeof(A.unmaxed[0]))             # <<<<<<<<<<<<<<
*         A.hiddens = <float*>calloc(n.states * n.hiddens, sizeof(A.hiddens[0]))
*         A.is_valid = <int*>calloc(n.states * n.classes, sizeof(A.is_valid[0]))
*/
__pyx_v_A->unmaxed = ((float *)calloc(((__pyx_v_n.states * __pyx_v_n.hiddens) * __pyx_v_n.pieces), (sizeof((__pyx_v_A->unmaxed[0])))));
```

Line 72 of `_parser_model.pyx` is where the memory was allocated in cython:

```python
if A._max_size == 0:
    A.token_ids = <int*>calloc(n.states * n.feats, sizeof(A.token_ids[0]))
    A.scores = <float*>calloc(n.states * n.classes, sizeof(A.scores[0]))
    A.unmaxed = <float*>calloc(n.states * n.hiddens * n.pieces, sizeof(A.unmaxed[0]))
    A.hiddens = <float*>calloc(n.states * n.hiddens, sizeof(A.hiddens[0]))
    A.is_valid = <int*>calloc(n.states * n.classes, sizeof(A.is_valid[0]))
    A._max_size = n.states
```

Searching the code shows that there’s no `free()` associated with these
`calloc()` calls, identifying the cause of the memory leak.

In this case, restructuring the code with utility functions that allocate
and free the memory when these structs are used in `nn_parser.pyx` fixes
the problem:

[https://github.com/explosion/spaCy/commit/3dfc76457709818fd3675b727d34e056aa6d434c](https://github.com/explosion/spaCy/commit/3dfc76457709818fd3675b727d34e056aa6d434c)
