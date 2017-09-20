# Project 1 - Threads

#### Part I – Efficient Alarm Clock
##### Data structures and functions
* __thread.h__
	* Add a new int64_t variable `sleep_ticks` for the thread struct that keeps track of the remaining number of ticks for the thread
* __thread.c__
	*  Add a new list struct variable `sleep_list` and initialize it in `thread_init`
	*  Write two comparator functions `decreasing_priority` and `increasing_ticks`
	*  Add a new function `thread_wakeup` to check if there are threads need to be woken up after each tick
	*  Add a new function `thread_sleep` called by `timer_sleep` in timer.c

##### Algorithm
*	__timer_sleep__ in timer.c
	*    Make sure ticks passed in is positive, otherwise return immediately
	*    Call `thread_sleep` with ticks as the argument
*	__thread_sleep__ in thread.c
	*   Get current time ticks using `timer_ticks`, and assign `sleep_ticks` the value of timer_ticks() + ticks
	*   Put thread into sleep queue by blocking the thread, and turn interrupts off
	*   The threads are inserted into the sleep queue using `list_insert_ordered` (list.h) and the comparator `increasing_ticks`, which arranges the list such that the first element has the smallest sleep_ticks value
*	__thread_tick__ in thread.c
	*  Function now calls `thread_wakeup` after each tick
*	__thread_wakeup__ in thread.c
	*   Iterate through entire `sleep_list` by calling `list_next` (built-in function of list.h)
	*   Wake up threads whose `sleep_ticks` is smaller or equal to current `timer_ticks`
	*   “Wake up” means transferring the thread from the sleep queue to the ready queue and set its status from blocked to ready. This is done by calling `thread_unblock`
*	__thread_unblock__ in thread.c
	*  We now insert the thread elements in decreasing priority order, using the built-in function from list.h `list_insert_ordered`
*	__decreasing_priority__ in thread.c
	*  Comparator, takes in two struct list_elem arguments. Using the threads they represent respectively, we compare their priority and order them in decreasing fashion.
*	__increasing_ticks__ in thread.c
	*  Has the same structure as `decreasing_priority`, except that the `sleep_ticks` variable of each thread is compared instead, and ordered in increasing fashion.
*	__thread_yield__ in thread.c:
	*  Insert current thread into the ready list in decreasing priority also (using `list_insert_ordered` and `decreasing_priority`)

##### Synchronization
*	Race conditions could occur if multiple threads call `timer_sleep` simultaneously
*	The interrupt handler is disabled when `timer_sleep` is called, and the thread is subsequently blocked to ensure synchronization
*	Since the list manipulation functions are not thread-safe, interrupt is disabled whenever one of those functions are used

##### Rationale
*	Mirroring the implementation of the `ready_list`, having an additional `sleep_list` is very easy to implement and maintain
*	In order to minimize the time spent in interrupt handler, `sleep_list` always kept in sorted order using the comparator function `increasing_ticks`. This means that not every thread in `sleep_list` needs to be checked in each iteration
*	Adding an extra variable `sleep_ticks` within each thread makes it easy to track and order the threads in `sleep_list`. This way, no additional data structure is required

#### Part II - Priority Scheduler
##### Data structures and functions
*	__thread.c__
	*    Add int `original_priority` to the thread struct in order to preserve the thread’s priority before donation
	*    Modify int `priority` in the thread struct to store the current potentially donated priority
	*    Add bool `received_donation` to the thread struct in order to indicate whether the current thread’s priority come from a donation
	*    Add struct list `lock_list` to the thread struct. This is used in multiple donations to keep track of the current thread’s priority according to the locks it holds
	*    Add struct lock* `blocking_thread_lock`. This is used in nested donation to track the next lock, NULL if there is no lock
	*    Add list_elem `elem_lock` to struct lock so it can be included in `lock_list`
	*    Add int `lock_priority` to struct lock, used in multiple donations to keep track of the highest-priority semaphore waiter.
*	__thread.h__
    *    add macro definition `NO_DONATION = -1` as the initial value for `lock_priority`

##### Algorithm

__Acquiring a Lock (lock_acquire)__
*    Disable interrupts
*    If the lock is not held by a thread, add lock to ordered `lock_list` of current thread; otherwise, `locking_thread_lock` of holder thread = current lock
*    Compare holder priority to `curr_thread` priority. If lower, set `lock_priority` to `curr_thread` priority, set holder priority to `lock_priority`
*    `curr_lock` = `blocking_thread_lock`
*    While `blocking_thread_lock` is not NULL and < limit (to prevent deadlocks)
*	Compare holder priority to `curr_thread` priority:
	*	If lower, set `lock_priority` to `curr_thread` priority, set holder priority to `lock_priority`
		*	`curr_thread` = holder thread
		*	`curr_lock` = holder thread's `blocking_thread_lock`
