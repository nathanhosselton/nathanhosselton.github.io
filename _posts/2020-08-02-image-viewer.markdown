---
layout: post
title: "A simple image viewer"
description: "Simple in quotes."
date: 2020-08-02
---
This week I needed to implement full screen image viewing/zooming, Photos.app style. It didn't need to show a gallery or even function as a carousel. I just wanted it to support double tap to zoom, pinch to zoom, and swipe to dismiss. There are doubtless innumerable libraries that provide this behavior and more since it's so commonplace. Honestly, I'm surprised Apple doesn't offer a proper UIKit class for it yet. But they still provide you something that gets you most of the way there: UIScrollView.

Scroll views naturally support panning as well as zooming. And since this is the foundation of what's needed for a full screen image viewer, I started there to see how far I could get. 

_Spoiler: I got all of the way there._

# Declaration and init

A view controller like this doesn't make sense in Storyboard, so we configure it for programmatic initialization.

{% highlight swift %}
/// A lightweight view controller for displaying single image content that is gesture responsive.
///
/// Initialize and present this controller to display an image full-screen on a black backdrop with
/// expected zooming functionality via pinch gestures and double taps. Dismisses automatically
/// when the user flicks an image at the scroll edges.
final class ImageViewerViewController: UIViewController, UIScrollViewDelegate {
  /// The image being displayed by this controller.
  let image: UIImage

  /// The designated initializer. Prepares the controller for modal presentation using the provided image.
  /// - parameter image: The image to be displayed in the controller's view. Expected to be larger
  ///   than the view in at least one dimension, else it is not zoomable.
  required init(displaying image: UIImage) {
    self.image = image
    super.init(nibName: nil, bundle: nil)
    modalPresentationStyle = .overFullScreen
    modalTransitionStyle = .crossDissolve
  }

  /// The image view which contains and displays the `image` inside the `scrollView`.
  private var imageView: UIImageView!
  
  ...
}
{% endhighlight %}

This establishes our public API and our primary view components, save for the scroll view which is declared as a lazy var further down to hide its configuration details away from where our core functionality will reside:

{% highlight swift %}
/// The scroll view managing zoom and dismissal gestures.
private lazy var scrollView: UIScrollView = {
  let lv = UIScrollView(frame: view.bounds)
  lv.delegate = self
  lv.scrollsToTop = false
  lv.alwaysBounceVertical = true
  lv.alwaysBounceHorizontal = true
  lv.showsVerticalScrollIndicator = false
  lv.showsHorizontalScrollIndicator = false
  lv.contentInsetAdjustmentBehavior = .never
  lv.translatesAutoresizingMaskIntoConstraints = false
  return lv
}()
{% endhighlight %}

The oddity here is probably the disabling of `contentInsetAdjustmentBehavior`, which keeps UIKit from inserting its standard content insets for safe areas. You may want this, but I didn't, and disabling it also prevented some extra arithmetic in calculations we'll be performing. It wasn't complicated, just noisy. The rest of these configurations are what you'd probably expect given our functionality.

# Layout

Most of our busy work takes place in viewDidLoad:

{% highlight swift %}
override func viewDidLoad() {
  super.viewDidLoad()

  view.backgroundColor = .black

  imageView = UIImageView(image: image)
  imageView.contentMode = .scaleAspectFit
  imageView.translatesAutoresizingMaskIntoConstraints = false

  view.addSubview(scrollView)
  scrollView.addSubview(imageView)

  NSLayoutConstraint.activate([
    // Pin the scrollView to the view
    scrollView.topAnchor.constraint(equalTo: view.topAnchor),
    scrollView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    scrollView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
    scrollView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    // Pin the imageView to the scrollView's content edges
    imageView.topAnchor.constraint(equalTo: scrollView.contentLayoutGuide.topAnchor),
    imageView.trailingAnchor.constraint(equalTo: scrollView.contentLayoutGuide.trailingAnchor),
    imageView.bottomAnchor.constraint(equalTo: scrollView.contentLayoutGuide.bottomAnchor),
    imageView.leadingAnchor.constraint(equalTo: scrollView.contentLayoutGuide.leadingAnchor)
  ])

  imageView.isUserInteractionEnabled = true
  imageView.addGestureRecognizer(imageDoubleTapGestureRecognizer)
}
{% endhighlight %}

