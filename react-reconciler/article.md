# React reconciler in the modern react minimal example to build your own

There are several articles about this issue on the internet. However, most of the start to be out of date. In order to understand this issue better, I decided to

It is important to understand that react-reconciler have no notion of instances or children and has zero dependencies on Dom. This means that in order to use it we need to define what is container, types, and props we are allow to use in our system. Unfortunately, this means that you will need to spend more time to build type system you want. However, it will easily pay off in a long run as it will help you build more reliable system.

So lets start with defining what exactly do we want to build and types. As our goal is to start simple and just build a proof of concept solution we will create a simple type system.

Can we go deeper and integrate it inside another component?

Reconciler allows us to build our own object model without any dependencies or DOM. Even better react-reconciler comes with typescript types where we can define our own model. In our example we will build simple canvas targeted model that will allow simple color and shapes to be drawn on the canvas as well as position. So let's define our type system§
