> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.viget.com](https://www.viget.com/articles/what-even-are-react-server-components/)

> React Server Components blur the line between client rendered and server rendered applications, allow......

About two and half years ago, React gave us our first glance into [React Server Components](https://react.dev/blog/2020/12/21/data-fetching-with-react-server-components). The original goal was to reduce the network waterfall – the series of network calls created by making several server requests sequentially.

![](https://viget.imgix.net/waterfall_2023-04-26-201941_qhad.png?auto=format%2Ccompress&crop=focalpoint&fit=crop&fp-x=0.5&fp-y=0.5&ixlib=php-2.1.1&q=90&w=1280&s=668e314d1766e73746d755d0386c3922)

There have been a lot of different tools and techniques created to avoid these waterfalls of loading. Server side rendering (SSR), parallel fetching, and simple architecture changes are just some of the ways you can avoid this waterfall of server requests. Each of these comes with some drawbacks. SSR blows away client state, making a bad user experience. Parallel fetching can be slow to build. Finally, architecture changes required to avoid the network waterfall tend to make maintaining the application more difficult. Do you _really_ want to pass all of your data through a React Context or Redux?

Server Components aim to be an alternative that creates a good user experience that is fast to build and easy to maintain.

Now, I’m not going to sit here and say that Server Components are the solution to all your React woes. The jury is still out on how useful these fancy new components will be.

Instead, I want to focus on how they work, when to use them, and how to get started with them.

So, What Are React Server Components?
-------------------------------------

React Server Components are React components rendered on the server. Blog post done! I answered the question.

Alright, that’s not a good answer. Let’s dig a bit deeper.

Right now, if you create a React application and bundle it with your favorite build tools, you ship all of the JavaScript the application needs to function to your client. This includes packages to fetch data, format it, and then render it. This can lead to absolutely massive JS bundles for stuff your client probably doesn’t actually need. There are some things you can do to reduce this bundle size, but you're still shipping a lot of JS.

Once your client has received the JS bundles, they can start rendering the app. The app will likely need some data, so now your app makes a request back to your backend server, waits for a response, and then renders out some more data.

![](https://viget.imgix.net/cdn.png?auto=format%2Ccompress&crop=focalpoint&fit=crop&fp-x=0.5&fp-y=0.5&ixlib=php-2.1.1&q=90&w=1280&s=23aa062cbd4931fdc35e7880ca5b6e3a)

The client is already making at least two trips just to finish the initial render, one of which is going to be downloading a large JS bundle from your CDN.

With React Server Components, we reduce this by populating the JS and HTML sent to your client with some initial data.

![](https://viget.imgix.net/server-component.png?auto=format%2Ccompress&crop=focalpoint&fit=crop&fp-x=0.5&fp-y=0.5&ixlib=php-2.1.1&q=90&w=1280&s=aae52de7c71bc0e09844ccc165b8d41e)

And this can nest. If your Server Component would render another component with a data requirement, you only need to make that initial request to load your initial data in the top level component and the data in its child component. Though this will still require multiple requests from your frontend server to your backend server, the client will only need to make one request.

![](https://viget.imgix.net/multiple.png?auto=format%2Ccompress&crop=focalpoint&fit=crop&fp-x=0.5&fp-y=0.5&ixlib=php-2.1.1&q=90&w=1280&s=9557ae4aee44d04e6b2d02299cccf720)

There’s an added benefit here, too. Remember how I said that you are bundling all of the JS your application needs to fetch, format, and render? With server components, you’re sending back minimal JS and a specially formatted response to render some HTML. If the client doesn’t need a package to function, it’s not getting sent over. We’ll look at this a bit closer in the next section.

What do Server Components Look Like in Code  

----------------------------------------------

This is probably my favorite part – they look like plain ole React components! There’s now a bit more instrumentation required to get the server working (this is still something people are figuring out) but as far as the React code is concerned, you just get to write React.

```
// app.js
import { format } from 'date-fns'

export const App = ({userId}) => {
	const user = getUser(userId)
	const lastLogin = format(user.lastLogin)

  return (
		<div>
			<p>{user.name}</p>
			<p>{`Last Login: ${lastLogin}`}</p>
		</div>
	)
}
```

This looks like a simple (and imperfect) client side React component. You’ve got some data fetching, some formatting, and some rendering. But, it’s actually a server component. And all the user is going to get back is a small amount of JS and a specially formatted response to render the HTML.

One of the nice benefits of this trimmed-down response is that you don’t need to send over packages to the client. Look at `date-fns` in the above example. All it's doing is formatting the date correctly. The client doesn’t need all of that JavaScript. All they need is the formatted date, so Server Components are just going to ship that and leave the package at the server.

There are a couple of caveats to this. Notice that I said you’re getting a “specially formatted response” that renders out to HTML. You don’t just get HTML, which has some implications I’ll discuss later in When to Use Server Components.

The other issue is that the client can’t interact with the Server Components. Things like `userState` and `useEffect` get stripped out. The only things that make it to user are JSON serializable structures. Fortunately, React Client Components are serializable. So, if you need interactivity, all you need to do is pop a child Client Component into your server component. Let’s upgrade our previous example to include a simple text field.

```
// app.js
import { format } from 'date-fns'

export const App = ({userId}) => {
	const user = getUser(userId)
	const lastLogin = format(user.lastLogin)

  return (
		<div>
			<p>{user.name}</p>
			<p>{`Last Login: ${lastLogin}`}</p>
			<StatusField userId={userId} />
		</div>
	)
}

// status-field.js
"use client"

export const StatusField = ({userId}) => {
	const [status, setStatus] = React.useState('')

	const onSave = async () => {
		await saveStatus(status)
	}

	return (
		<>
			<label htmlFor="status">Status</label>
			<input id="status" onChange={e => setStatus(e.target.value)} />
			<button onClick={onSave}>Save Status</button>
		</>
	)
}
```

So, what have we done here? We’ve added a React Client Component to a React Server Component so that user can set a simple status. Now, when a user visits our app, they’ll get all of that initial data from the previous example, plus a nice, interactive component.

Do note the use of “use client” at the top of `status-field.js`. That's required to tell React that this component is a Client Component. Outside of that, this will work like any old React component that you know and love.

Two final caveats: Client Components cannot render Server Components and you can’t use class components as Server Components. The former seems reasonable - Client Components are rendered on the client, so they no longer have direct access to the server. The latter might change as these continue to develop.

When to user Server Component
-----------------------------

Server Components seem awesome, but they are not the silver bullet to all of your React woes. When should you be using these fancy components?

There are three use-cases for **NOT** using server components.

### Small and Simple

If your app is really small and simple, you probably won’t benefit much from the extra complications of running a frontend server. Maybe you’ve got an app that doesn’t do much or any data fetching, or maybe you have a React app that is served as a single page in a larger application. In both of these cases, I’d default to a static client-side build and handle any data fetching you might need on the client side.

### Search Engine Optimization

Right now, Server Components don’t return HTML. They return a specially formatted string that React renders out. As a result, they’re not presently the best option for SEO. If SEO is a big concern and you’re building your whole app in React, Server Side Rendering is probably still your best bet, which will give you some of the benefits of Server Components.

### Production Deploys

Despite the fact that these were announced nearly two and a half years ago, there is still a lot to figure out with these components. Right now, they probably shouldn’t be deployed to production on anything but experimental apps. That doesn’t mean you shouldn’t keep your eye on them and start experimenting with them in your stacks. The more people test and toy with Server Components, the sooner we’ll find the bugs and edge cases, and the sooner we’ll be able to use these in production.

Unfortunately, this one in particular probably means you can’t use Server Components for anything serious right now, but that shouldn’t deter you from playing around with them and keeping your eye on them. Hopefully, we’ll be able to confidently deploy these to production in the near future.

### So… When Should You Use Server Components?

Once they're production ready, I think these will be the default React experience for large applications. If you’re already familiar with SSR for your React applications, this will be a nice alternative to it that should cut down on some boilerplate. Right now, you should use Server Components when you have the opportunity to experiment with new technology.

How to Get Started with Server Components
-----------------------------------------

By now, you might be wondering what switch you need to hit in React to get Server Components working. Presently, only [NextJS v13’s App Router](https://beta.nextjs.org/docs/rendering/server-and-client-components) supports Server Components, though you can play with Server Components without NextJS if you [clone the React team’s demo application.](https://github.com/reactjs/server-components-demo)

The main reason for this is that React Server Components need to integrate with something else to work. They need a server and a router - both of which NextJS provides. Right now, the React team is helping other frameworks figure out how to work with Server Components, but we’re likely still a ways off from that. Even still, Server Components are still in beta on NextJS.

Bundling It All Up
------------------

Hopefully, you now have a better understanding of React Server Components. These new React components are easy to write and maintain, and they offer a whole new way to pass data to your application and avoid the dreaded network waterfall. Though we’re still a little ways off from being able to use them in production, I think they’re going to be an invaluable tool in the React developer’s toolkit in the coming years.