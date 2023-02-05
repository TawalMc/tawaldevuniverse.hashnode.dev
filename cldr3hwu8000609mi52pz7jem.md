# Building a side project, a SaaS product based on T3 Stack

*Before starting I want to thank Hashnode and point out that my article for* [*Dev Retro 2022*](https://tawaldevuniverse.hashnode.dev/dev-retro-2022-beginning-of-my-professional-career) *was nominated among* [*the winners*](https://townhall.hashnode.com/dev-retro-2022-winners)*, very happy.*

## The project

As the title said, I'm building a SaaS as a side project (maybe I can call it SaaSaSP 🤣). The idea is to allow people to build exercises with questions and responses on the platform and share this with other people who can practice. This platform will be used on one hand by students during revision periods and on another hand, by teachers who can share homework with their students.

I only work evenings and Saturdays on the project (because I have a full-time job). And currently, I can say that I am at 20% of the first version of the product.

## The Design

I don't have a design so I went to websites with which I like the design and I try to reproduce them for the pages I need. For example:

* Sign in: I'm stunned by the login page of Adobe so I tend to reproduce it
    
    ![Adobe login page](https://cdn.hashnode.com/res/hashnode/image/upload/v1675030226869/3b1131cb-e879-4f63-b38d-3879aaf85093.jpeg align="center")
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1675030348467/bda72fed-e10b-48f0-93c6-cf0ff1ca0b30.png align="center")
    
* Dashboard: I like vercel and sanity dashboards
    
    ![Sanity dashboard](https://cdn.hashnode.com/res/hashnode/image/upload/v1675030487545/3bf61160-af2c-4276-b582-3cc80e217700.jpeg align="center")
    
    ![Vercel dashboard](https://cdn.hashnode.com/res/hashnode/image/upload/v1675030570222/c75bb0c9-7c5c-402a-9e04-801d59deb51e.jpeg align="center")
    

I don't finish to build the dashboard so I haven't concrete pictures.

## The Stack and why this?

Yes, the big part. To make my choice here are the criteria I followed:

1. I want to use tools with which I have some prior experiences to code the application quickly and which are popular and stable so that if I encounter a bug I can find resources to resolve it. In my case, I use only: (TS)React/Next.js. So the frontend lib/framework must be one of these two.
    
2. I don't want to deal too much with CSS and I hope the UI lib will have a lot of basic components with a good styling approach. Here I was thinking about 3 UI lib based on React (in order of consideration given):
    
    * [React Spectrum](https://react-spectrum.adobe.com/) & [React Aria](https://react-spectrum.adobe.com/react-aria/index.html): A collection of lib and tools from Adobe developers. They implement Adobe design systems which I find nice. It has many components including a date range picker, drags and drops, and useful resources. What I like about this lib are the efforts the maintainers put into accessibility and [WAI-ARIA](https://www.w3.org/TR/wai-aria-practices-1.2/). Also, they have many hooks like [useDateRangePicker](https://react-spectrum.adobe.com/react-aria/useDateRangePicker.html) that you can use to build a custom component, here is a date range picker 😛.
        
    * [Mantine](https://mantine.dev/): A relatively new lib with a bunch of pre-built advanced components like a search bar with a filter (more featured than what I built [here](https://hashnode.com/post/clb7wly1g000a08l123l793wf)). Its strength lies in the ease with which you can customize its components with a better developer experience than with Material UI.
        
    * [Next UI](https://nextui.org/): A new lib too, with a beautiful design around gradients and based on...React Aria ( 😎 Yes, the same as I mentioned above)
        
    * [Chakra UI](https://chakra-ui.com/): One of my favourite UI components libs. Simple, easy to use with a very well-designed approach and built by a Nigerian (😎 a border country to Benin, my country). For my side projects, I use Chakra UI. It's very well documented and even if It doesn't have a lot of pre-built components like a date range picker, I was comfortable using it.
        
        **So my choice?**: Chakra, why?
        
        I found NextUI css API a little complicated (it supports styled-components), with React Spectrum the [button's](https://react-spectrum.adobe.com/react-spectrum/Button.html) onClick prop becomes onPress (can you imagine ? I don't know the reason) and I don't like Mantine basic style (😄 someone who is not good in CSS criticizes a UI components libs). But Chakra, everything seems easy, simple and intuitive: ***\_hover*** is a prop that you can provide with style when the mouse hover your component (really intuitive, isn't it ?)
        
        **NB:** I am not a fan of Material UI, customization is an obstacle course where you need to check the css class name of children in chrome inspector to overrides their styles
        
3. I want a real full-stack React/Next framework that'll allow me to write JS code in both the server and front end. If it can include a config for database and authentication it'll be **very good**. And no, I don't have the time and energy to go with API first architecture so I cannot have the Node server separate from the frontend. So I have 3 choices:
    
    * [Redwood.js](https://redwoodjs.com/): It's a good candidate and I like the name and the full battery included: Routing, GraphQL, TypeScript, Prisma, and Authentication. It's good but I need to jump to GraphQL and they have their routing system (It'll take me some time to be ready, the time I don't have 😄). But I want to test it one day
        
    * [Blitz.js](https://blitzjs.com/): Good too, built on top of...Next.js. It comes with authentication, is database agnostic and has a CLI tool: *blitz.* Two things I like about it: [Route manifest](https://blitzjs.com/docs/route-manifest) which allows you to refer to a page by *name through an object* instead of hand-coded string (useful with dynamic route) and their use of **Zod** to validate input in form (it's the first time I hear about Zod). But they were talking about pivoting to a framework agnostic and they were relatively new, so I didn't pick them
        
    * [T3 Stack](https://create.t3.gg/): On the home page you can read: **The best way to start a full-stack, typesafe Next.js app.** So I choose it 😄. Ok I'll explain. T3 Stack doesn't bring anything new but it puts together well know/established tools and libs to let you focus on your business logic. Firstly the core of this stack is also TypeScrpt/Next.js. Secondly, for authentication, they use [NextAuth.js](https://next-auth.js.org/), for the API they use tRPC which allows you to share your types between client and server code (input validations, return types) by writing typed procedures on the server side. tRPC has a plugin for the well-known [React Query](https://react-query-v3.tanstack.com/) that T3 Stack includes for you. For the database, this stack uses the popular ORM [Prisma](https://www.prisma.io/) so you can connect your application to all databases (PostgreSQL in my case) supported by Prisma. When configuring it asks to install tailwindcss for styling. Something great is that: you can install what you need only.
        
        You can understand that I chose T3 Stack. **At the end of the project, I'll share my feelings after using this stack.**
        

There is another choice I need to do: the deployment platform. As I have't finished or have a eta version yet, I haven't made my choice yet. But I'm thinking between: [fly.io](https://fly.io/), [render](https://render.com/) and [vercel](https://vercel.com/).

We are athe end of this article, in my next article I'll share some tricks that I'm using with this stack.