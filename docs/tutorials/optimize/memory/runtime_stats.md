### Runtime memory statistics

Arm Mbed OS 5 provides various runtime statistics to help characterize resource usage. This allows easy identification of potential problems, such as a stack close to overflowing. The metrics currently supported are available for the [heap](#heap-statistics) and the [stack](#stack-statistics).

#### Heap statistics

Heap statistics provide exact information about the number of bytes dynamically allocated by a program. It does not take into account heap fragmentation or allocation overhead. This allows allocation size reports to remain consistent, regardless of order of allocation (fragmentation) or allocation algorithm (overhead).

##### To enable heap stats from Mbed CLI:

1. Add the command-line flag `-DMBED_HEAP_STATS_ENABLED=1`, for example `mbed compile -DMBED_HEAP_STATS_ENABLED=1`
1. Use the function `mbed_stats_heap_get()` to take a snapshot of heap stats.

##### To enable heap stats using `mbed_app.json`:

1. Add the following to `mbed_app.json`:
   
   ```
   {
       "macros": [
           "MBED_HEAP_STATS_ENABLED=1"
       ],
       ...
   }
   ```

1. Use the function `mbed_stats_heap_get()` to take a snapshot of heap stats.

   <span class="notes">**Note**: This function is available even when the heap stats are not enabled but always returns zero for all fields.</span>

   `mbed_stats_heap_get()` returns a struct containing the following:

   | Variable          | Description                                       |
   | :---------------- | :------------------------------------------------ |
   | current_size      | Bytes allocated currently.                        |
   | max_size          | Max bytes allocated at a given time.              |
   | total_size        | Cumulative sum of bytes ever allocated.           |
   | reserved_size     | Current number of bytes allocated for the heap.   |
   | alloc_cnt         | Current number of allocations.                    |
   | alloc_fail_cnt    | Number of failed allocations.                     |


#### Example heap statistics use cases

- Getting worst case memory usage, `max_size`, to properly size MCU RAM.
- Detecting program memory leaks by the current size allocated (`current_size`) or number of allocations in use (`alloc_cnt`).
- Use `alloc_fail_cnt` to check if allocations have been failing, and if so, how many.

##### Example program using heap statistics

```
#include "mbed.h"
#include "mbed_stats.h"

int main(void)
{
    mbed_stats_heap_t heap_stats;

    printf("Starting heap stats example\r\n");

    void *allocation = malloc(1000);
    printf("Freeing 1000 bytes\r\n");

    mbed_stats_heap_get(&heap_stats);
    printf("Current heap: %lu\r\n", heap_stats.current_size);
    printf("Max heap size: %lu\r\n", heap_stats.max_size);

    free(allocation);

    mbed_stats_heap_get(&heap_stats);
    printf("Current heap after: %lu\r\n", heap_stats.current_size);
    printf("Max heap size after: %lu\r\n", heap_stats.max_size);
}
```

##### Side effects of enabling heap statistics

- An additional 8 bytes of overhead for each memory allocation.
- The function `realloc` never reuses the buffer it is resizing.
- Memory allocation is slightly slower due to the added bookkeeping.

#### Stack statistics

Stack stats provide information on the allocated stack size of a thread and the worst case stack use. You can query any thread on the system for stack information.

To enable stack stats from the command-line, add the command-line flag `-DMBED_STACK_STATS_ENABLED=1`, for example `mbed compile -DMBED_HEAP_STATS_ENABLED=1`

Alternatively, to enable stack stats using `mbed_app.json`, add the following to `mbed_app.json`:

```
{
    "macros": [
        "MBED_STACK_STATS_ENABLED=1"
    ],
    ...
}
```

There are two functions you can use to access the stack stats:

- `mbed_stats_stack_get` calculates combined stack informations for all threads.
- `mbed_stats_stack_get_each` provides stack informations for each thread separately.

<span class="notes">**Note**: These functions are available even when the stack stats are not enabled but always return zero for all fields.</span>

Both of these functions return a struct containing the following:


| Variable        | Description                                                               |
| :-------------- | :------------------------------------------------------------------------ |
| thread_id       | Identifier for the thread that owns the stack or 0 if multiple threads.   |              
| max_size        | Maximum number of bytes used on the stack.                                |
| reserved_size   | Current number of bytes allocated for the stack.                          |
| stack_cnt       | Number of stacks stats accumulated in the structure.                      |


#### Example stack statistics use cases

- Using `max_size` to calibrate stack sizes for each thread.
- Detecting which stack is close to overflowing.

##### Example program using stack statistics

```
#include "mbed.h"
#include "mbed_stats.h"

int main(void)
{
    printf("Starting stack stats example\r\n");

    int cnt = osThreadGetCount();
    mbed_stats_stack_t *stats = (mbed_stats_stack_t*) malloc(cnt * sizeof(mbed_stats_stack_t));

    cnt = mbed_stats_stack_get_each(stats, cnt);
    for (int i = 0; i < cnt; i++) {
        printf("Thread: 0x%X, Stack size: %u, Max stack: %u\r\n", stats[i].thread_id, stats[i].reserved_size, stats[i].max_size);
    }
}
```
