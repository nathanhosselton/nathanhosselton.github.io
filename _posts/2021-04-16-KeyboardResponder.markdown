---
layout: post
title:  "Responding to the keyboard"
description: "For example: \"Kick rocks!!\""
date:   2021-04-16
---
When Apple made the iPhone, they went all-in on a bet that the qwerty physical keyboard -- which had just begun to creep into the hands of the casual cellphone wielder -- was only going to survive as long as it took for someone to make the first good touchscreen keyboard. And they figured they'd just hop the line.

Turns out they were right. Also turns out that a keyboard with nowhere else to go but over top of your app's UI is a huge pain in the ass. And not just for developers.

# Dealing with it

Adding insult to injury, since iOS 2, Apple provides only a single, bare-bones mechanism for responding to keyboard events: a generic `Notification` object and accompanying `userInfo` dictionary of the type `[AnyHashable : Any]?`. That's not a fun type to interact with in Swift. And this is what you are expected to use in every view controller of your app.

We're just getting warmed up here though. You've been notified that the keyboard is appearing. Now what? To help you determine how you may need to adjust your UI, the dictionary contains the keyboard's ending frame. That's it. Oh btw the frame is in the coordinate space of the app's window. GLHF.

In a perfectly ideal scenario, your view's coordinate space matches the window's, and all of your constraints are responsive enough to do the right thing when the view shrinks. So you just shrink the view by the height of the keyboard and let autolayout do the rest.

Back in ~~hell~~ reality, you often want to do different things. Some of those things will be on views in nested coordinate spaces. Maybe you don't care if some of your UI gets covered. Maybe that's better than scrunching it all. Now you have to start doing some coordinate translations and really specific math in between, applying these exceptions to your layout without irreparably breaking it. 

Apple doesn't help you here at all, aside from providing a suite of (confusing) instance methods on UIView for coordinate conversion. You get the keyboard's frame. The rest is on you. Perhaps you've already made the big-brain move to using transforms instead of directly manipulating your frames and constraints. But you still have to write all of this code into your view controller, repeated for as many in your app accept user input (perhaps more than one per screen!).

I'm exhausted already.

> **Update:** Apple is providing a new [UIKeyboardLayoutGuide](https://developer.apple.com/videos/play/wwdc2021/10259/) that looks amazing and solves every issue I describe in this post (provided you are able to exclusively target iOS 15).

# Writing a cleaner keyboard API

Abstracting keyboard responses is almost a fool's errand out of the gate. The very nature of the problem resides in the endless ways that you'll find yourself needing to work around it. Every time that I have thought I've finally seen every permutation of keyboard layout, another appears and I'm back to square one. So how can any abstraction hold water?

Our goal isn't going to be to handle every case. We're only going to handle the most common case for which we can trust ourselves to reliably provide benefit. For everything else, we'll simply provide a cleaner API.

{% highlight swift %}
/// A class which abstracts keyboard event subscription and handling on behalf
/// of view controllers, offering conveniences in calculations and automatic 
/// view repositioning.
final class KeyboardResponder {
  /// Represents keyboard presentation events that should be handled.
  enum Event {
    /// A keyboard appearance event. Provides the keyboard's final top position
    /// as well as its animation duration and curve. Also provides a function
    /// which returns the amount that the keyboard will overlap a given view.
    case appear(minY: CGFloat, duration: TimeInterval, curve: UIView.AnimationCurve, overlapOf: (UIView) -> CGFloat)
    /// A keyboard dismissal event. Provides the animation duration and curve.
    case disappear(duration: TimeInterval, curve: UIView.AnimationCurve)
  }
  ...
}
{% endhighlight %}

We're declaring a class because it's going to have instances. We can't get around per-view controller keyboard layout, so we may as well lean into it. If a view controller takes keyboard input, it gets a `KeyboardResponder` instance as a property.

Then we declare the cornerstone of the entire system: the two events and their associated information, necessary for knowing how to react. It's an enum because our API will be providing it through the single callback we'll accept. You could use two separate callbacks, similar to how you use `NotificationCenter` already, but I prefer doing everything in one place. And once we pull out all of the messy boilerplate, your handlers are going to look surprisingly clean. No need to split them in two.

{% highlight swift %}
final class KeyboardResponder {
  ...
  
  /// The designated initializer which immediately subscribes to keyboard notifications.
  init() {
    NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillShow), name: UIResponder.keyboardWillShowNotification, object: nil)
    NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillHide), name: UIResponder.keyboardWillHideNotification, object: nil)
  }

  deinit {
    NotificationCenter.default.removeObserver(self)
  }
}
{% endhighlight %}

