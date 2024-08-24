# DSAC

DSAC is a collection of header only libraries that aims to provide powerful implementations of different Data Structures and Algorithmns in C

This code also targets kernel environments, with most of the data structures taking in allocators in their [make macros](#make-macros) parameters. It also returns booleans to indicate whether a certain operation was able to allocate or not making it useful in places where memory is limited (like in a kernel).

**IMPORTANT:**
Please also checkout [common patterns](#common-patterns)

# Examples
## Hashmaps
### Single file
```c
#include <stdio.h>      // printf
#include <stdlib.h>     // malloc+free
#include <string.h>     // strcmp

typedef struct {
    int age;
} Person;
#define HASHMAP_ALLOC(n) malloc(n)
#define HASHMAP_FREE(ptr, n) free(ptr)
#define streq(a, b) (strcmp(a,b)==0)
// The example uses djb2, but you can use any hash function
uintptr_t djb2(const char *str) {
    uintptr_t hash = 5381;
    char c;
    while ((c = *str)) {
        hash = ((hash << 5) + hash) + c;
        str++;
    }
    return hash;
}

#define HASHMAP_DEFINE
#include "hashmap.h"

MAKE_HASHMAP_EX(PersonMap, person_map, Person, const char*, djb2, streq, HASHMAP_ALLOC, HASHMAP_FREE) 
#undef HASHMAP_DEFINE // Just in case :)


// NOTE: Example does not check for allocation failure.
// You can just wrap *_insert in assert to make it panic
int main(void) {
    PersonMap map = {0};
    person_map_insert(&map, "John", (Person){.age=31}));
    person_map_insert(&map, "Bob" , (Person){.age=22}));

    Person* person;
    const char* name;
    name = "John";
    if((person=person_map_get(&map, name))) {
        printf("Found %s, age: %d\n", name, person->age);
    }

    name = "Bob";
    if((person=person_map_get(&map, name))) {
        printf("Found %s, age: %d\n", name, person->age);
    }

    name = "Dan";
    if((person=person_map_get(&map, name))) {
        printf("Found %s, age: %d\n", name, person->age);
    }

    person_map_destruct(&map);
    return 0;
}
```
Result:
```
Found John, age: 31
Found Bob, age: 22
```


### Multiple Files
person.h:
```c
#pragma once
// Declare/Define is decided by the source file
#include "hashmap.h"

#include <stdlib.h>     // malloc+free
#include <string.h>     // strcmp

typedef struct {
    int age;
} Person;
#define HASHMAP_ALLOC(n) malloc(n)
#define HASHMAP_FREE(ptr, n) free(ptr)
#define streq(a, b) (strcmp(a,b)==0)
uintptr_t djb2(const char *str);

MAKE_HASHMAP_EX(PersonMap, person_map, Person, const char*, djb2, streq, HASHMAP_ALLOC, HASHMAP_FREE) 
```
person.c:
```c
#include <stdint.h>
#include <stdbool.h>
#include <stddef.h>
uintptr_t djb2(const char *str) {
    uintptr_t hash = 5381;
    char c;
    while ((c = *str)) {
        hash = ((hash << 5) + hash) + c;
        str++;
    }
    return hash;
}
#define HASHMAP_DEFINE
#include "person.h"
```
main.c:
```c
#include <stdio.h>    // printf
#include "person.h"

int main(void) {
    PersonMap map = {0};
    person_map_insert(&map, "John", (Person){.age=31}));
    person_map_insert(&map, "Bob" , (Person){.age=22}));

    Person* person;
    const char* name;
    name = "John";
    if((person=person_map_get(&map, name))) {
        printf("Found %s, age: %d\n", name, person->age);
    }

    name = "Bob";
    if((person=person_map_get(&map, name))) {
        printf("Found %s, age: %d\n", name, person->age);
    }

    name = "Dan";
    if((person=person_map_get(&map, name))) {
        printf("Found %s, age: %d\n", name, person->age);
    }

    person_map_destruct(&map);
    return 0;
}
```
Result:
```
Found John, age: 31
Found Bob, age: 22
```

# Ideas
## Make Macros
Make macros are slightly different from DECLARE/DEFINE macros you'd usually see in a standard DSA library for C. The idea behind them is that you don't have to have the same parameteres in two places, so changing something is very easy and you shouldn't have to worry about updating it in two places.

The main idea of this project lies in the concept of a `make` macro. A make macro is a macro that will expand to the declaration of a type if no \<data structure\>\_DEFINE macro is present when the macro was included. If the \<data structure\>\_DEFINE macro is present when the `make` macro was included the `make` macro will expand to the definition of a data structure and its methods.

This is necessary because defining a generic hashmap implementation otherwise would either require the object to always be a pointer (or integer) and have the hashmap use function pointers to do things at runtime, and also have some sort of magic macros which don't really work well when you're trying to also notify in case an allocation error occurs. With `make` macros you add compile time information to a type which makes it way easier to define such functions.

The general layout of the arguments of a `make` macro are:
```
<Type name>, <'method' prefix>, (Types...), (Functions (key_eq, hash_key)...), (Allocators (ALLOC, DEALLOC)...)
```

## Common Patterns
There is one case where this library would fall short, and thats if you had one hashmap file include another file with another hashmap. To solve this there are declaration guards:
```c
/*Taken from the original example for hashmaps*/
#ifdef PERSON_DEFINE
#define HASHMAP_DEFINE
#endif
#include "hashmap.h"
MAKE_HASHMAP_EX(PersonMap, person_map, Person, const char*, djb2, streq, HASHMAP_ALLOC, HASHMAP_FREE) 

#ifdef PERSON_DEFINE
#undef HASHMAP_DEFINE // Just in case :)
#endif
```
That way its explicit that you are in fact defining specifically the person hashmap and not any other one.
