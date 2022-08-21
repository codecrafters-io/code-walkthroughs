### What does git init do?

[`git init`][git-init-command] creates an empty Git repository. This basically involves creating a bunch of files in the 
`.git` directory. Here's the list of files `git init` creates: 

```shell
.git
├── HEAD
├── config
├── description
├── hooks
│   ├── applypatch-msg.sample
│   ├── commit-msg.sample
│   ├── fsmonitor-watchman.sample
│   ├── post-update.sample
│   ├── pre-applypatch.sample
│   ├── pre-commit.sample
│   ├── pre-merge-commit.sample
│   ├── pre-push.sample
│   ├── pre-rebase.sample
│   ├── pre-receive.sample
│   ├── prepare-commit-msg.sample
│   ├── push-to-checkout.sample
│   └── update.sample
├── info
│   └── exclude
├── objects
│   ├── info
│   └── pack
└── refs
    ├── heads
    └── tags

8 directories, 17 files
```

That's a lot of files! We'll look at most of them in this walkthrough. 

### Entrypoint

The entry point for `git init` is the [`cmd_init_db`][fn-cmd-init-db] function.

^^ referenced_code
link:https://github.com/git/git/blob/90d242d36e248acfae0033274b524bfa55a947fd/builtin/init-db.c#L528
highlighted_lines:5
```c
int cmd_init_db(int argc, const char **argv, const char *prefix)
{
    // Parse options, validate user input...
    
	return init_db(git_dir, real_git_dir, template_dir, hash_algo, initial_branch, flags);
}
```

`cmd_init_db` primarily does a bunch of user input validation (are flags valid, is the directory already a git repository etc.) 
and delegates to [`init_db`][fn-init-db].

^^ referenced_code
link:https://github.com/git/git/blob/90d242d36e248acfae0033274b524bfa55a947fd/builtin/init-db.c#L384
highlighted_lines:7,11
```c
int init_db(const char *git_dir, const char *real_git_dir,
	    const char *template_dir, int hash, const char *initial_branch,
	    unsigned int flags)
{
    // ...
    
	safe_create_dir(git_dir, 0);
    
	// ...

	create_object_directory();
    
	// ... 
    
	return 0;
}
```

`init_db` is a pretty long function, but primarily it is creating the files we listed above.

Before we dive into the files `init_db` creates, a bit of history...

### History

You might've noticed that the entrypoint for the init command is named `cmd_init_db` and not `cmd_init`. 

That's inconsistent with with how other commands like `git add` are named (`cmd_add`): 

^^ referenced_code
link:https://github.com/git/git/blob/795ea8776befc95ea2becd8020c7a284677b4161/git.c#L549-L550
highlighted_lines:2
```c
static struct cmd_struct commands[] = {
	{ "add", cmd_add, RUN_SETUP | NEED_WORK_TREE },
	{ "am", cmd_am, RUN_SETUP | NEED_WORK_TREE },
	{ "annotate", cmd_annotate, RUN_SETUP | NO_PARSEOPT },
	{ "apply", cmd_apply, RUN_SETUP_GENTLY },
	{ "archive", cmd_archive, RUN_SETUP_GENTLY },
	{ "bisect--helper", cmd_bisect__helper, RUN_SETUP },
	{ "blame", cmd_blame, RUN_SETUP },
	{ "branch", cmd_branch, RUN_SETUP | DELAY_PAGER_CONFIG },
	{ "bugreport", cmd_bugreport, RUN_SETUP_GENTLY },
	{ "bundle", cmd_bundle, RUN_SETUP_GENTLY | NO_PARSEOPT },
	{ "cat-file", cmd_cat_file, RUN_SETUP },
    	
	// ...
    	
}
```

There's a reason for this.

In the first version of Git, `init` was named `init-db`. It was [renamed][git-init-db-to-init-rename] to `init` in 2007. 
Git still supports `init-db` to this day:

[git-init-db-to-init-rename]: https://github.com/git/git/commit/515377ea9ec6192f82a2fa5c5b5b7651d9d6cf6c

^^ referenced_code
link:https://github.com/git/git/blob/795ea8776befc95ea2becd8020c7a284677b4161/git.c#L549-L550
highlighted_lines:5
```c
static struct cmd_struct commands[] = {
    // ... commands
    
	{ "init", cmd_init_db },
	{ "init-db", cmd_init_db },
    
    // ... commands	
};
```

### Files in the .git directory

Let's look at the files created in the `.git` directory. 

#### 1. objects

The `.git/objects` directory stores data for [Git Objects][git-objects], like blobs, commits and trees.

#### 2. HEAD

The `HEAD` file contains information on the current branch.

```shell
$ cat .git/HEAD
ref: refs/heads/master
```

`refs/heads/master` is a [Git reference][git-ref], a friendly name that points to a
[Git object][git-objects]. Branches are Git refs that point to commits.

#### 3. refs

Git references (refs, for short) are stored in the `.git/refs` directory. An empty repository won't contain any refs by default, but once a repository
has branches, the `.git/refs` directory will look something like this:

```shell
.git/refs
├── heads
│   ├── main
│   └── branch1
│   └── branch2
│   └── branch3
│   └── ...
```

So when you're creating a new branch with `git checkout -b new-branch`, under the hood that's just creating the
`.git/refs/heads/new-branch` file.

[git-objects]: https://git-scm.com/book/en/v2/Git-Internals-Git-Objects
[git-ref]: https://git-scm.com/book/en/v2/Git-Internals-Git-References
[fn-cmd-init-db]: https://github.com/git/git/blob/90d242d36e248acfae0033274b524bfa55a947fd/builtin/init-db.c#L528
[fn-init-db]: https://github.com/git/git/blob/90d242d36e248acfae0033274b524bfa55a947fd/builtin/init-db.c#L384
[git-init-command]: https://git-scm.com/docs/git-init