Here's another reason we're using classes. You gotta unsubscribe from your notifications. May as well let the wrapper object do that rather than your view controller. We don't accept the event callback in the init because it's going to require a self reference at the call site which doesn't work well with property declarations.

You could make a case for a global/singleton interface where only one subscription happens during your app's lifetime, and all view controllers merely provide their respective handlers. But I dislike globals. For a lot of reasons. But in this specific case, you'd need to start managing collections of handlers, which is a can of worms best left closed whenever possible. An instance retains one handler that dies with it automatically. 

{% highlight swift %}
final class KeyboardResponder {
  ...
  /// The handler provided by our owner to which events should be sent.
  private var eventCallback: (Event) -> Void = { _ in }
  ...
}
{% endhighlight %}

Not exposing this property is mostly an API choice; we'll provide a method that accepts the handler instead soon. In the meantime, it's worth noting here that any new callback received will implicitly override any prior. This is different than how `NotificationCenter` behaves, were you to subscribe multiple times. I don't know why you would do this for keyboard events, and it seems like it would almost always be an error to do so. But if you'd prefer to make the behavior explicit, you can make the property public instead of providing a method.

> This becomes a non-issue when you use a publisher mechanism internally rather than a callback. In that case, your publisher object would be private, and you'd accept arbitrary subscribers via an exposed method. Again, probably not necessary for an instanced object like this, but the gotchas could come from both directions. Pick the one you prefer.

We've left some selector declarations dangling. There are no declared methods `keyboardWillShow` or `keyboardWillHide`. Before we get to those, let's take a minute to preemptively clean up some of the code they'd otherwise have inside them.

{% highlight swift %}
final class KeyboardResponder {
  ...
}

private extension Notification {
  /// The value in this notification's `keyboardAnimationDurationUserInfoKey`,
  /// or the iOS 14 default value if nil.
  var keyboardAnimationDuration: TimeInterval {
    userInfo?[UIResponder.keyboardAnimationDurationUserInfoKey] as? TimeInterval ?? 0.25
  }
  /// The value in this notification's `keyboardAnimationCurveUserInfoKey`,
  /// or the iOS 14 default value if nil.
  var keyboardAnimationCurve: UIView.AnimationCurve {
    UIView.AnimationCurve(rawValue: userInfo?[UIResponder.keyboardAnimationCurveUserInfoKey] as? Int ?? 7) ?? .easeInOut
  }
}
{% endhighlight %}

Technically we're making some operating system assumptions here for the fallback values. But also technically, we should never fail to get the real values out so long as the Notification object is from a keyboard event. This is also why you probably wouldn't want to expose this extension publicly.

Now, lets start writing our notification handler methods.

{% highlight swift %}
final class KeyboardResponder {
  ...
  
  /// Handler for `.keyboardWillHideNotification` events.
  @objc private func keyboardWillHide(_ notification: Notification) {
    eventCallback(.disappear(duration: notification.keyboardAnimationDuration, curve: notification.keyboardAnimationCurve))
  }
  
  ...
}
{% endhighlight %}

Okay that one was too easy. Let's do the other.

{% highlight swift %}
final class KeyboardResponder {
  ...
  
