This walkthrough won't cover _why_ Redis uses an event loop or what event loops are. For that
we'd recommend looking at these links:

- ["Anatomy of a Redis Command"][video-anatomy-of-a-redis-command] by Madelyn Olson at Redis Days Seattle
- ["What the heck is the event loop anyway?"][video-philip-roberts] by Philip Roberts at JSConf EU
- ["In the Loop"][video-jake-archibald] by Jake Archibald at JSConf.Asia

[video-anatomy-of-a-redis-command]: https://youtu.be/rgE7tZ1yH80?t=129
[video-philip-roberts]: https://www.youtube.com/watch?v=8aGhZQkoFbQ
[video-jake-archibald]: https://www.youtube.com/watch?v=cCOL7MC4Pl0

## What is the Redis event loop responsible for?

The primary functions of Redis's event loop are to: 

1. Accept new client connections
2. Respond to commands from existing clients

## Where is it initialized? 

The Redis event loop is created inside [`initServer`][function-initServer] as part of the Redis boot process. 

[function-initServer]: https://google.com

^^ referenced_code
link:https://github.com/redis/redis/blob/8203461120bf244e5c0576222c6aa5d986587bca/src/server.c#L2459
highlighted_lines:4
```c
void initServer(void) {
    // ... 
    
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
    if (server.el == NULL) {
        serverLog(LL_WARNING,
            "Failed creating the event loop. Error message: '%s'",
            strerror(errno));
        exit(1);
    }
    
    // ...
}
```

[`aeCreateEventLoop`][function-aeCreateEventLoop] is a function defined in [`ae.c`][], an event loop library bundled with 
Redis. This file hasn't changed much since the inception of Redis in 2009, it was included in the very [first commit][first-commit].

[`aeCreateEventLoop`][function-aeCreateEventLoop] initializes and returns the [`aeEventLoop`][symbol-aeEventLoop] data structure.

[file-ae-c]: https://google.com
[function-aeCreateEventLoop]: https://google.com
[symbol-aeEventLoop]: https://google.com
[first-commit]: https://google.com

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:2
```c
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;
```

We won't cover all these fields in this walkthrough, but we'll look at two important ones: `events` & `fired`.

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:6,7
```c
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;
```

Keep these in mind, we'll get back to them in the upcoming sections. 

Note that "creating an event loop"

# How are the events registered?
