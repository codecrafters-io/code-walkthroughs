The `PING` command is the simplest of all Redis commands. It always returns `PONG` as a response (that's not exactly true, as 
you'll learn in this walkthrough). This command is often used to test if a connection is still alive, or to measure latency.

The function that handles the `PING` command is [`pingCommand`][function-pingCommand]:

[function-pingCommand]: https://github.com/redis/redis/blob/1e85b89aefe8e7e24a46bd1c8fb251fdab024b74/src/server.c#L4285

^^ referenced_code
link:https://github.com/redis/redis/blob/1e85b89aefe8e7e24a46bd1c8fb251fdab024b74/src/server.c#L4285
```c
void pingCommand(client *c) {
    /* The command takes zero or one arguments. */
    if (c->argc > 2) {
        addReplyErrorArity(c);
        return;
    }
    
    if (c->flags & CLIENT_PUBSUB && c->resp == 2) {
        addReply(c,shared.mbulkhdr[2]);
        addReplyBulkCBuffer(c,"pong",4);
        if (c->argc == 1)
            addReplyBulkCBuffer(c,"",0);
        else
            addReplyBulk(c,c->argv[1]);
    } else {
        if (c->argc == 1)
            addReply(c,shared.pong);
        else
            addReplyBulk(c,c->argv[1]);
    }
}
```

There's a lot to unpack here, let's tackle this step by step. 

#### Step 1: Validate input

In the first 4 lines, we check that the number of arguments given don't exceed what the the `PING` command allows.

^^ referenced_code
link:https://github.com/redis/redis/blob/1e85b89aefe8e7e24a46bd1c8fb251fdab024b74/src/server.c#L4285
highlighted_lines:3,4,5,6
```c
void pingCommand(client *c) {
    /* The command takes zero or one arguments. */
    if (c->argc > 2) {
        addReplyErrorArity(c);
        return;
    }
    
    // ...
}
```

Notice how it says "zero **or one** arguments" above? Although `PING` is mostly used without an argument, it also 
supports an optional argument. If the argument is supplied, it responds with that argument instead of `PONG`. This is
similar to what the [`ECHO`][redis-echo-command] command does.

[redis-echo-command]: https://redis.io/commands/echo

#### Step 2: Respond

The implementation of `PING` differs depending on whether the client is in [Pub/Sub][redis-pubsub-mode] mode or not. Pub/Sub mode is a whole 
topic of its own, so we'll skip that section for now and look at the `else` block.

[redis-pubsub-mode]: https://redis.io/docs/manual/pubsub/

^^ referenced_code
link:https://github.com/redis/redis/blob/1e85b89aefe8e7e24a46bd1c8fb251fdab024b74/src/server.c#L4285
highlighted_lines:7,8,9,10,11,12
```c
void pingCommand(client *c) {
    // Step 1: Validate Input
    
    if (c->flags & CLIENT_PUBSUB && c->resp == 2) {
        // Respond when running in Pub/Sub mode)
    } else {
        if (c->argc == 1)
            // No arguments? Respond with PONG
            addReply(c,shared.pong);
        else
            // One argument? Respond back with the same argument.
            addReplyBulk(c,c->argv[1]);
    }
}
```

# Command Table

We've seen how the `pingCommand` function works, but how does it get called in the first place?

Definitions for all Redis commands are stored in [src/commands.c][file-src-commands-c]. 

[file-src-commands-c]: https://github.com/redis/redis/blob/82b82035553cdbaf81983f91e0402edc8de764ab/src/commands.c

^^ referenced_code
link:https://github.com/redis/redis/blob/82b82035553cdbaf81983f91e0402edc8de764ab/src/commands.c#L7184
highlighted_lines:10
```c
/* Main command table */
struct redisCommand redisCommandTable[] = {
    // ...

    /* connection */
    {"auth","Authenticate to the server","O(N) where N is the number of passwords defined for the user","1.0.0",CMD_DOC_NONE,NULL,NULL,COMMAND_GROUP_CONNECTION,AUTH_History,AUTH_tips,authCommand,-2,CMD_NOSCRIPT|CMD_LOADING|CMD_STALE|CMD_FAST|CMD_NO_AUTH|CMD_SENTINEL|CMD_ALLOW_BUSY,ACL_CATEGORY_CONNECTION,.args=AUTH_Args},
    {"client","A container for client connection commands","Depends on subcommand.","2.4.0",CMD_DOC_NONE,NULL,NULL,COMMAND_GROUP_CONNECTION,CLIENT_History,CLIENT_tips,NULL,-2,CMD_SENTINEL,0,.subcommands=CLIENT_Subcommands},
    {"echo","Echo the given string","O(1)","1.0.0",CMD_DOC_NONE,NULL,NULL,COMMAND_GROUP_CONNECTION,ECHO_History,ECHO_tips,echoCommand,2,CMD_LOADING|CMD_STALE|CMD_FAST,ACL_CATEGORY_CONNECTION,.args=ECHO_Args},
    {"hello","Handshake with Redis","O(1)","6.0.0",CMD_DOC_NONE,NULL,NULL,COMMAND_GROUP_CONNECTION,HELLO_History,HELLO_tips,helloCommand,-1,CMD_NOSCRIPT|CMD_LOADING|CMD_STALE|CMD_FAST|CMD_NO_AUTH|CMD_SENTINEL|CMD_ALLOW_BUSY,ACL_CATEGORY_CONNECTION,.args=HELLO_Args},
    {"ping","Ping the server","O(1)","1.0.0",CMD_DOC_NONE,NULL,NULL,COMMAND_GROUP_CONNECTION,PING_History,PING_tips,pingCommand,-1,CMD_FAST|CMD_SENTINEL,ACL_CATEGORY_CONNECTION,.args=PING_Args},
    {"quit","Close the connection","O(1)","1.0.0",CMD_DOC_NONE,NULL,NULL,COMMAND_GROUP_CONNECTION,QUIT_History,QUIT_tips,quitCommand,-1,CMD_ALLOW_BUSY|CMD_NOSCRIPT|CMD_LOADING|CMD_STALE|CMD_FAST|CMD_NO_AUTH,ACL_CATEGORY_CONNECTION},
    {"reset","Reset the connection","O(1)","6.2.0",CMD_DOC_NONE,NULL,NULL,COMMAND_GROUP_CONNECTION,RESET_History,RESET_tips,resetCommand,1,CMD_NOSCRIPT|CMD_LOADING|CMD_STALE|CMD_FAST|CMD_NO_AUTH|CMD_ALLOW_BUSY,ACL_CATEGORY_CONNECTION},
    {"select","Change the selected database for the current connection","O(1)","1.0.0",CMD_DOC_NONE,NULL,NULL,COMMAND_GROUP_CONNECTION,SELECT_History,SELECT_tips,selectCommand,2,CMD_LOADING|CMD_STALE|CMD_FAST,ACL_CATEGORY_CONNECTION,.args=SELECT_Args}

    // ...
}
```

[src/commands.c][file-src-commands-c] is a **huge** file - almost 7,500 lines long. For each command, it stores details 
like the name of the command, how many arguments it takes and which function implements the command. 

This file is machine-generated by the [`utils/generate-command-code.py`][file-utils-generate-command-code] script. The 
script takes files like [`src/commands/ping.json`][file-src-commands-ping-json] as input:

[file-utils-generate-command-code]: https://github.com/redis/redis/blob/82b82035553cdbaf81983f91e0402edc8de764ab/utils/generate-command-code.py
[file-src-commands-ping-json]: https://github.com/redis/redis/blob/82b82035553cdbaf81983f91e0402edc8de764ab/src/commands/ping.json
[file-src-commands-c]: https://github.com/redis/redis/blob/82b82035553cdbaf81983f91e0402edc8de764ab/src/commands.c

^^ referenced_code
link:https://github.com/redis/redis/blob/82b82035553cdbaf81983f91e0402edc8de764ab/src/commands/ping.json
highlighted_lines:8
```c
{
    "PING": {
        "summary": "Ping the server",
        "complexity": "O(1)",
        "group": "connection",
        "since": "1.0.0",
        "arity": -1,
        "function": "pingCommand",
        "command_flags": [
            "FAST",
            "SENTINEL"
        ],
        "acl_categories": [
            "CONNECTION"
        ],
        "command_tips": [
            "REQUEST_POLICY:ALL_SHARDS",
            "RESPONSE_POLICY:ALL_SUCCEEDED"
        ],
        "arguments": [
            {
                "name": "message",
                "type": "string",
                "optional": true
            }
        ]
    }
}
```
