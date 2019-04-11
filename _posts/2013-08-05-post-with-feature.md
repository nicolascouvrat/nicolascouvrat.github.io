---
layout: post
title: Simultaneous yet independent gestures in React Native
description: "In my younger and more vulnerable years my father gave me some advice that I’ve been turning over in my mind ever since."
category: articles
tags: [intro, beginner, jekyll, tutorial]
image:
  feature: "https://farm8.staticflickr.com/7014/6840341477_e49612bc59_o_d.jpg"
---

*Disclaimer*: This is a repost from my original (now defunct) blog. The original post date was **2017-11-21**

I've started dealing with React native just recently, as I am trying to build a really simple "remote-controller"
app to drive a robot around. Since I seem to get a deeper understanding of the issues I face when I write them down,
I will do so in a series of posts following my tribulations in the world of React Native. 
Without further ado, let's dive straight into what interests us: gestures and touches in react-native.

## What am I trying to do?

As mentioned above, what I want is to remotely control a little robot that some of my colleagues built a few weeks ago.
We're doing that remote control stuff mostly for fun, and - as far as the application is concerned - requirements are simple: 
emit **two** numbers between -1 and 1 each, representing respectively the forward/back and left/right commands.

I'm therefore absolutely free on the appearance that this controller will have. Heck, I could even have two text inputs and
one button! That wouldn't be fun and intuitive though... And guess what is fun and intuitive as far as controlling a remote
device goes? That's right. *Joysticks*.

## Building a joystick in react native

I've made up my mind rather quickly this time around - perhaps was I influenced by fond memories of playing with RC cars? -
the robot will be controlled by a joystick. It is with this idea in mind that I started looking for documentation on react native,
heading straight for the <a href="https://facebook.github.io/react-native/docs/getting-started.html">official website's tutorial</a>.
Now, my first aim was a circular joystick, similar to <a href="https://github.com/yoannmoinet/nipplejs">this one</a>, 
with a nice vanilla gaming flavor. Time to make a list of what is needed:


* **a draggable element** (the handle) that the user will move around
* **a constraint** to limit the possibilities as far as moving the handle goes - here, a simple fixed radius will suffice
* **a way to know the position relative to a fixed point** (preferably the center)
* and that's it!

Looking around the tutorial and fiddling a little bit with junior high school trigonometry, setting up all of this proved
fairly easy making use of react-native's `PanResponder` and `Animated`. This did not cause much of a problem, so I will not go
over it in details. I managed to create a simple component:

```jsx
<Joystick
    top={Window.height/2}
    left={Window.width/2}
    mainDimension={100}
    onMove={this.onMove.bind(this)}
    onRelease={this.onRelease.bind(this)}
    shape='circle'
/>
```

Rendering like this:

<iframe 
    width="360" height="500" src="/assets/videos/circle_joystick_light.webm"
    frameborder="0" style="display: block; margin:auto;"
></iframe>

And it more or less did the trick. Far from perfect, but still, close to what I wanted. Not bad for just one day of work,
including react-native's installation! 

I immediately proceeded to show it to my colleagues. To my dismay, they did not seem that impressed :( They did like the
joystick idea though, but with a twist: what about making **two** joysticks, one for each command, that could be 
**simultaneously** operated with the phone in landscape mode?

With a component such as the one I built, the only thing left to do was changing the shape and the constraints so that the
handle could only move in one direction. Then, simply add two of them in the same view and boom! Job done. Or so I thought...

## The problem: operate two joysticks *simultaneously*

This is where the fun starts: my two joysticks components did work nicely, but it was impossible to move them together. 
The best I could do was moving them one after the other, implying raising one finger every time before moving the other one.
Way to throw any intuitive aspect out of the window... Indeed, no matter how hard I tried, as long as I kept a finger pressed
against on joystick, that first joystick seemed to "take control", disabling the second one, and vice versa. 

### Exit `PanResponder`...

Unlike what I thought at first, this is in fact not an unexpected behavior at all: the `PanResponder`'s `gestureState` in fact
*combines every touch together in one gesture*. 
As written in the [documentation](https://facebook.github.io/react-native/docs/panresponder): 


> `PanResponder` reconciles several touches into a single gesture. It makes single-touch gestures resilient to extra touches,
> and can be used to recognize simple multi-touch gestures.

While this probably allows for an easy managing of fairly complex moves, it is the exact opposite of what we want to achieve here,
i.e. treat the two touches separately :/ After a bit of research though, I found that the `nativeEvent` that is returned along the
`gestureState` as argument to the `PanHandler`'s callbacks did contain an array of the different touches on the screen, namely in
`evt.nativeEvent.touches`.

```jsx
componentWillMount: function() {
    this._panResponder = PanResponder.create({
        // Assuming the right onStartShouldSet... calls
        onPanResponderMove: (evt, gestureState) => {
            // evt.nativeEvent.touches contains a list of all touches
        },
    });
},

render: function() {
    return (
        <View {...this._panResponder.panHandlers} />
    );
},
```

At this point, you might think something like: "let's just throw that high level API away, go one layer deeper and simply intercept
that native event to build on it!". This is exactly was I tried too.

### ...Hello standard `Views`!

This is a little bit hidden in the docs ([that page]("https://facebook.github.io/react-native/docs/gesture-responder-system.html") 
talks about it without much details and without any code), but you can actually access touch events directly using the `props` of 
just about any `View`. It goes like this:

```jsx
export default class App extends React.Component {
    /* assuming imports are done */
    render() {
        return (
            <View
                onStartShouldSetResponder={() => true}
                onResponderStart={(evt) => console.log("I have started!")}
                // onResponderMove={}
                // etc.
            >
                <Text>Click me!</Text>
            </View>
        )
    }
}
```

This works as intended: touching the text will indeed log a message in the console. I thus reworked my `Joystick` component to 
integrate event response this way. I ended up with something like:

```jsx
export default class App extends React.Component {
    /* assuming imports are done */
    render() {
        return (
            <View>
                <Joystick name="leftRight" onMove={this.onLeftRightMove}/>
                <Joystick name="upDown" onMove={this.onUpDownMove} />
            </View>
        )
    }
}
```

But it did not work any better, and the results were exactly the same as when using a `PanResponder`. It is at this point that 
I realized that I'd better get seriously up to speed with how event bubbling and responders work in React Native if I wanted 
to solve this problem. What I naively thought of as a trivial problem was revealing more complex than first anticipated.

To be continued...
