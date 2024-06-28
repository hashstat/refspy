# README 

![RefSpy Logo](https://github.com/eukras/refspy/raw/master/refspy-logo.svg)


[![python](https://img.shields.io/badge/Python-3.11-3776AB.svg?style=flat&logo=python&logoColor=white)](https://www.python.org)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
![Version: 0.9 BETA](https://img.shields.io/badge/Version-0.9_BETA-red)




## Introduction

Refspy is a Python package for working with biblical references in ordinary text.

* [eukras/refspy on Github](https://github.com/eukras/refspy)
* [refspy on PyPI](https://pypi.org/project/refspy/)  --> `pip install refspy`

## Demo

![RefSpy Demo](https://github.com/eukras/refspy/raw/master/refspy-demo.png)

(See `refspy/docs`)

## Features

* Find biblical references in strings, using normal shorthands
* Construct and manipulate verses, ranges, and references
* Format references as names, abbreviated names, and URL parameters
* Store verses as `UNSIGNED INT(12)` for database indexing
* Compare and sort verses, ranges, and references (`<`, `==`, `>=`)
* Test if a range contains, overlaps, or adjoins another range
* Collate references by library and book for iteration
* Sequentially replace matched references in strings, e.g. with HTML links
* In English, we follow SBL conventions, including matching 'First Corinthians'
  for '1 Corinthians'.


## The Reference Manager


Initialising `refspy` with corpus and language names will return a reference
manager. This provides a single convenient interface for the whole library. 
By default, Refspy provides a Protestant canon in English.

```
from refspy.helper import refspy
__ = refspy()
```

Or, to create specific canons:

```
from refspy.language.en import ENGLISH
from refspy.libraries.en_US import DC, DC_ORTHODOX, NT, OT
from refspy.manager import Manager 

# Protestant
__ = refspy('protestant', 'en_US')
__ = Manager(libraries=[OT, NT], ENGLISH)

# Catholic
__ = refspy('catholic', 'en_US')
__ = Manager(libraries=[OT, DC, NT], ENGLISH)

# Orthodox
__ = refspy('orthodox', 'en_US')
__ = Manager(libraries=[OT, DC, DC_ORTHODOX, NT], ENGLISH)
```

The file `refspy/corpus.py` shows valid corpus names, and `refspy/language.py`
contains valid language names. (Only English at present.)

The `en_US` libraries conform to the SBL Style Guide for book names and
abbreviations. Other libraries can be defined in the libraries directory.


### Creating references 

Shortcut functions can create simple references using any book name,
abbreviation, code, or alias in the libraries list. Firstly, we can create
references from strings:

```
ref = __.r('Rom 2:6,9,1,2')
```

We can construct references more programmatically with `__.bcv()`:

```
ref = ___.bcv('Rom')                # Rom (whole book)  
ref = ___.bcv('Rom', 2)             # Rom 2 (whole chapter)
ref = ___.bcv('Rom', 2, 2)          # Rom 2:2
ref = ___.bcv('Rom', 2, 2, 3)       # Rom 2:2-3 (optional end verse)
```

Or `__.bcr()` to specify book, chapter and verse ranges:

```
ref = ___.bcr('Rom', 2, [(2, 3), 7])    # Rom 2:2-3,7 (from range) 
```


### Formatting references

```
ref = ___.r('Rom 2:3-4, 7')

__.name(ref)           # 'Romans 2:3-4,7'
__.abbrev(ref)         # 'Rom 2:3-4,7'
__.code(ref)           # 'rom+2.3-4,7'
__.numbers(ref)        # '2:3-4,7'
```


### Comparing references

A reference can be a set of any verses and verse ranges spread across multiple
libraries. Comparing references A and B just means comparing their first verse. 

```
rom_2 = __.r('Rom 2')
rom_4 = __.r('Rom 4')
rom_4a = __.bc('rom', 4)

assert rom_2 < rom_4
assert not rom_2 >= rom_4
assert rom_4 == rom_4a
```

Because references can be compared with `<`, they can also be sorted, or used
in `min()` and `max()`. 

```
assert sorted([rom_4, rom_2]) == [rom_2, rom_4]
assert min([rom_4, rom_2]) == rom_2
```

### Contains, Overlaps, Adjoins

We will commonly want to know if one reference `contains()`, or `overlaps()`
another. The `adjoins()` function works out adjacency for chapters and verses,
but note it is limited by not knowing the lengths of chapters. (Adjacency may
be used in future to combine and simplify references.)

```
gen1 = __.r('Gen 1') 
gen2 = __.r('Gen 2') 
gen1_23_23 = __.r('Gen 1:22-23') 
gen1_24_28 = __.r('Gen 1:24-28')
  
gen = __.b('Gen')
ex = __.b('Ex')

assert gen1.contains(gen1_22_23)
assert gen1_22_23.overlaps(gen1)
assert gen1_22_23.adjoins(gen1_24_28)
assert gen1.adjoins(gen2)
assert not gen1.overlaps(gen2)
```

## Manipulating references

Among other transformations, references can be turned into their (first) book
objects or references to just their books or chapters.

```
ref1 = __.ref('Rom 2:3-4,7')

assert __.book(ref1).chapters == 16
assert __.name(ref1.book_reference()) == 'Romans'
assert __.name(ref1.chapter_reference(ref1)) == 'Romans 2'
```

### Navigating references

```
assert next_chapter(rom_2) == rom_3
assert prev_chapter(rom_2) == rom_1
assert prev_chapter(rom_1) == acts_28
assert prev_chapter(matt_1) == None
```

To create chapter references:

```
from refspy.libraries.en_US import NT

nt_chapter_refs_ = [  
  __.bcv(book.name, ch)   
  for ch in range(1, book.chapters)   
  for book in NT.books  
] 
```


### Matching references in text

Here's how we might find references in text and print HTML links for them:

```
url = 'https://www.biblegateway.com/passage/?search=%s&version=NRSVA"

text = "Rom 1; 1 Cor 8:3,4; Rev 22:3-4"
   
strs, refs = __.find_references(text)
for match_str, ref in zip(strs, refs):
   print(f"{match_str} -> {url % __.code(ref)}")
```

### Replacing references in text

To produce the demo image above, we use the `sequential_replace` function from `refspy/utils`:

```
from refspy.utils import sequential_replace

matches = __.find_references(text, include_books=True)
strs, tags = [], []
for match_str, ref in matches:
    strs.append(match_str)
    if ref.is_book():
        tags.append(f'<span class="yellow">{match_str}</span>')
    else:
        tags.append(
            f'<span class="green">{match_str}</span><sup>{__.abbrev(ref)}</sup>'
        )
html = sequential_replace(text, strs, tags)}
```

### Collating and Indexing

To produce the index for the demo image above, we used:

```
matches = __.find_references(text, include_books=True)

index = []
for library, book_collation in __.collate(
    sorted([ref for _, ref in matches if not ref.is_book()])
):
    for book, reference_list in book_collation:
        new_reference = __.merge(reference_list)
        index.append(__.abbrev(new_reference))

html_list = "; ".join(index)
```
