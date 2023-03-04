---
title: "Some tips when using T3 Stack: Part 1"
datePublished: Sat Mar 04 2023 09:56:50 GMT+0000 (Coordinated Universal Time)
cuid: cletsiicd000e08l06w3le65q
slug: some-tips-when-using-t3-stack-part-1
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1677923475166/b1be1008-1c69-4eec-8e59-7d33be8c8f51.jpeg
tags: nextjs, prisma, trpc, t3stack

---

In my last [article](https://hashnode.com/post/cldr3hwu8000609mi52pz7jem), I introduced the side project that I'm building and the tools (stacks) I'm using. And explain why I choose [T3 Stack](https://create.t3.gg/) to build the project. Good, in this article I'll share some tips I discovered using T3 Stack when building the project. Let's go!

## Reminder

* NextAuth needs/uses for authentication a model User with some pre-configured columns like name, image, email, etc.
    
* With T3 Stack we use Prisma as ORM
    
* We have added some columns to the User model like the username
    

## NextAuth and Module augmentation

[NextAuth](https://next-auth.js.org/) is used for authentication in Next.js applications and is packed in T3 Stack. When using NextAuth, the hook `useSession` returns session data that can be checked to verify if someone is signed like this:

```javascript
const index = () => {
    const { data: session } = useSession()
    if (session) {
        return <div>You're signed as {session.user.email}</div>    
    }
    return <div>Not signed, we'll redirect you</div>
}
```

But for security reasons, the session data returned (by default) contained just

```typescript
export interface DefaultSession {
    user?: {
        name?: string | null;
        email?: string | null;
        image?: string | null;
    };
    expires: ISODateString;
}
```

So what can we do if we want to add a new property like the username which is a column of our User model that we define with Prisma?

T3 dedicated's [section](https://create.t3.gg/en/usage/next-auth) to NextAuth and NextAuth [itself](https://next-auth.js.org/getting-started/typescript#module-augmentation) already documented this part. To do it, we'll use module augmentation offered by TypeScript which allows a user to override classes, modules,... types that he/she doesn't access (maybe from a lib). To do it,

1. We can override the Session interface in the **next-auth module** declaration inside the file ***types/next-auth.d.ts.***
    
    ```typescript
    import { DefaultSession } from "next-auth";
    
    declare module "next-auth" {
      interface Session {
        user?: {
          id: string;
          username: string | null  
        } & DefaultSession["user"];
      }
    }
    ```
    
    We import `next-auth` module and override the Session interface by adding not only a `username` but also merging the type of user in DefaultSession so we still have the default properties of the user provided by NextAuth.
    
2. Then in the ***/pages/api/auth/\[...nextauth\].ts*** file we can override session callback by adding `username` like this
    
    ```typescript
    callbacks: {
        session({ session, user }) {
          if (session.user) {
            session.user.id = user.id;
            session.user.username = user.username;
          }
          return session;
        },
      },
    ```
    
    But if you leave it like this, you'll get this warning from TS: `Property 'username' does not exist on type 'User | AdapterUser'`. Why?
    
    Because, if you go to the TS definition of `User` (used by Nextauth) you'll see that it extends `DefaultUser` which has the following definition (the minimum)
    
    ```typescript
    export interface DefaultUser {
        id: string;
        name?: string | null;
        email?: string | null;
        image?: string | null;
    }
    ```
    
    So to fix this error, I think you guess the solution, yes: override the declaration of `User` as we did it with `Session` little above.
    
3. So back to the ***types/next-auth.d.ts,*** we can override `User` after the `Session` interface (inside next-auth declaration) like this
    
    ```typescript
      interface User extends DefaultUser{
        username: string | null
      }
    ```
    
    We've just override `User` interface by firstly extending `DefaultUser` (provided by NextAuth) and secondly adding a `username`.
    

Let's move to another tip.

## Models types deductions (with tRPC)

With Prisma, we define our tables as models inside ***schema.prisma*** file but we don't have a direct access to their types in TypeScript because it generates types definitions in ***node\_modules/.prisma/client/index.d.ts*** by default (read this [link](https://www.prisma.io/docs/concepts/components/prisma-client/working-with-prismaclient/generating-prisma-client#why-is-prisma-client-generated-into-node_modulesprismaclient-by-default) if you want to change this output file location). But there are some ways to get these types:

1. Use the return type of Prisma client methods like this
    
    ```typescript
    const getById = async () => await prisma.post.findUnique({
    //....
    })
    
    type Post = ReturnType<typeof getById>
    ```
    
2. Or you can use the `GetPayload` version of the model type where we can specify all relations between this model and others
    
    ```typescript
    import { PrismaClient, Prisma } from "@prisma/client";
    type User = Prisma.UserGetPayload<{
        // here you can specify all relations like: select, include, ...
    }>
    ```
    
    You can read more [here](https://www.prisma.io/docs/concepts/components/prisma-client/advanced-type-safety/prisma-validator) and on this [link](https://www.prisma.io/docs/concepts/components/prisma-client/advanced-type-safety/prisma-validator).
    

I don't recommend you these solutions because we have **tRPC** (don't blame me).

In our case, we use Prisma client through tRPC. And you'll be happy to know that tRPC has a utility to infer routers' input/return types. Let's suppose that we have a router named `user` and we define inside this router a procedure `retrieve`. To get the return type in plain tRPC of this procedure we can do:

```typescript
import type { inferRouterInputs } from "@trpc/server"

type User = inferRouterInputs<AppRouter>['user']['retrieve']
```

Nice! But how will you be if I say that in T3 Stack there is a little helper to make this operation easier:

```typescript
import {RouterOutputs} from "@utils/api";
type User = RouterOutputs['project']['getById']
```

And yes, if you go to look inside the ***@utils/api*** you'll see that

```typescript
export type RouterOutputs = inferRouterOutputs<AppRouter>;
```

Anyway, it's soft to use.

**Bonus:**

With this approach, TS can warm you that `User` type could be null, so to avoid it you can use the `NonNullable` utility:

```typescript
import {RouterOutputs} from "@utils/api";
type User = NonNullable<RouterOutputs['project']['getById']>
```

Right!

## Conclusion

I love the end-to-end typesafe that we get from tRPC and it's very cool to work with T3 Stack where they put together some great libs and frameworks to make full-stack development with JS much easier.

We are at the end of this article. I hope you enjoy it. If I get another tip while building my SaaS, I'll share them. Feedback or anything that could help me to improve this post is welcome.