  /// Handler for `.keyboardWillShowNotification` events.
  @objc private func keyboardWillShow(_ notification: Notification) {
    if let frame = notification.userInfo?[UIResponder.keyboardFrameEndUserInfoKey] as? NSValue {
      /// The top of the keyboard in the window.
      let keyboardTop = frame.cgRectValue.minY

      /// A function which calculates the amount that the keyboard will overlap
      /// the provided view.
      let overlap: (UIView) -> CGFloat = { view in
        guard let window = view.window else {
          // If the view has no window it is no longer visible
          return 0.0
        }
        /// The `view`'s superview, if available, to be used for more accurate
        /// point translation.
        let superview = view.superview ?? view
        /// The bottom of the provided view translated to the window.
        let contentBottom = superview.convert(CGPoint(x: 0, y: view.frame.maxY), to: window).y
        // Return the difference between the kb top and content bottom, which
        // is negative when overlapping
        return min(keyboardTop - contentBottom, 0)
      }

      eventCallback(.appear(minY: keyboardTop, duration: notification.keyboardAnimationDuration, curve: notification.keyboardAnimationCurve, overlapOf: overlap))
    }
  }
  
  ...
}
{% endhighlight %}

Yeah that's a bit meatier. So here we're doing some of that legwork that would otherwise be up to the each view controller and is mostly boilerplate. We're digging out the keyboard ending frame's minY, as well as the animation's duration and curve. Pretty much the only things you ever need (unless you're on iPad -- so adjust to your needs).

But we're also providing a function that does some of the boilerplate coordinate conversion and calculation for more fine tuned layout updates. `overlapOf()` takes any view and spits out the amount that the keyboard is overlapping that view, or 0 (we don't want to accidentally move our UI _down_ towards the keyboard).

This is one of the places where we can reasonably and reliably help ourselves. It's not going to solve every layout case, but it can solve a lot. And when it doesn't, we can reach for the `minY` to run our own calculations. We can go another step further now though.

Wait actually hold on, first we need this:

{% highlight swift %}
final class KeyboardResponder {
  ...
  
  /// Subscribes to keyboard notifications with an arbitrary handler function
  /// which receives each `Event` as it occurs.
  ///
  /// Where possible, prefer `automaticallyAdjust(view:padding:)` over this 
  /// method to avoid accidental retain cycles and receive standard behavior
  /// for automatically repositioning the content that should not be covered.
  /// This method is provided for more complex view hierarchies that require
  /// additional logic.
  ///
  /// - Parameters:
  ///   - subscriber: The function to be called when a keyboard event occurs.
  ///   - event: The event that triggered the `callback`, containing relevant 
  ///            keyboard information.
  func subscribe(_ callback: @escaping (_ event: Event) -> Void) {
    eventCallback = callback
  }

  ...
}
{% endhighlight %}

There, now we can actually provide callbacks to our KeyboardResponder. What is `automaticallyAdjust(view:padding:)` though? It's the last piece of the KeyboardResponder and, hopefully, the only method you'll most often need for keyboard adjustments. Which is great, because it doesn't even require a closure:

{% highlight swift %}
final class KeyboardResponder {
  ...
  
  /// Performs a standard translation animation on the provided `view` when a
  /// keyboard event occurs.
  ///
  /// This method adapts the common pattern of repositioning a view just above
  /// the keyboard when it appears, returning it to its original position when
  /// it dismisses. No frame or constraint modifications are applied to the
  /// provided view, only a `y` translation transform.
  ///
  /// When the user moves immediately from one field to the next (thus skipping
  /// willHide notifications) this method will readjust to the largest offset 
  /// required across the displayed keyboard types, remaining at that position
  /// until the next willHide notification.
  ///
  /// - Parameters:
  ///   - view: The view to be adjusted when a keyboard event occurs. 
  ///           Weakly retained.
  ///   - padding: The padding to be added between the bottom of the `view` and
  ///              the top of the keyboard.
  func automaticallyAdjust(_ view: UIView, padding: CGFloat = 10.0) {
    subscribe { [weak view] event in
      guard let view = view else { return }

      switch event {
      case .appear(_, let duration, let curve, let overlapOf):
        /// The amount that the view should be offset to account for the keyboard.
        let offset = overlapOf(view)
        // Ensure there is a required change before wasting an animation dispatch
        guard !offset.isZero else { return }
        UIViewPropertyAnimator(duration: duration, curve: curve) {
          view.transform = view.transform.translatedBy(x: 0, y: offset - padding)
        }.startAnimation()
      case .disappear(let duration, let curve):
        UIViewPropertyAnimator(duration: duration, curve: curve) {
          view.transform = .identity
        }.startAnimation()
      }
    }
  }

}
{% endhighlight %}