Here we ready our `imageView` and `scrollView` by adding them to the view hierarchy with constraints. The `scrollView` gets pinned to all edges of our view, while the `imageView` gets pinned to the _content_ edges of the `scrollView`, so to allow it to extend beyond our view's bounds when resized. Finally we add the double-tap gesture recognizer, which is just a standard UITapGestureRecognizer with `numberOfTapsRequired = 2` configured in a lazy var.

Just before the view appears, we set the zoom configurations for the `scrollView` based on the image size:

{% highlight swift %}
override func viewWillAppear(_ animated: Bool) {
  super.viewWillAppear(animated)
  // Set min zoom at the point where the image just fits within the view
  scrollView.minimumZoomScale = min(scrollView.bounds.width / imageView.bounds.width, 1)
  // Set max zoom at the point where the image just fills the scroll content vertically
  scrollView.maximumZoomScale = min(scrollView.bounds.height / imageView.bounds.height, 1)
  // Begin at min zoom
  scrollView.setZoomScale(scrollView.minimumZoomScale, animated: false)
}

func viewForZooming(in scrollView: UIScrollView) -> UIView? {
  return imageView
}
{% endhighlight %}

We want the displayed image to never zoom out any further than the size of our view, because that would be pointless. But we also want it to never zoom in any further than the height of the view. This allows the image to always bounce on the vertical scroll axis, which is how we're going to enable dismissal. If we let the user zoom in further, we would create a condition where they would be unable to dismiss the image via the expected gesture, or we would need to make some concession on the vertical scroll experience where _sometimes_ scrolling without a bounce does dismiss (e.g. via a high enough velocity).

That wouldn't necessarily be a bad thing, but it would be more complicated, and I didn't deem it necessary for my use case. My users aren't really going to need to zoom in further than the height of their screen, and if they want to, they can still achieve that by holding a pinch gesture in place. When they let go, the zoom will bounce back and they'll be able to dismiss.

> Note: These zoom scales assume that the image's height will never be the height of the view, which would prevent zooming. They also assume that the view is never in landscape orientation. Both of these were fine for my situation, because I knew both will always be true for the foreseeable future.
>
> This is the advantage of writing your own solutions; you have only the exact amount of code that you need and no more. And if you ever do need more, you can add to it, because you own it.

# Gestures

Handling double-taps is easy enough. If we're not yet zoomed in all the way, zoom in. Otherwise, zoom out.

{% highlight swift %}
/// Handler for double tap gesture events from the `imageDoubleTapGestureRecognizer` on the `imageView`.
@objc private func onImageDoubleTap(_ gesture: UITapGestureRecognizer) {
  guard gesture.state == .ended else { return }
  if scrollView.zoomScale < scrollView.maximumZoomScale {
    // Zoom in when not yet fully zoomed in
    scrollView.setZoomScale(scrollView.maximumZoomScale, animated: true)
  } else {
    // Zoom out
    scrollView.setZoomScale(scrollView.minimumZoomScale, animated: true)
  }
}
{% endhighlight %}

Things get a bit more complex inside of our swipe-to-dismiss logic. We want to meet three conditions before we dismiss:

1. The scroll view is bouncing at its content edge
2. The bounce is in the direction with the highest velocity
3. The velocity is significant

The first one is a given; it's the desired functionality. Why isn't detecting bounce by itself sufficient?

