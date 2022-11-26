# Make React prop depends to other props

I'll share here a trick I used recently (after some research 🤣) to make the existence of a prop (react) value depend
on another using TypeScript.

## The context

We have a `Menu` component that can have as children `MenuItem` or a custom react component.
Let's say the signature of `MenuItem` is like this :

```js
// MenuItem.tsx
export type MenuItemProps = {
... // some props
	label: string
}

export const MenuItem: React.FC<MenuItemProps> = ({ label }) => {
  return (
    <div>
      <h1>{label}</h1>
    </div>
  )
}
```

And the `Menu` component takes a list of `MenuItem` props and displays each `MenuItem` with its props:

```js
// Menu.tsx
export type MenuProps = {
	... // some props
	menuItemList: MenuItemProps[]
}

export const Menu: React.FC<MenuProps> = ({menuItemList}) => {
	return (
		<div>
			<h1>Menu</h1>
				<div>
					{
						menuItemList.map((menuItem, index) => <MenuItem key={index} {...menuItem} />)
					}
				</div>
		</div>
	)
}

```

## The problem

Now, I want that the user can display a custom component that he/she builds. Let's say a div with icon. We need `children` as props to `Menu` like this:

```js
export type MenuProps = {
	... // some props
	menuItemList: MenuItemProps[]
	children: React.ReactNode
}
```

But the problem: where do I put the children prop?

- Before or after the list of MenuItem ?
- Between the list of MenuItem ? (More pains for nothing)

Yes, because there will be some _devs_ who will provide these two props even if there are optionals.

I finally decide: **Only one of them will be available to use** so **"If user/dev set menuItemList array he/she cannot set children prop"**.

## The solutions

We can try to define these two optional props and then check their existence and raise an error if both are available.
I did not like this and did not try to implement it. The solution I used has the same logic but cleaner.

### TypeScript + never + union

_If like me, you are wondering where we can use **never**, be patient, here you'll use it 🤣_

The final type will be in two parts.

#### 1st step

```js
export type MenusProps = {
  children?: null | undefined,
  menuItemList?: MenuItemProps[],
}
```

We make all of them optionals. Since we want to base our condition to the availability of `children`, we make it optional as well.

#### 2nd step

```js
export type MenusProps = {
  children: React.ReactNode,
  menuItemList?: never,
}
```

With this type, `MenuProps` can have `children` as prop but not `menuItemList`.

#### 3rd step

```js
export type MenusProps =
  | {
      children?: null | undefined,
      menuItemList?: MenuItemProps[],
    }
  | {
      children: React.ReactNode,
      menuItemList?: never,
    }
```

We now indicate that `MenuProps` is an union of the above types. So it can have one of them but not both.
In this case, if the user of `Menu` component provides `children` as prop, he/she cannot provides again `menuItemList` as
prop and if he/she can pass nothing.

#### Bonus

The final type is

```js
export type MenusProps =
  | {
      children?: null | undefined,
      menuItemList?: MenuItemProps[],
    }
  | {
      children: Exclude<React.ReactNode, null | undefined>,
      menuItemList?: never,
    }
```

A `React.ReactNode` can be `null | undefined` and so we have two optionals properties in the second union which is not the desired behavior. So we Exclude
some values for children in this case and so we can write

```js
// Menu.tsx
export type MenusProps =
  | {
      children?: null | undefined;
      menuItemList?: MenuItemProps[];
    }
  | {
      children: Exclude<React.ReactNode, null | undefined>;
      menuItemList?: never;
}


export const Menu: React.FC<MenuProps> = ({ menuItemList }) => {
  return (
    <div>
      <h1>Menu</h1>
      <div>

				{menuItemList && {menuItemList.map((menuItem, index) => (
          <MenuItem key={index} {...menuItem} />
        ))}}

				{children}
      </div>
    </div>
  )
}

```

Now, if you try to give a children and a menuItemList to Menu, TypeScript will display a type incompatibility warning.

## So

This solution may not be the best but it helps us to make some props depends to another and TypeScript can raise a
warning if user didn't
respect this rule.

I'm sharing a few tips that I use in my daily tasks, and I hope you have others that you want to share with us.
I'm open to tips, and my social accounts (below) are there for that.

_So can you smell what TawalMc is cooking?_