We're calling our own `subscribe(…)` method with a callback that applies a y translation to the view we were given, animating it just above the top of the keyboard.

How is this helpful? Why is it broadly applicable? I'll explain. But first, let's see how our newly completed abstraction looks in practice.

# Cleaner, faster keyboard handling

Your view controller's root view has an input field that can end up behind the keyboard of smaller sized devices. You want to adjust your UI so the input field is above the keyboard on the those small screens, but leave everything untouched for bigger screens.

Not a very complicated situation, but not overly contrived either. You'd subscribe to each keyboard event through NotificationCenter, pull the values out of the `userInfo`, calculate whether you need to adjust your content and by how much, then shrink the view, update constraints, or apply a transform accordingly.

Let's assume you go with a transform. It doesn't matter that some of the view's content at the top gets pushed off screen. You just want to ensure the field, as well as any action buttons or instructional text around it, remain above the keyboard.

With our new `KeyboardResponder`, that would look like this:

{% highlight swift %}
final class InputViewController: UIViewController {
  ...
  
  private let keyboardResponder = KeyboardResponder()
  
  override func viewDidLoad() {
    super.viewDidLoad()
    
    keyboardResponder.subscribe { [weak self] (event) in
      guard let self = self else { return }
      
      switch event {
      case .appear(_, let duration, let curve, let overlapOf):
        UIViewPropertyAnimator(duration: duration, curve: curve) {
          self.view.transform = self.view.transform.translatedBy(x: 0, y: overlapOf(self.view))
        }.startAnimation()
      case .disappear(let duration, let curve):
        UIViewPropertyAnimator(duration: duration, curve: curve) {
          self.view.transform = .identity
        }.startAnimation()
      }
    }
  }
  
}
{% endhighlight %}

Or, you could use the convenience method, which does exactly the same thing:

{% highlight swift %}
final class InputViewController: UIViewController {
  ...
  
  private let keyboardResponder = KeyboardResponder()
  
  override func viewDidLoad() {
    super.viewDidLoad()
    
    keyboardResponder.automaticallyAdjust(self.view)
  }
  
}
{% endhighlight %}

If you don't want translate your entire root view, then don't. Enclose the content that you want to adjust in a container view and provide that view object to the method instead. Even from within a different coordinate space, it'll be adjusted appropriately. The implementation of `overlapOf` takes care of it, and everything else will be left alone.

> The reason that we can provide this convenience method and expect it to work reliably is specifically due to how transforms behave. They don't alter any actual frame or constraint values, so nothing can get broken with your layout. That doesn't mean you can't do something that looks bad, just that doing so will never break a constraint or affect any outside calculation based on the now transformed views. Only their rendered positions on screen change.
>
> Try this kind of abstraction with frame or constraint value changes, and you'll be pulling your hair out by its second usage.

# In practice

I wrote `KeyboardResponder` almost a year ago now and have been using it in shipped projects since. Nevertheless, this was an iffy topic to make a post about because of how finicky and risky a keyboard abstraction can be.

Recently I've been doing some prototyping that required quick iteration on how the layout adapted to the keyboard, and I realized there was no way that I could have done it with raw numbers and type casts and optional checks splayed out all over my code. I would have been fighting with myself trying to make any change.

A good abstraction can go a long way. But a bad one can keep you from getting anywhere. There isn't a one-size-fits-all solution. So I try to keep things small and simple. And by implementing things myself, I'm always prepared to drop down below them when it doesen't meet a use case.
