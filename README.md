# hash.h

A generic hash table written in C.

This implementation is used to build dictionaries where entries won't be removed. (i.e.: This implementation does not allow you to remove entries with the provided functions)

It's original use case did not require a remove function;
However, it would not be difficult to implement a version where you _could_ remove entries.

## History

This header was written originaly for _CMidi_, a software based MIDI synthesizer for a game called _Mya and their Cat_. It's purpose was two fold: 1) To process MIDI Marker events so that CMidi could stash song position information for later use; 2) To process MIDI Text events into native C function calls.
The use case didn't require the need to remove entries.

## Warning

To use this header you'll need to use pointers very heavily. This is because the code provides a sense of 'generic-ness' through the use of `void*`.

`HashTable_At` returns a `void*` but this `void*` is actually a pointer to a pointer. You can see this within the example at the bottom of this document where we cast the return value of `HashTable_At` to a pointer to a function pointer and then dereference it to get the function pointer itself.

You could write some macros to help with this issue. An example of one might be:

```C
#define AT(TYPE,TABLE,KEY) (*(TYPE*)HashTable_At(&TABLE,KEY))
struct HashTable intTable = {0};

...

AT(int, intTable, "forty-two") = 42;
int answer = AT(int, intTable, "forty-two");
```
You can see an example of a `registerFunc` macro used in the example at the bottom of this document.

## Structs

### HashPair

```C
struct HashPair {
    void* key;
    void* element;
};
```

#### Description

`struct HashPair` is a struct that contains both a pointer to the _key_ and to the _element_.

Both `HashTable_At` and `HashTable_Has` return a pointer to the `element` member of the `HashPair` struct. This means that you'll need to dereference those functions return values to access `element` for both getting and setting it's value.

You'll also need to cast `element` to your desired pointer type.

(e.g.: `*(int*)HashTable_At(&table, key) = 42;`)

The only time you'll need to interface with this struct is as a parameter to the `onRemove` function that you'll need to specify when calling `HashTable_Init`. See the callbacks section for more information.

---

### HashList

```C
struct HashList {
    struct HashList* next;
    struct HashPair* pair;
};
```

#### Description

A struct used internally. It's a linked list node that acts as the bin of the hash table.

You won't need to interface with this struct.

---

### HashTable
```C
struct HashTable {
    struct HashList** hashListArr;
    int (*cmp)(void* a, void* b);
    unsigned int (*hash)(void* key);
    int (*onRemove)(struct HashPair*);
};
```

#### Description

The hash table struct that you'll be passing into functions as to do various operations on the hash table.

See the callbacks section for more information on `cmp`, `hash`, and `onRemove`.

---

## Usage

### HashTable_Init

```C
struct HashTable*
HashTable_Init
(
    struct HashTable* hashTable,
    int (*cmp)(void* a, void* b),
    unsigned int (*hash)(void* key),
    int (*onRemove)(struct HashPair*)
);
```

#### Description

Initializes a `struct HashTable` for use.
This method must be called to use any other function.

#### Returns

Returns `hashTable` parameter.

#### Parameters

|     |     |
| --- | --- |
| `hashTable` | Pointer to a `struct HashTable` |
| `cmp` | A function to compare hash table elements |
| `hash` | A function to generate hashcodes for the hash table elements |
| `onRemove` | A function to call when an entry is removed from the hash table

---

### HashTable_Destroy

```C
void
HashTable_Destroy
(
    struct HashTable* hashTable
);
```

#### Description

Destroy a `struct HashTable`. This frees any memory the hash table manages as well as calls the `onRemove` function for each `struct HashPair` in the table. Specify the `onRemove` function when initializing the hash table with `HashTable_Init`.

#### Returns

(void)

#### Parameters

|     |     |
| --- | --- |
| `hashTable` | Pointer to an initialized `struct HashTable` |

---

### HashTable_Has

```C
void*
HashTable_Has
(
    struct HashTable* hashTable,
    void* key
);
```

#### Description

Check to see if a key is found within the hash table.
If the return value is not null then you can dereference it to _get_ or _set_ the `element` (which itself is a `void*` see `struct HashPair`).

#### Returns

If the key is found a pointer to `element` is returned (see `struct HashPair`).

If the key is not found a null pointer is returned (0).

#### Parameters

|     |     |
| --- | --- |
| `hashTable` | Pointer to an initialized `struct HashTable` |
| key | Pointer to the key

---

### HashTable_At

```C
void*
HashTable_At
(
    struct HashTable* hashTable,
    void* key
);
```

#### Description

