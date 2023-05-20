---
title: "Some tips when using T3 Stack: Unit Testing with tRPC procedures - environment setup"
datePublished: Sat May 20 2023 21:27:45 GMT+0000 (Coordinated Universal Time)
cuid: clhwi3mr9000109mgcfc8h604
slug: some-tips-when-using-t3-stack-unit-testing-with-trpc-procedures-environment-setup
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1684606664630/6cff47ad-ce67-4893-bad1-289974d86e08.jpeg
tags: reactjs, testing, prisma, trpc

---

In this [article](https://hashnode.com/post/cldr3hwu8000609mi52pz7jem), I introduced the side project that I'm building and the tools (stacks) I'm using. And in my last [article](https://tawaldevuniverse.hashnode.dev/some-tips-when-using-t3-stack-nextauth-and-models-types), I talk about NextAuth and module augmentation. Here I'll share with you some tips I used when I was learning **unit tests** with tRPC procedures and Prisma. We'll setup our test environment here and in a future article we'll write little tests. At the end, you'll have some useful links that I used to do it.

## Unit testing with tRPC + Prisma?

Unit testing in software development is the process to check if a small (*unit)* part of the code executes like you want. You write code to test if some code behaves like you want without logging something. And I try to do it with tRPC + Prisma while using T3 Stack and it was funny. Here is the procedure I want to test

```typescript
getByProjectIdAndPos: protectedProcedure
    .input(
      z.object({
        projectId: z.string().nullish(),
        pos: z.number().nonnegative()
      })
    )
		.query(async ({ ctx, input }) => {
			// 1. should return undefined if projectId is not provided
      if (!input.projectId) return undefined

      const countExercises = await ctx.prisma.exercise.count({
        where: { projectId: input.projectId }
      })

			// 2. should throw an error if the provided pos is superior to number of exercises
      if (input.pos >= countExercises) {
        throw new TRPCError({
          code: 'NOT_FOUND',
          message: `No exercise at position '${input.pos + 1}'`
        })
      }

			// 3. should return one exercise if position < countExercises
			// 4. should return undefined if no exercise at this position
      return await ctx.prisma.exercise.findFirst({
        select: defaultExerciseSelect,
        where: {
          projectId: input.projectId
        },
        orderBy: {
          order: 'asc'
        },
        skip: input.pos,
        take: 1
      })
    })
```

Based on the position of the exercise provided, this procedure returns an exercise or throws an error. You can read the comments in the code to know what we want to test. But we have a few problems:

## Some difficulties to test this procedure

1. How to test a protected procedure?
    
2. How to test the body of this procedure where we're using Prisma client?
    

One answer for all of them: Mocking

%[https://twitter.com/Tawal_Mc/status/1651986440467755031] 

So we need to *mock* the object that made this procedure to be protected and after we can mock Prisma client methods too.

## What makes a tRPC procedure protected in T3 Stack?

A protected procedure is a tRPC procedure whose access is done when the request is authenticated so the request needs a valid session. Check the session on NextAuth to know more about it. So we must mock a session if we want to test this procedure. And since we're using T3 Stack and Prisma as ORM, the prisma client also is set when creating the tRPC context. Look at the file ***src/server/api/trpc.ts*** file you will get this

```typescript
// I keep only useful code for the explanations
import type {CreateNextContextOptions} from "@trpc/server/adapters/next";
import type {Session} from "next-auth";

import {getServerAuthSession} from "../auth";
import {prisma} from "../db";

import {initTRPC, TRPCError} from "@trpc/server";

type CreateContextOptions = {
  session: Session | null;
};

/**
 * This helper generates the "internals" for a tRPC context. If you need to use
 * it, you can export it from here
 *
 * Examples of things you may need it for:
 * - testing, so we dont have to mock Next.js' req/res
 * - trpc's `createSSGHelpers` where we don't have req/res
 * @see https://create.t3.gg/en/usage/trpc#-servertrpccontextts
 */
export const createInnerTRPCContext = (opts: CreateContextOptions) => {
  return {
    session: opts.session,
    prisma,
  };
};

/**
 * This is the actual context you'll use in your router. It will be used to
 * process every request that goes through your tRPC endpoint
 * @link https://trpc.io/docs/context
 */
export const createTRPCContext = async (opts: CreateNextContextOptions) => {
  const { req, res } = opts;

  // Get the session from the server using the unstable_getServerSession wrapper function
  const session = await getServerAuthSession({ req, res });

  return createInnerTRPCContext({
    session,
  });
};

/**
 * 3. ROUTER & PROCEDURE (THE IMPORTANT BIT)
 *
 * These are the pieces you use to build your tRPC API. You should import these
 * a lot in the /src/server/api/routers folder
 */

/**
 * This is how you create new routers and subrouters in your tRPC API
 * @see https://trpc.io/docs/router
 */
export const createTRPCRouter = t.router;

/**
 * Public (unauthed) procedure
 *
 * This is the base piece you use to build new queries and mutations on your
 * tRPC API. It does not guarantee that a user querying is authorized, but you
 * can still access user session data if they are logged in
 */
export const publicProcedure = t.procedure;

/**
 * Reusable middleware that enforces users are logged in before running the
 * procedure
 */
const enforceUserIsAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.session || !ctx.session.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({
    ctx: {
      // infers the `session` as non-nullable
      session: { ...ctx.session, user: ctx.session.user },
    },
  });
});

/**
 * Protected (authed) procedure
 *
 * If you want a query or mutation to ONLY be accessible to logged in users, use
 * this. It verifies the session is valid and guarantees ctx.session.user is not
 * null
 *
 * @see https://trpc.io/docs/procedures
 */
export const protectedProcedure = t.procedure.use(enforceUserIsAuthed);
```

You can see that the ***createInnerTRPCContext*** takes a session as parameters and returns this session with an instance of the prisma client: it's what we'll use... Inside the [section tRPC](https://create.t3.gg/en/usage/trpc#sample-integration-test) on T3 Stack documentation, you can find the solution

```typescript
test("protected example router", async () => {
  const ctx = await createInnerTRPCContext({
    session: {
      user: { id: "123", name: "John Doe" },
      expires: "1",
    },
  });
  const caller = appRouter.createCaller(ctx);

  // ...
});
```

or simply

```typescript
test("protected example router", async () => {
  const session = {
      user: { id: "123", name: "John Doe" },
      expires: "1",
    },

  // The two lines below are the most important
  const ctx = await createInnerTRPCContext({session});
  const caller = appRouter.createCaller(ctx);
});
```

In our case we'll add the prisma client instance

```typescript
// I'll explain this line after: we create a mock version of the prisma client to avoid to have a direct access to the db
import prismaMock from '@/server/__mocks__/db'

test("protected example router", async () => {
  const session = {
      user: { id: "123", name: "John Doe" },
      expires: "1",
    },

  const ctx = await createInnerTRPCContext({session});
  const caller = appRouter.createCaller({...ctx, prisma: prismaMock});
  
   // And then
   caller.exercise.getByProjectIdAndPos({ projectId: '1', pos: 1 })    
});
```

we'll after explain the first line ***import prismaMock from '@/server/\_\_mocks\_\_/db***.

## Setup our environment

### Add vitest and mock package

I'm using vitest because it's blazing fast 🤣 and seems to be the new choice for tests in JS area.

```bash
yarn add vitest vitest-mock-extended -D
```

### File for test

Let's create the file inside ***src/server/api/routers/tests/exo*** where we'll put our tests related to exo.

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

vi.mock('../../../db') // 1

describe('exo procedures testing', () => {
  // 2  
  beforeEach(() => {
    vi.restoreAllMocks()
  })
)
```

1. Vitest will mock the module found from the given path
    
2. Will reset all mocked functions/methods
    

**NB**: In the version of T3 Stack that I'm using (7.1.0, yes I need to upgrade), I add a default export of prisma inside ***src/server/db.ts***

### Mock the prisma client

Inside ***/server/\_\_mocks\_\_/db***, we have a deep-mocked (mock Prisma's method like *findFirst*, *create*, etc) version of the Prisma client which is reset before we run each test to avoid having direct access to the DB (we'll install vitest soon).

```typescript
import type { PrismaClient } from "@prisma/client"
import {beforeEach} from "vitest"
import {mockDeep, mockReset} from "vitest-mock-extended";

beforeEach(() => {
    mockReset(db)
})

const db = mockDeep<PrismaClient>()
export default db
```

### Mock the session object

Not something hard, we need just to create a [Session](https://tawaldevuniverse.hashnode.dev/some-tips-when-using-t3-stack-nextauth-and-models-types#heading-nextauth-and-module-augmentation) object

***src/server/api/routers/tests/exo***

```typescript
import type { Session } from 'next-auth'
// code ...

describe("exo's procedures testing", () => {
    const session: Session = {
    expires: '1',
    user: {
      id: 'clgb17vnp000008jjere5g15i',
      username: ''
    }
  }
// code ..
}
```

So now we can imitate the protected behavior of our procedures after mocking:

* our session object
    
* Prisma client
    

### Configure tRPC for tests

What we're doing is like to create a little environment (by mocking some external libs) for our tests so we'll do the same with tRPC by creating a context that'll be used in our tests only. So let's add after the session object the following code to our test file

***src/server/api/routers/tests/exo***

```typescript
  // 1
  const ctx = createInnerTRPCContext({ session })
  // 2  
  const caller = appRouter.createCaller({ ...ctx, prisma: prismaMock }
```

1. We pass to *createInnerTRPCContext* function provided by tRPC the session object we created (the mocked version of the session). So it's like we have an authenticated user represented by this session.
    
2. *caller* will allow us to have access to the procedure we defined in our tRPC routers so it takes the mocked version of prisma and the context we created earlier
    

With that we can do something like this

```typescript
await caller.targetTable.targetProcedure(/* input here*/)
```

### Final Setup

If we put everything together we get this

```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'
import type { Session } from 'next-auth'

// 1- mock prisma module
vi.mock('../../../db') 

// 2- our tests 
describe('exo procedures testing', () => {
// 3- Reset everything  
  beforeEach(() => {
    vi.restoreAllMocks()
  })
  
  // 4- session mocked  
  const session: Session = {
    expires: '1',
    user: {
      id: 'clgb17vnp000008jjere5g15i',
      username: ''
    }
  }
  
  // 5- init tRPC for test
  const ctx = createInnerTRPCContext({ session })
  const caller = appRouter.createCaller({ ...ctx, prisma: prismaMock }   
   
  describe("procedure 1 - tests", () => {
  // put tests here  
  })  
)
```

## So

In this article we set up our environment to write tests. In the next part of this series we'll write some tests to ***test*** 🤣 our setup. You can check these useful links:

* [The Ultimate Guide to Testing with Prisma](https://www.prisma.io/blog/testing-series-1-8eRB5p0Y8o)
    
* [Sample integration test - tRPC and T3 Stack](https://create.t3.gg/en/usage/trpc#sample-integration-test)
    
* [Example of tests with tRPC - tRPC Github](https://github.com/trpc/trpc/blob/main/examples/next-prisma-starter/src/server/routers/post.test.ts)