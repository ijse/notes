> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [dev.to](https://dev.to/femi_dev/how-to-prevent-unnecessary-react-component-re-rendering-3c08)

> Understanding how React renders components is essential for building efficient and performant...

![](https://res.cloudinary.com/practicaldev/image/fetch/s--HjR8beBO--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pkiyhjvwyygl4g3ipsym.png)

Understanding how [React](https://react.dev/) renders components is essential for building efficient and performant applications. When a component’s state or props change, React automatically updates the User Interface(UI) to reflect those changes. As a result, React calls the component's render method again to generate the updated UI representation.

In this article, we will explore three React Hooks and how they prevent unnecessary renderings in React

*   [useMemo](https://react.dev/reference/react/useMemo)
*   [useCallback](https://react.dev/reference/react/useCallback)
*   [useRef](https://react.dev/reference/react/useRef)

These tools allow us to optimize our code by avoiding unnecessary re-renders, improving performance, and storing values efficiently.

By the end of this article, we'll better understand how to make our React applications faster and more responsive using these handy React hooks.

In React, `useMemo` can prevent unnecessary re-renderings and optimize performance.

Let's explore how the `useMemo` hook can prevent unnecessary re-renders in our React components.

By memorizing the result of a function and tracking its dependencies, `useMemo` ensures that the process is recomputed only when necessary.

Consider the following example:  

```
import { useMemo, useState } from 'react';

    function Page() {
      const [count, setCount] = useState(0);
      const [items] = useState(generateItems(300));

      const selectedItem = useMemo(() => items.find((item) => item.id === count), [
        count,
        items,
      ]);

      function generateItems(count) {
        const items = [];
        for (let i = 0; i < count; i++) {
          items.push({
            id: i,
            isSelected: i === count - 1,
          });
        }
        return items;
      }

      return (
        <div class>
          <h1>Count: {count}</h1>
          <h1>Selected Item: {selectedItem?.id}</h1>
          <button onClick={() => setCount(count + 1)}>Increment</button>
        </div>
      );
    }

    export default Page;
```

The code above is a React component called `Page` that uses `useMemo` to optimize the `selectedItem` calculation.

Here's the explanation:

*   The component maintains a state variable `count` using the `useState` hook.
    
*   The `items` state is initialized using the `useState` hook with the result of the `generateItems`function.
    
*   The `selectedItem` is calculated using `useMemo,` which memorizes the result of the `items.find` operation. It only re-calculates when either `count` or `items` change.
    
*   The `generateItems` function generates an array of items based on the given count.
    
*   The component renders the current `**count**` value, the `selectedItem id,` and a button to increment the `count`.
    

Using `useMemo` optimizes performance by memoizing the result of the `items.find` operation. It ensures that the calculation of `selectedItem` is only performed when the dependencies (`count` or `items`) change, preventing unnecessary re-calculations on subsequent renders.

> **Memoization should be employed selectively for computationally intensive operations, as it introduces additional overhead to the rendering process.**

The `useCallback` hook in React allows for the memoization of functions, preventing them from being recreated during each component render. By utilizing `useCallback`. a part is created only once and reused in subsequent renders as long as its dependencies remain unchanged.

Consider the following example:  

```
import React, { useState, useCallback, memo } from 'react';

    const allColors = ['red', 'green', 'blue', 'yellow', 'orange'];

    const shuffle = (array) => {
      const shuffledArray = [...array];
      for (let i = shuffledArray.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [shuffledArray[i], shuffledArray[j]] = [shuffledArray[j], shuffledArray[i]];
      }
      return shuffledArray;
    };

    const Filter = memo(({ onChange }) => {
      console.log('Filter rendered!');

      return (
        <input
          type='text'
          placeholder='Filter colors...'
          onChange={(e) => onChange(e.target.value)}
        />
      );
    });

    function Page() {
      const [colors, setColors] = useState(allColors);
      console.log(colors[0])

      const handleFilter = useCallback((text) => {
        const filteredColors = allColors.filter((color) =>
          color.includes(text.toLowerCase())
        );
        setColors(filteredColors);
      }, [colors]);


      return (
        <div className='tutorial'>
        <div className='align-center mb-2 flex'>
          <button onClick={() => setColors(shuffle(allColors))}>
            Shuffle
          </button>
          <Filter onChange={handleFilter} />
        </div>
        <ul>
          {colors.map((color) => (
            <li key={color}>{color}</li>
          ))}
        </ul>
      </div>
      );
    }

    export default Page;
```

The code above demonstrates a simple color filtering and shuffling functionality in a React component. Let's go through it step by step:

*   The initial array of colors is defined as `allColors`.
    
*   The `shuffle` function takes an array and shuffles its elements randomly. It uses the Fisher-Yates algorithm to achieve shuffling.
    
*   The `Filter` component is a memoized functional component that renders an input element. It receives an onChange prop and triggers the callback function when the input value changes.
    
*   The Page component is the main component that renders the color filtering and shuffling functionality.
    
*   The state variable colors are initialized using the `useState` hook, with the initial value set to `allColors`. It represents the filtered list of colors.
    
*   The `handleFilter` function is created using the `useCallback` hook. It takes a `text` parameter and filters the `allColors` array based on the provided `text.` The filtered colors are then set using the `setColors` function from the `useState` hook. The dependency array `[colors]` ensures that the `handleFilter` function is only recreated if the `colors` state changes, optimizing performance by preventing unnecessary re-renders.
    
*   Inside the `Page` component is a button for shuffling the colors. When the button clicks, it calls the `setColors` function with the shuffled array of `allColors`.
    
*   The `Filter` component is rendered with the `onChange` prop set to the `handleFilter` function.
    
*   Finally, the `colors` array is mapped to render the list of color items as `<li>` elements.
    

The `useCallback` hook is used to memoize the `handleFilter` function, which means the function is only created once and reused on subsequent renders if the dependencies (in this case, the `colors` state) remain the same.

This optimization prevents unnecessary re-renders of child components that receive the `handleFilter` function as a prop, such as the `Filter` component.  
It ensures that the `Filter` component is not re-rendered if the `colors` state hasn't changed, improving performance.

Another approach to enhance performance in React applications and avoid unnecessary re-renders is using the `useRef` hook. Using `useRef`, we can store a mutable value that persists across renders, effectively preventing unnecessary re-renders.

This technique allows us to maintain a reference to a value without triggering component updates when that value changes. By leveraging the mutability of the reference, we can optimize performance in specific scenarios.

Consider the following example:  

```
import React, { useRef, useState } from 'react';

    function App() {
      const [name, setName] = useState('');
      const inputRef = useRef(null);

      function handleClick() {
        inputRef.current.focus();
      }

      return (
        <div>
          <input
            type="text"
            value={name}
            onChange={(e) => setName(e.target.value)}
            ref={inputRef}
          />
          <button onClick={handleClick}>Focus</button>
        </div>
      );
    }
```

The example above has a simple input field and a button. The `useRef` hook creates a ref called `inputRef.` As soon as the button is clicked, the `handleClick` function is called, which focuses on the input element by accessing the `current` property of the `inputRef` ref object. As such, it prevents unnecessary rerendering of the component when the input value changes.

> **To ensure optimal use of `**useRef,**` reserve it solely for mutable values that do not impact the component's rendering. If a mutable value influences the component's rendering, it should be stored within its state instead.**

Throughout this tutorial, we explored the concept of React re-rendering and its potential impact on the performance of our applications. We delved into the optimization techniques that can help mitigate unnecessary re-renders. React offers a variety of hooks that enable us to enhance the performance of our applications. We can effectively store values and functions between renders by leveraging these hooks, significantly boosting React application performance.

[How To Stop React Components From Re-Rendering](https://blog.openreplay.com/how-to-stop-react-components-from-rerendering/)  
[Methods Of Improving And Optimizing Performance In React Apps](https://www.smashingmagazine.com/2020/07/methods-performance-react-apps/)