Fetches an element from the table. If the key is not found then a new entry is added to the table and a pointer to the new `element` is returned.

You can dereference the return value to get or set the `element` (which itself is a `void*` see `struct HashPair`).

#### Returns

A pointer to an `element` (see `struct HashPair`).

#### Parameters

|     |     |
| --- | --- |
| `hashTable` | Pointer to an initialized `struct HashTable` |
| key | Pointer to the key

---

### HashTable_Ready

```C
int
HashTable_Ready
(
    struct HashTable* hashTable
);
```

#### Description

Checks to see if `hashTable` is initialized and ready.
A `hashTable` will be ready if it was initialized using `HashTable_Init`.

#### Returns

If the hash table is _not_ ready than 0 will be returned.
Otherwise the hash table is ready.

#### Parameters

|     |     |
| --- | --- |
| `hashTable` | Pointer to a `struct HashTable` |

---

## Callbacks

### `cmp`

```C
int
(*cmp)
(
    void* a,
    void* b
);
```

#### Details

Compares two hash table keys.

`cmp` should return 0 when both keys are equal.

---

### `hash`

```C
unsigned int
(*hash)
(
    void* key
);
```

#### Details

Generates a hashcode from a key.

The generated hashcode should be deterministic (i.e.: the same key should generate the same value).

---

### `onRemove`

```C
int
(*onRemove)
(
    struct HashPair* hashPair
);
```

#### Details

When we add entries to the hash table we are effectively giving up ownership of those objects. As such we need to provide a callback that specifies how the `key` and `element` objects are freed.

In the `onRemove` function we only need to handle how `hashPair->key` and `hashPair->element` are freed. If the `key` and `element` are raw values (not pointers to memory that needs to be freed) you can specify a null pointer (0).

---

## Example 1

```C
#include <stdio.h>
#include "hash.h"

#define registerFunc(KEY,VALUE) (*((funcPtr*)HashTable_At(&funcTable, KEY)) = VALUE)

typedef void (*funcPtr)();

struct HashTable funcTable = {0};

int cmp(void* m1, void* m2) {
    return strcmp(m1, m2);
}

unsigned int hash(void* key) {
    char* k = key;
    int hashval;
    for (hashval = 0; *k != '\0'; k++) {
        hashval = *k + 31 * hashval;
    }
    return hashval;
}

void hello() {
    puts("Hello, World!");
}

void bye() {
    puts("Good-bye!");
}

int main(int argc, char* argv[]) {
    //Setup table
    HashTable_Init(&funcTable,cmp,hash,0);
    registerFunc("Hello", hello);
    registerFunc("Bye", bye);

    //Perform actions on table.
    (*(funcPtr*)HashTable_At(&funcTable, "Hello"))();
    (*(funcPtr*)HashTable_At(&funcTable, "Bye"))();

    return 0;
}
```

##### Expected Output

```
Hello, World!
Good-bye!
```

## Example 2

```C
#include <stdio.h>
#include <stdlib.h>
#include "hash.h"

#define AT(TYPE,TABLE,KEY) (*(TYPE*)HashTable_At(&TABLE,KEY))

struct Element {
    const char* element_value;
};

struct Element* CreateElement(const char* value) {
    struct Element* e = malloc(sizeof(struct Element));
    e->element_value = value;
    return e;
}

void DestroyElement(struct Element* e) {
    free(e);
}

int cmp(void* m1, void* m2) {
    return m1 - m2;
}

unsigned int hash(void* key) {
    return (unsigned int)key;
}

void onRemove(struct HashPair* hashPair) {
    struct Element* e = hashPair->element;
    printf("Destroying Element: %s\n", e->element_value);
    DestroyElement(hashPair->element);
}

int main(int argc, char* argv[]) {
    //Setup table
    struct HashTable table;
    HashTable_Init(&table, cmp, hash, onRemove);

    //Add stuff to table
    AT(struct Element*, table, 42) = CreateElement("This is the answer.");
    AT(struct Element*, table, 35) = CreateElement("This is not the answer.");

    //Access the table.
    struct Element* e1 = AT(struct Element*, table, 42);
    struct Element* e2 = AT(struct Element*, table, 35);

    //Do stuff with accessed things.
    printf("Answer 42  : %s\n", e1->element_value);
    printf("Answer 35  : %s\n", e2->element_value);

    HashTable_Destroy(&table);

    return 0;
}
```

##### Expected Output

```
Answer 42  : This is the answer.
Answer 35  : This is not the answer.
Destroying Element: This is not the answer.
Destroying Element: This is the answer.
```
