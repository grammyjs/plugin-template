# The Hitchhiker's Guide to grammY Plugins
grammY is very extensible and it supports installing plugins, but how does one go about doing it? In this article we will cover all the steps of developing a plugin for grammY bot framework.

# Table of Content
- [The Hitchhiker's Guide to grammY Plugins](#the-hitchhikers-guide-to-grammy-plugins)
- [Table of Content](#table-of-content)
- [What Is a Plugin?](#what-is-a-plugin)
- [Types of Plugins in grammY](#types-of-plugins-in-grammy)
  - [Type I: Middleware Plugins](#type-i-middleware-plugins)
  - [Type II: Transformer Plugins](#type-ii-transformer-plugins)
- [Rules of Contribution](#rules-of-contribution)
- [Designing a Dummy Middleware Plugin](#designing-a-dummy-middleware-plugin)
- [Designing a Dummy Transformer Plugin](#designing-a-dummy-transformer-plugin)
- [Exctraction Into a Plugin](#exctraction-into-a-plugin)
- [There Is a Boilerplate!](#there-is-a-boilerplate)
- [I Don't Like Deno!](#i-dont-like-deno)
- [How to Submit?](#how-to-submit)

# What Is a Plugin?
There is a popular principle in computing stating that software should be consice and minimal, but extensible. Why? Because not everyone uses everything! Plugins are designed as extra functionalities added to said pieces of software.

# Types of Plugins in grammY
All that glitters is gold, right? well, a different kind of gold! grammY can take advantage of two types of plugins: middleware plugins and transformer plugins. In simple terms, plugins in grammY return either a middleware function or a transformer one. Let's talk about the differences.

## Type I: Middleware Plugins
A [middleware](https://grammy.dev/guide/middleware.html) is a function that handles incoming data in various forms. Middleware plugins are plugins that are fed to a bot as a - well you guessed it - middleware. This means that your plugin's sole job is to return a middleware that can be fed to grammY.

## Type II: Transformer Plugins
A [transformer](https://grammy.dev/advanced/transformers.html#installing-a-transformer-function) is a exactly the opposite of a middleware! It is a function that handles outgoing data. Transformer plugins are plugins that are fed to a bot as a - crazy! guessed it again - transformer. Similarly, this type of plugin returns a transformer function.

# Rules of Contribution
Before diving into some hands-on examples, there are some notes to pay attention to it you would like your plugins to be submitted to the documentation:

1. You should document your plugin (README with instructions).
2. Explain the purpose of your plugin and how to use it by adding a page to the [docs](https://github.com/grammyjs/website).
3. Choose a permisive license such as MIT or ISC.

Finally you should know that even though grammY supports both node and [deno](https://deno.land/), it is a deno project first and we also encourage you to write your pluggins in deno (and subsequently in style!). There is a handy-dandy tool called [deno2node](https://github.com/wojpawlik/deno2node) that transpiles your code from deno to node so we can support both platforms. This is **NOT** a must, however very much encouragd.

# Designing a Dummy Middleware Plugin
Let's assume we would like to design a plugin that only responds to certain user ids! Here is a dummy example:

```ts
// Importing the types from grammY - I have them in a file called deps.deno.ts
import type { Context, NextFunction } from './deps.deno.ts'

// assume 2 user types in our bot
type userTypes = "EVEN ID" | "ODD ID"

// your plugin can have 1 main function that returns a middleware (remember this is a middleware plugin)
export const onlyAccept = (userType: userTypes = "EVEN ID") => {

    // The actual middleware returned
    const middleware = async (ctx: Context, next: NextFunction) => {

        // check if there is a from.id
        const userId = ctx.from?.id;
        if (!userId) return await next();

        // check if the user id is an even number
        const isEven = userId % 2 === 0;

        // scenario 1: Bot only responds to users with even id numbers
        if (userType === "EVEN ID" && isEven) {
            ctx.reply(`Welcome ${ctx.from.first_name}`);

            // return next so the request does not die inside our plugin
            return await next();
        }

        // scenario 2: Bot only responds to users with odd id numbers
        if (userType === "ODD ID" && !isEven) {
            ctx.reply(`Welcome ${ctx.from.first_name}`);
            return await next();
        }
    }

    // make sure to return the actual middleware from the main plugin function
    return middleware;
}
```
Now it can be used in a real bot:

```ts
import { onlyAccept } from './testing.ts'
import { Bot } from './deps.deno.ts'

const bot = new Bot("YOUR BOT TOKEN HERE");

bot.use(onlyAccept("ODD ID"));

bot.on("message", (ctx) => { ctx.reply("You passed the middleware plugin") })

bot.start();
```

Voilà! You got yourself a plugin, right? not so fast. We still need to package it up, but before that let's take a look at transformer plugins as well.

# Designing a Dummy Transformer Plugin
Imagine writing a plugin that retries failed requests automatically. Pretty cool, right?

```ts
// A function to pause before retrying
function pause(seconds: number) {
    return new Promise(resolve => setTimeout(resolve, 1000 * seconds))
}

type AutoRetryTransformer = (...args: any[]) => any

export interface AutoRetryOptions {
    maxDelaySeconds: number
    maxRetryAttempts: number
}

// the main plugin function
export function autoRetry(options?: AutoRetryOptions): AutoRetryTransformer {

    // setting the timings or using the default options
    const maxDelay = options?.maxDelaySeconds ?? 3600
    const maxRetries = options?.maxRetryAttempts ?? 3

    // the actual transformer
    return async (prev, method, payload) => {

        let remainingAttempts = maxRetries
        let result = await prev(method, payload)

        // as long as we don't have a result and
        // we have not exceeded the retry limit, we keep retrying
        while (
            !result.ok &&
            typeof result.parameters?.retry_after === 'number' &&
            result.parameters.retry_after <= maxDelay &&
            remainingAttempts-- > 0
        ) {
            await pause(result.parameters.retry_after)
            result = await prev(method, payload)
        }

        // return the result eventually
        return result
    }
}
```

and we can use it:

```ts
import { autoRetry } from './testing.ts'
import { Bot } from './deps.deno.ts'

const bot = new Bot("YOUR BOT TOKEN HERE");

bot.api.config.use(autoRetry())

bot.start();
```
This is an official plugin of grammY called [auto-retry](https://grammy.dev/plugins/auto-retry.html#plugin-summary)

# Exctraction Into a Plugin
Whichever type of plugin you made, you have to bundle it in a standalone package. This is a fairly simple task. There are no specific rules on how to do this and npm is your oyster, but just to keep things organized we have a template suggestion for you. You can download the code from this repository and start developing your plugin without any time spent on configuration.

The initial suggested folder structure: 

```
plugin-template/
├─ src/
│  ├─ deps.deno.ts
│  ├─ deps.node.ts
│  ├─ index.ts
├─ package.json
├─ tsconfig.json
├─ README.md

```
**`deps` Files**: This is for the developers who are willing to write the plugin for deno first and then transpile it to node. As mentioned before, we use deno2node to transpile our deno code into node. deno2node has a feature which allows you to provide runtime-specific files to it. These files should be adjacent to each other and follow the `*.deno.ts` and `*.node.ts` name structure as [explained it the docs](https://github.com/wojpawlik/deno2node#runtime-specific-code). This is why there are two files `deps.deno.ts` and `deps.node.ts`. If there are any node specific dependencies put them in `deps.node.ts`, otherwise leave it empty. 

> ***Note***: You may also use other tools such as [deno dnt](https://github.com/denoland/dnt) to transpile your deno codebase or use other folder structures. The tooling is irrelevant and the point is that writing deno is better and easier.

**`tsconfig.json`**: This is the tsc config used by deno2node to transpile your code. A default one is provided in the repository as a suggestion.

**`package.json`**: The package.json file for the npm version of your plugin. **Make sure to change it according to your project**.

**`README.md`**: Instructions on how to use the plugin. **Make sure to change it according to your project**.

**`index.ts`**: The file containing your business logic (your main code).


# There Is a Boilerplate!
If you would like to develop a plugin for grammY and do not know where to start, we highly suggest the template code in this repository. You can clone the code for yourself and start coding based on what was covered in this article. This repository also includes some extra goodies such as `.editorconfig`, `LICENSE`, `.gitignore`, etc. but you may choose to delete them.

# I Don't Like Deno!
Well, your missing out! but you may also write your plugins in node. In such a case, you may use any folder structure you please as long as it is organized like any other npm project. Simply install grammY through npm with `npm install grammy` and get coding.


# How to Submit?
If you have a plugin ready, you may simply submit a pull request on GitHub (accoding to the [Rules of Contribution](#rules-of-contribution)) or notify us in the [community chat](https://t.me/grammyjs) for further help.