Well, technically it is. But I don't think it would feel polished. The tiniest bounce could trigger a dismiss, including in a direction the user wasn't intentionally trying to scroll. Remember, the image will always bounce vertically. So if a user is panning to the left or right, they'll trigger a dismiss if their swipe isn't perfectly horizontal. We also don't want to dismiss if there is no velocity, which is the case when the user drags, stops, then releases. We only want to dismiss when the user is already at a scroll edge and swipes primarily in the bouncing direction with ending velocity (inertia).

I've probably made this sound really complicated, but the resulting logic is pretty much three lines:

{% highlight swift %}
func scrollViewWillEndDragging(_ scrollView: UIScrollView, withVelocity velocity: CGPoint, targetContentOffset _: UnsafeMutablePointer<CGPoint>) {
  // - Swipe-to-dismiss
  /// The absolute velocity values of the drag event in each direction.
  let absV: (x: CGFloat, y: CGFloat) = (abs(velocity.x), abs(velocity.y))
  /// Whether or not the scroll view is bouncing (over scrolling) on the x axis.
  let horizontallyBouncing: Bool = absV.x > absV.y && (scrollView.contentOffset.x < 0 || scrollView.contentOffset.x > scrollView.contentSize.width - scrollView.bounds.width)
  /// Whether or not the scroll view is bouncing (over scrolling) on the y axis. This is simplified
  /// because our scroll content (the image) never exceeds the vertical size of our scroll bounds.
  let verticallyBouncing: Bool = absV.y > absV.x && abs(scrollView.contentOffset.y) > 0
  // Dismiss when we're bouncing with a minimum (small) velocity.
  if max(absV.x, absV.y) > 1, horizontallyBouncing || verticallyBouncing {
    self.dismiss(animated: true)
  }
}
{% endhighlight %}

> Note: This is one point where we would need to factor in the safe area insets had we not removed them.

This detects all three of our required conditions and is self contained within the scroll view's delegate method; there is no outside state or helper functions to maintain.

# Centering

If you run this now, you'll find that it works, but the image is always at the top of the screen. Turns out that there isn't a built-in way to keep UIScrollView content centered when zoomed out. It's not too bad to math yourself though.

{% highlight swift %}
func scrollViewDidZoom(_ scrollView: UIScrollView) {
  /// The amount to inset the scroll content from the top to keep it centered in the view.
  let inset: CGFloat = (scrollView.bounds.height - imageView.bounds.height * scrollView.zoomScale) / 2
  /// The amount to subtract from the top inset value to keep the content visually centered.
  let visualCenteringOffset = scrollView.bounds.height * 0.02
  // Set the top content inset relative to the current zoom level
  scrollView.contentInset = .top(max(inset - visualCenteringOffset, 0))
}
{% endhighlight %}

Now the scroll content will always offset from the top relative to the current zoom level. This looked a little lower than center to me, so I subtract the `visualCenteringOffset` value to keep the image where my eyes wanted it to be. We make sure the inset is never negative, which is possible due to the admittedly blunt visual centering solution. There's a more accurate formula to apply here, but I moved on.

Now when we run it, everything feels pretty dang good. Not as polished as Photos.app, but maybe like, 80% of the way there? And easily with less than a third of the code and complexity you'd get from using a library. There is no state or external configuration in this controller aside from the image itself. You simply won't find that in any library that does this for you.

# No transitions?

Right now the most notable bit of missing polish is the lack of custom transitions. We've got a simple cross fade which doesn't look bad, but the expected transition for this interaction animates the image from its position in the source view controller to its final zoom/position in the viewer controller, then reverses out from the swipe gesture when dismissing. This isn't difficult to implement with `UIViewControllerAnimatedTransitioning`, but it also wasn't a priority during implementation relative to the time investment. This is certainly the point where you could end up winning in the short term by adding a dependency.

If image viewing were a primary aspect of your app's experience however, I think the extra time from doing it yourself right now would be justifiable. This wasn't my situation, so I'll probably revisit it during some downtime in the future, perhaps when the feature containing the functionality is expanded. For now, this is more than sufficient, and easy to revisit and extend. It is the right amount of code.
