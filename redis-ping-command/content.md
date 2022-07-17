When a Redis Server boots, one of the first functions that gets called is [`initServer`][function-initServer]. It is
within this function that the Redis server binds to a port and starts listening for connections:

[function-initServer]: https://github.com/redis/redis/blob/1e85b89aefe8e7e24a46bd1c8fb251fdab024b74/src/server.c#L2391

^^ referenced_code
link:https://github.com/redis/redis/blob/1e85b89aefe8e7e24a46bd1c8fb251fdab024b74/src/server.c#L2391
highlighted_lines:5,8
```c
void initServer(void) {
    // ... 
    
    /* Open the TCP listening socket for the user commands. */
    if (server.port != 0 && listenToPort(server.port,&server.ipfd) == C_ERR) {
        exit(1);
    }
    if (server.tls_port != 0 && listenToPort(server.tls_port,&server.tlsfd) == C_ERR) {
        exit(1);
    }
    
    // ...
}
```

If you take a a closer look at [`listenToPort`][function-listenToPort], you'll see that it delegates to two functions: [`anetTcpServer`][function-anetTcpServer] and
[`anetTcp6Server`][function-anetTcp6Server]. These use [`listen`][unix-listen] from `sys/socket.h` under the hood.

[function-anetTcp6Server]: https://github.com/redis/redis/blob/ef68deb3c2a4d6205ddc84141d4d84b6e53cbc1b/src/anet.c#L476
[function-anetTcpServer]: https://github.com/redis/redis/blob/ef68deb3c2a4d6205ddc84141d4d84b6e53cbc1b/src/anet.c#L481
[function-listenToPort]: https://github.com/redis/redis/blob/8203461120bf244e5c0576222c6aa5d986587bca/src/server.c#L2282
[unix-listen]: https://man7.org/linux/man-pages/man2/listen.2.html

^^ referenced_code
link:https://github.com/redis/redis/blob/8203461120bf244e5c0576222c6aa5d986587bca/src/server.c#L2282-L2322
highlighted_lines:9,12
```c
/* Initialize a set of file descriptors to listen to the specified 'port'
 * binding the addresses specified in the Redis server configuration.
 */
int listenToPort(int port, socketFds *sfd) {
    // ... 
    
    if (strchr(addr,':')) {
        /* Bind IPv6 address. */
        sfd->fd[sfd->count] = anetTcp6Server(server.neterr,port,addr,server.tcp_backlog);
    } else {
        /* Bind IPv4 address. */
        sfd->fd[sfd->count] = anetTcpServer(server.neterr,port,addr,server.tcp_backlog);
    }
    
    // ...
}
```

# Handling port updates

Fun fact: the port on a running Redis server can be updated without a restart!

This can be done by running the [`CONFIG SET`][redis-config-set-command] command: `CONFIG SET port <new_port>`.

The function that handles port updates is [`changeListenPort`][function-changeListenPort]:

[redis-config-set-command]: https://redis.io/commands/config-set
[function-changeListenPort]: https://github.com/redis/redis/blob/8203461120bf244e5c0576222c6aa5d986587bca/src/server.c#L6193-L6223

^^ referenced_code
link:https://github.com/redis/redis/blob/ef68deb3c2a4d6205ddc84141d4d84b6e53cbc1b/src/server.c#L6260-L6290
highlighted_lines:3,6
```c
int changeListenPort(int port, socketFds *sfd, aeFileProc *accept_handler) {
    /* Close old servers */
    closeSocketListeners(sfd);
    
    /* Bind to the new port */
    if (listenToPort(port, &new_sfd) != C_OK) {
        return C_ERR;
    }
}
```

Not all config options can be updated like we saw with `port`. We can see why this is by looking at
[`src/config.c`][file-src-config-c], which is where config options are defined.

[file-src-config-c]: https://github.com/redis/redis/blob/99a425d0f3b7b00896cb855d5de4ae93be1fe3f0/src/config.c#L3024

^^ referenced_code
link:https://github.com/redis/redis/blob/99a425d0f3b7b00896cb855d5de4ae93be1fe3f0/src/config.c#L3022
highlighted_lines:3
```c
/* Integer configs */
createIntConfig("databases", NULL, IMMUTABLE_CONFIG, 1, INT_MAX, server.dbnum, 16, INTEGER_CONFIG, NULL, NULL),
createIntConfig("port", NULL, MODIFIABLE_CONFIG, 0, 65535, server.port, 6379, INTEGER_CONFIG, NULL, updatePort),
```

In the above snippet you'll notice that the `createIntConfig` invocation for `port` has `updatePort` as the last
parameter, but the one for `databases` has `NULL`. This last parameter is what gets called when a config option is
updated. If it is present, a config option can be updated on the fly.