* If donation happens, remove `curr_thread` and all threads receiving donations and re-insert in sorted order (done in `set_priority_helper`)
*   Set interrupts back to original status
##### __Releasing a Lock (lock_release)__
*   Disable interrupts
*   Remove lock from `curr_thread`’s `lock_list`
*   Change lock priority back to `NO_DONATION`
*   If `received_donation` is FALSE, set lock holder’s priority to `original_priority`
*   Otherwise, look at the `lock_list` (if it’s empty, set lock holder’s priority to `original_priority`)
*   If the `lock_list` is not empty, we look at the first lock of the `lock_list` and compare `lock_priority` to `original_priority`. If `lock_priority` is greater, set priority of curr_thread to be `lock_priority`
*   Update lock’s holder to NULL

__thread_set_priority (thread.c)__
*   Set priority using helper function `set_priority_helper(struct thread *t, int new_priority, bool received_donation)`

__set_priority_helper (thread.c)__
*   Disable interrupts
* 	If the operation is not a donation, i.e. `received_donation` is false, and the new priority is less than or equal to the donated priority, we preserve it in `original_priority`
*   Otherwise, set priority to the new priority, and mark the thread as a donated one
*   If the current thread is READY, reinsert it into the `ready_list` to keep the `ready_list` in order. If it is running, and the current thread’s priority is now smaller than the largest one in the `ready_list`, we yield the CPU

##### Synchronization
*	During priority donation, the lock holder’s priority may be set by its donor. At the mean time, the thread itself could change its own priority, which causes a race condition. To solve this, we disable interrupt in both cases to prevent it from happening.

##### Rationale
*	We added `received_donation`, as well as `original_priority` to the thread struct, even though we didn’t want to expand it too much. These two variables are necessary in determining whether a donation has occurred. 
*	For the struct lock, we also had to find a way to get the highest priority in its waiters. We could’ve avoided adding another variable to its struct, by using the `list_front` function; however, the code would have been more difficult to read and more complex. 

#### Part III - Advanced Scheduler
##### Data structures and functions
__thread.c__
* `thread_get_nice` (return niceness variable from current thread)
* `thread_set_nice` (set niceness variable for current thread)
* `thread_get_load_average` 
* `thread_get_recent_cpu`
* static int[] `priority_lists` (list of 64 priority lists, one for each list)

__thread.h__

* Add new struct variable `niceness` (initialize to 0 in thread_init)
* Add new struct variable `recent_cpu` (initialize to 0)
* Add new struct variable `load_average`
* Create 64 linked lists for each priority

__list.h__

* Add new struct list_elem variable `sublist_elem`

##### Algorithm

At each 4th call to tick we will update the `load average`, `recent_cpu`, and `priority` values. Then we will reorganize the threads in the list of queues. At each call to `schedule` we will pop off the first element of the highest priority queue in our list of priority queues. The round robin behavior at each tick will be maintained by pushing back each thread to the end of each queue, so that recently used threads will come last in the queue. The list of queues will be pre-initialized in static memory, and maintained by changing pointers
 
__thread_tick__ in thread.c

* Update priority whenever tick time is divisible by 4
* Update `recent_cpu` when tick time is divisible by `TIMER_FREQ`
* Update `load_avg`

__next_thread_to_run__ in thread.c

* Pop off thread from highest priority non-empty priority queue

__schedule__ 
* Push back the current thread by its (possibly) updated priority, or don’t if the thread is in a dying state or is the idle thread.
* Remove and push back threads whose priority has changed. (we need to keep track of this).
* Remove any empty queues so that at each call to `next_thread_to_run` all queues are non-empty. 

__Implement skeleton functions__: `thread_get_nice`, `thread_set_nice` , `thread_get_load_average`,`thread_get_recent_cpu`.

__Block thread_set_priority() / ignore thread_create() priority__

##### Synchronization
* Disable interrupts whenever using list functions, as these are not thread safe
* Use local variables for `priority`, `recent_cpu`, and `load_avg`
* Must acquire lock on relevant sublist when modifying

##### Rationale
We chose the our algorithm and data structures to minimize CPU time and avoid dynamic allocation. All queues are initialized statically in an array to avoid dynamic memory and to allow for constant access via priority value (i.e the i-th priority list in the array has priority i). The fact that there will be at most 64 queues means that the list of non-empty queues will be sortable in constant time. Furthermore, moving threads between queues is relatively simple and can be done in linear time with very little bookkeeping.  







