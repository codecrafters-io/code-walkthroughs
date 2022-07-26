This walkthrough won't cover _why_ Redis uses an event loop or what event loops are. For that
we'd recommend looking at these links:

- [Anatomy of a Redis Command][video-anatomy-of-a-redis-command] by Madelyn Olson at Redis Days Seattle
- [What the heck is the event loop anyway?][video-philip-roberts] by Philip Roberts at JSConf EU
- [In the Loop][video-jake-archibald] by Jake Archibald at JSConf.Asia

## Introduction

The primary functions of Redis's event loop are to: 

1. Accept new client connections
2. Respond to commands from existing connections

We'll look at how the event loop data structure is initialized, how events are registered and how the event loop works
internally.

## Where is it initialized? 

The Redis event loop is created inside [`initServer`][symbol-initServer] as part of the Redis boot process. 

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

[`aeCreateEventLoop`][symbol-aeCreateEventLoop] is a function defined in [`ae.c`][file-ae-c]. This file hasn't changed 
much since the inception of Redis in 2009, it was included in the [initial commit][initial-commit].

[`aeCreateEventLoop`][symbol-aeCreateEventLoop] initializes and returns the [`aeEventLoop`][symbol-aeEventLoop] data structure.

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

## How are event handlers registered?

[`aeCreateEventLoop`][symbol-aeCreateEventLoop] only initializes the [`aeEventLoop`][symbol-aeEventLoop] data 
structure, we still need to specify what events we want the event loop to wait on.

This is done using the [`aeCreateFileEvent`][symbol-aeCreateFileEvent] function.

As mentioned in the introduction section, there are two primary events we want the event loop to notify us of: **(1)** when 
a new connection is available, and **(2)** when existing connections are ready to read from. 

The event handler for new connections is registered early in the boot process: 

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:7
```c
/* Create an event handler for accepting new connections in TCP or TLS domain sockets.
 * This works atomically for all socket fds */
int createSocketAcceptHandler(socketFds *sfd, aeFileProc *accept_handler) {
    int j;

    for (j = 0; j < sfd->count; j++) {
        if (aeCreateFileEvent(server.el, sfd->fd[j], AE_READABLE, accept_handler,NULL) == AE_ERR) {
            /* Rollback */
            for (j = j-1; j >= 0; j--) aeDeleteFileEvent(server.el, sfd->fd[j], AE_READABLE);
            return C_ERR;
        }
    }
    return C_OK;
}
```

The event handler for connections being ready to read can't be registered when the server boots since connections aren't 
established at that point. These events are set up after a connection is accepted, and they're removed when a connection 
is disconnected.

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:11
```c
/* Register a read handler, to be called when the connection is readable.
 * If NULL, the existing handler is removed.
 */
static int connSocketSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
    if (func == conn->read_handler) return C_OK;

    conn->read_handler = func;
    if (!conn->read_handler)
        aeDeleteFileEvent(server.el,conn->fd,AE_READABLE);
    else
        if (aeCreateFileEvent(server.el,conn->fd,
                    AE_READABLE,conn->type->ae_handler,conn) == AE_ERR) return C_ERR;
    return C_OK;
}
```

## How are event handlers triggered?

We've seen how the event loop is initialized and how event handlers are registered. In this section we'll take a closer 
look at how the event loop itself works. 

Redis's event loop implementation is self-contained and lives in [`ae.c`](file-ae-c).

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:1,2,3
```c
/* A simple event-driven programming library. Originally I wrote this code
 * for the Jim's event-loop (Jim is a Tcl interpreter) but later translated
 * it in form of a library for easy reuse.
 *
 * Copyright (c) 2006-2010, Salvatore Sanfilippo <antirez at gmail dot com>
 * All rights reserved.
```

**Fun Fact**: The name `ae.c` [is short for](https://twitter.com/antirez/status/1551184879416737793) "(a)ntirez (e)vents". 
You'll see this pattern in many other files too, like `adlist.c` ("(a)ntirez (d)ynamic (l)ists").

Underneath the hood, Redis uses one of the following system APIs to wait on events: 

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:1,2
```c
/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

When the event loop is started, it continuously loops and processes any events that are ready.

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:3,4,5,6,7
```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```

When events are ready, they're placed in `fired` field. 

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:7
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

The `aeProcessEvents` function reads events from this field:

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:11,19
```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
    // ... 
    
    /* Call the multiplexing API, will return only on timeout or when
     * some event fires. */
    numevents = aeApiPoll(eventLoop, tvp);
    
    // ...
   
    for (j = 0; j < numevents; j++) {
        int fd = eventLoop->fired[j].fd;
        aeFileEvent *fe = &eventLoop->events[fd];
        int mask = eventLoop->fired[j].mask;
        int fired = 0; /* Number of events fired for current fd. */
        
        // ...
   
        if (!invert && fe->mask & mask & AE_READABLE) {
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
            fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
        }
        
        // ...
    }
```

The `rfileProc` function call above is invoking the same function that was passed in the `aeCreateFileEvent` call:

^^ referenced_code
link:https://github.com/redis/redis/blob/99ab4236afb210dc118fad892210b9fbb369aa8e/src/ae.h#L98
highlighted_lines:10
```c
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    // ...
    
    aeFileEvent *fe = &eventLoop->events[fd];

    // ...
    
    if (mask & AE_READABLE) fe->rfileProc = proc;
    
    // ...
}
```

This loop runs ad-infinitum:

1. wait for events to fire
2. invoke handlers for fired events
3. .. repeat

[file-ae-c]: https://google.com
[initial-commit]: https://google.com
[symbol-aeCreateEventLoop]: https://google.com
[symbol-aeCreateFileEvent]: https://google.com
[symbol-aeEventLoop]: https://google.com
[symbol-initServer]: https://google.com
[video-anatomy-of-a-redis-command]: https://youtu.be/rgE7tZ1yH80?t=129
[video-jake-archibald]: https://www.youtube.com/watch?v=cCOL7MC4Pl0
[video-philip-roberts]: https://www.youtube.com/watch?v=8aGhZQkoFbQ
