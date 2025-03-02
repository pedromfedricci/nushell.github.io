# Hooks

Hooks allow you to run a code snippet at some predefined situations.
They are only available in the interactive mode ([REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop)), they do not work if you run a Nushell with a script (`nu script.nu`) or commands (`nu -c "echo foo"`) arguments.

Currently, we support these types of hooks:

- `pre_prompt` : Triggered before the prompt is drawn
- `pre_execution` : Triggered before the line input starts executing
- `env_change` : Triggered when an environment variable changes

To make it clearer, we can break down Nushell's execution cycle.
The steps to evaluate one line in the REPL mode are as follows:

1. Check for `pre_prompt` hooks and run them
1. Check for `env_change` hooks and run them
1. Display prompt and wait for user input
1. After user typed something and pressed "Enter": Check for `pre_execution` hooks and run them
1. Parse and evaluate user input
1. Return to 1.

## Basic Hooks

To enable hooks, define them in your [config](configuration.md):

```
let-env config = {
    # ...other config...

    hooks: {
        pre_prompt: { print "pre prompt hook" }
        pre_execution: { print "pre exec hook" }
        env_change: {
            PWD: {|before, after| print $"changing directory from ($before) to ($after)" }
        }
    }
}
```

Try putting the above to your config, running Nushell and moving around your filesytem.
When you change a directory, the `PWD` environment variable changes and the change triggers the hook with the previous and the current values stored in `before` and `after` variables, respectively.

Instead of defining a just a single hook per trigger, it is possible to define a **list of hooks** which will run in sequence:

```
let-env config = {
    ...other config...

    hooks: {
        pre_prompt: [
            { print "pre prompt hook" }
            { print "pre prompt hook2" }
        ]
        pre_execution: [
            { print "pre exec hook" }
            { print "pre exec hook2" }
        ]
        env_change: {
            PWD: [
                {|before, after| print $"changing directory from ($before) to ($after)" }
                {|before, after| print $"changing directory from ($before) to ($after) 2" }
            ]
        }
    }
}
```

Also, it might be more practical to update the existing config with new hooks, instead of defining the whole config from scratch:

```
let-env config = ($env.config | upsert hooks {
    pre_prompt: ...
    pre_execution: ...
    env_change: {
        PWD: ...
    }
})
```

## Changing Environment

One feature of the hooks is that they preserve the environment.
Environment variables defined inside the hook **block** will be preserved in a similar way as [`def-env`](environment.md#defining-environment-from-custom-commands).
You can test it with the following example:

```
> let-env config = ($env.config | upsert hooks {
    pre_prompt: { let-env SPAM = "eggs" }
})

> $env.SPAM
eggs
```

The hook blocks otherwise follow the general scoping rules, i.e., commands, aliases, etc. defined within the block will be thrown away once the block ends.

## Conditional Hooks

One thing you might be tempted to do is to activate an environment whenever you enter a directory:

```
let-env config = ($env.config | upsert hooks {
    env_change: {
        PWD: [
            {|before, after|
                if $after == /some/path/to/directory {
                    load-env { SPAM: eggs }
                }
            }
        ]
    }
})
```

This won't work because the environment will be active only within the `if` block.
In this case, you could easily rewrite it as `load-env (if $after == ... { ... } else { {} })` but this pattern is fairly common and later we'll see that not all cases can be rewritten like this.

To deal with the above problem, we introduce another way to define a hook -- **a record**:

```
let-env config = ($env.config | upsert hooks {
    env_change: {
        PWD: [
            {
                condition: {|before, after| $after == /some/path/to/directory }
                code: {|before, after| load-env { SPAM: eggs } }
            }
        ]
    }
})
```

When the hook triggers, it evaluates the `condition` block.
If it returns `true`, the `code` block will be evaluated.
If it returns `false`, nothing will happen.
If it returns something else, an error will be thrown.
The `condition` field can also be omitted altogether in which case the hook will always evaluate.

The `pre_prompt` and `pre_execution` hook types also support the conditional hooks but they don't accept the `before` and `after` parameters.

## Hooks as Strings

So far a hook was defined as a block that preserves only the environment, but nothing else.
To be able to define commands or aliases, it is possible to define the `code` field as **a string**.
You can think of it as if you typed the string into the REPL and hit Enter.
So, the hook from the previous section can be also written as

```
> let-env config = ($env.config | upsert hooks {
    pre_prompt: 'let-env SPAM = "eggs"'
})

> $env.SPAM
eggs
```

This feature can be used, for example, to conditionally bring in definitions based on the current directory:

```
let-env config = ($env.config | upsert hooks {
    env_change: {
        PWD: [
            {
                condition: {|_, after| $after == /some/path/to/directory }
                code: 'def foo [] { print "foo" }'
            }
            {
                condition: {|before, _| $before == /some/path/to/directory }
                code: 'hide foo'
            }
        ]
    }
})
```

When defining a hook as a string, the `$before` and `$after` variables are set to the previous and current environment variable value, respectively, similarly to the previous examples:

```
let-env config = ($env.config | upsert hooks {
    env_change: {
        PWD: {
            code: 'print $"changing directory from ($before) to ($after)"'
        }
    }
}
```

## Examples

### Adding a single hook to existing config

An example for PWD env change hook:

```
let-env config = ($env.config | upsert hooks.env_change.PWD {|config|
    let val = ($config | get -i hooks.env_change.PWD)

    if $val == $nothing {
        $val | append {|before, after| print $"changing directory from ($before) to ($after)" }
    } else {
        [
            {|before, after| print $"changing directory from ($before) to ($after)" }
        ]
    }
})
```

### Automatically activating an environment when entering a directory

This one looks for `test-env.nu` in a directory

```
let-env config = ($env.config | upsert hooks.env_change.PWD {
    [
        {
            condition: {|_, after|
                ($after == '/path/to/target/dir'
                    and ($after | path join test-env.nu | path exists))
            }
            code: "overlay add test-env.nu"
        }
        {
            condition: {|before, after|
                ('/path/to/target/dir' not-in $after
                    and '/path/to/target/dir' in $before
                    and 'test-env' in (overlay list))
            }
            code: "overlay remove test-env --keep-env [ PWD ]"
        }
    ]
})
```
