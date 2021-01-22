# Basic timer API's in Linux Kernel

### Key to note
- Kernel timers are run as the result of a software interrupt
- Any function running in interrupt context should be atomic

### struct timer_list
```c
    #include <linux/timer.h>
    
    struct timer_list {
        /*
         * All fields that change during normal runtime grouped to the
         * same cacheline
         */
        struct list_head entry;
        unsigned long expires;
        struct tvec_base *base;
    
        void (*function)(unsigned long);
        unsigned long data;
    
        int slack;
    
    #ifdef CONFIG_TIMER_STATS
        int start_pid;
        void *start_site;
        char start_comm[16];
    #endif
    #ifdef CONFIG_LOCKDEP
        struct lockdep_map lockdep_map;
    #endif
    };
```

### Timer API's
- init_timer() function can be used to initialize the timer.
``` c
    void init_timer(struct timer_list *timer);
```
- Add the timer to kernel timer list
``` c
    int add_timer(struct timer_list *timer);
```
- Add the timer from kernel timer list
``` c
    void del_timer(struct timer_list *timer);
```
- Instead of initializing the timer manually by calling init_timer, you can use the function to set data and function of timer_list structure and initialize the timer.
``` c
    void timer_setup(struct timer_list *timer, void(*callback)(struct timer_list *), unsigned int flags);
```
- del_timer_sync() is similar to the del_timer(). It also guarantees that when it returns, the timer function is not running on any CPU. It is used to avoid race conditions on SMP systems. Care when calling del_timer_sync() while holding locks. If the timer function attempts to obtain the same lock, the system can deadlock.
``` c
    int del_timer_sync(struct timer_list *timer, unsigned long expires);
```
- The mod_timer function is used to modify the timer timeout. This is more efficient way to update the xpire field of an active timer. Note if the timer is inactive it will be activated.
``` c
    int mod_timer(struct timer_list *timer, unsigned long expires);
```
- timer_pending can be used to check whether a timer is pending. It returns true if the timer is pending els returns false.
``` c
    int timer_pending(struct timer_list *timer);
```

### Timer callback function

Callback function will be executed from interrupt context. If you want to check that you can use function in_interrupt(), which takes no parameters and returns nonzero if the processor is currently running in interrupt context, either hardware or software interrupt. Since it is running in interrupt context, user cannot perform some actions inside the callback function:
- Go to sleep or relinquish the processor
- Acquire a mutex
- Perform timer-consuming tasks
- Access user space virtual memory

in_atomic() function can be used to check whether you are in atomic context