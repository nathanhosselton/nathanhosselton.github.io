---
layout: post
title:  "FutureImage"
description: "Just like self confidence, pretending you have an image is almost as good as actually having it."
date:   2020-08-26
---
Image fetching is a common 'gotcha' for new learners. I've used it in the classroom numerous times. It's fun. See, first you surprise them with the realization that their JSON response doesn't contain the image data, just a url _to_ the image data. Then you let them figure out what to do with that.

Inevitably, they download the images inside their repeating content's display, or worse, try to save the images to a separate collection that is never in sync with the source data collection. Either way, their code is twisted up every which way. You chuckle to yourself, remembering how the developer you learned from did this to you, savoring the moment as they once did.

Then you have to actually fix it. If there isn't already a custom model, you make one, and you either put it in charge of the download task or have some manager object do it and assign the image back to the model when it's done. But that's only half the battle. The other half is to somehow notify the UI that a downloaded image is ready for display.

Then you feel bad. Not because you put them through hell to start, but because the _real_ hell is actually in the solution.

# FutureImage

Let's do this better. Let's just completely ignore the download task altogether.

We'll accomplish this by subclassing `UIImage`. Why?

{% highlight swift %}
/// A static object representing a future image to be retrieved asynchronously
/// from a remote url.
///
/// Because class instances cannot assign to self, objects of this type will 
/// never become the future image; they act only as a proxy for that image.
/// The extension `var futureImage` on `UIImageView` manages this relationship 
/// automatically. This object's subclassing of `UIImage` is only to afford
/// some conveniences in that relationship.
///
/// Instances of this object are safe to utilize in repeating content (such as
/// a table view) when retained as part of that content's data source, as 
/// changes on the corresponding image views will automatically reflect without
/// restarting the internal image fetch operation, and reused views will be
/// updated immediately regardless of whether the future image has become 
/// available.
///
/// This object also supports a convenient pattern whereby instance vars may 
/// be declared lazy on a Decodable model object containing the resource url.
/// In doing so, the image fetch does not begin until the future's first usage, 
/// preventing greedy loading. Afterward, the model then becomes the cache for
/// the resource, able to be passed between screens without recipients'
/// awareness of whether the image is already available or not. 
///
/// Thus, the image's fetching and usage are entirely decoupled.
final class FutureImage: UIImage {
  ...
}
{% endhighlight %}

Well that sounds great, but there's nothing here. And what's this extension on UIImageView it's talking about? Seems like that's where the magic is.

We'll have an initializer and some logic in this class eventually, but indeed, the show stealer is the extension.

{% highlight swift %}
/// An interface for a view which accepts and responds to changes 
/// from a `FutureImage`.
private protocol FutureImageBinding: UIView {
  /// A reference to a future image to be displayed by this view.
  var futureImage: FutureImage? { get set }
  /// Notifies the view that an image is ready for display from the future.
  func imageAvailable(_ image: UIImage, in future: FutureImage)
}

extension UIImageView: FutureImageBinding {
  /// A reference to a future image to be displayed by this view.
  ///
  /// If the image within the future is already available, assigning to this
  /// property will immediately display it. Otherwise, the image will be 
  /// automatically displayed once it becomes available.
  ///
  /// Assigning a new value or `nil` to this property are both sufficient for 
  /// terminating the connection between this view and any prior future.
  var futureImage: FutureImage? {
    set {
      // Disconnect from the current future
      futureImage?.unbind(self)
      // Set the new future (or nil)
      image = newValue
      // Bind to the new future's updates
      if let newFuture = newValue { newFuture.view = self }
    }
    get { image as? FutureImage }
  }

  fileprivate func imageAvailable(_ image: UIImage, in future: FutureImage) {
    // Ensure we're not receiving an update from a dangling future
    guard futureImage != nil, future === futureImage else {
      // This should be made impossible in futureImage.set, but it's a trival
      // check to perform for the added safety.
      future.unbind(self)
      return
    }

    // - Note: Here we necessarily lose reference to our future in order to
    // assign the real image that we've been waiting for. So we make this
    // explicit and unbind ourselves.
    futureImage = nil
    self.image = image
  }
}
{% endhighlight %}

What's going on here amounts to a (leaky) delegate relationship. But since UIImageView is the only class in UIKit that we use to display images, there's no need for us to worry our consumers about it. And so we hide it away for simplicity. What's left is merely the FutureImage instance, and the `futureImage` property on UIImageView to which it is assigned.

A consumer initializes a future with a url to the remote image, then assigns that future to the image view that will display that image. That's it. Everything else is managed behind the scenes without a care for what's happening on the outside, including reassignment of the future to another image view, which is common in repeating content.

This is where the initializer and some of the private members of `FutureImage` come in.

{% highlight swift %}
final class FutureImage: UIImage {
  /// Initializes an object for the image resource at the provided url, which 
  /// begins fetching immediately.
  ///
  /// You do not manage this object after initialization. Instead, you are 
  /// expected to eventually provide this to an instance of `UIImageView` via 
  /// its `futureImage` member, which will receive and self-update with the 
  /// future image once it becomes available.
  ///
  /// An instance may be rebound to a view any number of times without 
  /// consideration for the underlying download task. When the image becomes 
  /// available, the currently connected view will receive it. If an image is
  /// already available, the newly connected view will immediately receive it.
  ///
  /// By default, this object is an empty image. Alternatively, you may provide
  /// an asset name to be used until the future image becomes available.
  ///
  /// - Parameters:
  ///   - url: The remote location of the image to be fetched.
  ///   - named: The name of the bundled image asset to be used until the
  ///     future image is fetched.
  convenience init(at url: URL, placeholder named: String? = nil) {
    if let placeholder = named {
      self.init(imageLiteralResourceName: placeholder)
    } else {
      self.init()
    }

    self.url = url

    fetch()
  }
  
  /// Executes the download task for the image resource at the remote url 
  /// provided during initialization and sets the `futureImage` once complete, 
  /// which in turn notifies any bound `view`. Retries on failure.
  fileprivate func fetch() {
    ...
  }
  
  /// Safely unbinds the provided view from this object's `view`.
  fileprivate func unbind(_ view: FutureImageBinding) {
    // Only unbind if the requesting object is our current view
    if view === self.view {
      self.view = nil
    }
  }

  /// The view to receive the `futureImage` once it becomes available.
  ///
  /// A future may be rebound to any number of views and each view will be 
  /// provided the fetched image, immediately when already available.
  fileprivate weak var view: FutureImageBinding? {
    didSet {
      if let image = futureImage { view?.imageAvailable(image, in: self) }
    }
  }

  /// The image fetched from the `url`, once available.
  private var futureImage: UIImage? {
    didSet {
      if let image = futureImage { view?.imageAvailable(image, in: self) }
    }
  }
  
  /// The remote location of the image being fetched by this object.
  private var url: URL!
}
{% endhighlight %}

As was indicated, there's almost nothing here. Just the fetch task (which contains nothing of consequence) and some `didSet` handlers that enable the delegate mechanisms. It's lean on logic and API, but big on code impact. The best kind of abstraction.

# Ignore The Download™️

Let's start with a typical Decodable model:

{% highlight swift %}
struct StoreItem: Decodable {
  let title: String
  let description: String
  let cost: Double
  let primaryImageUrl: URL
}
{% endhighlight %}

At this point, you'd either need to custom implement the initializer and immediately start the download task for the image, or wait until some later point when it's about to be displayed, but either way provide a mechanism for broadcasting to the UI when the download is complete.

Let's use a future instead:

{% highlight swift %}
struct StoreItem: Decodable {
  ...
  /// The remote location of the item's primary image.
  private let primaryImageUrl: URL
  /// The future for the item's primary image.
  lazy var primaryImageFuture: FutureImage = { 
    FutureImage(at: primaryImageUrl, placeholder: "store_placeholder") 
  }()
}
{% endhighlight %}

By declaring the future lazy, we delay the download task until the first time it's used. So let's use it:

{% highlight swift %}
// In StorefrontViewController.collectionView(:cellForItemAt:)
let item = storeItems[indexPath.item]
cell.titleLabel.text = item.title
//etc…
cell.primaryImageView.futureImage = item.primaryImageFuture
{% endhighlight %}

What happens next? Well, if this is the first time the user has scrolled to this item, the image view gets the placeholder and the image download starts. When the download is finished, the image view updates.

If the image was already downloaded, the image view just gets the real image immediately. You don't care either way, and neither does the image view. If the user scrolls before the download has finished, the recycled image view would get the next future and all the same things happen.

You're passing around an actual image object. And you can use it to behave as if you already have the final image. It's fun to pretend, isn't it?

Okay but now what happens when the user taps on that item and we need to display the detail page for it?

{% highlight swift %}
// In StorefrontViewController.prepareForSegue
storeDetailViewController.displayItem = selectedItem
{% endhighlight %}

{% highlight swift %}
// In StoreDetailViewController.viewDidLoad
self.primaryImageView.futureImage = displayItem!.primaryImageFuture
self.detailImageCarousel.images = displayItem!.detailImageFutures
{% endhighlight %}

Not only did you ignore that download task again, you then went and passed an array of futures as if it was an array of the actual blamming images. How do you sleep at night?

Now, obviously, that `detailImageCarousel` is going to need to know to be on the lookout for FutureImage instances so that it can assign them to the `futureImage` properties on its image views. The only way to avoid that would be to also subclass UIImageView and override its `image` property. But then you're muddling your UI, and I think it's much nicer to keep the hidden things behind the model, and make everything explicit in your UI.

But `FutureImage` objects _are_ `UIImage` objects, and you can treat them as such in places where you don't want to create extra API just for handling futures. The code above really can work. I've done it.

# Cacheing (is evil)

The nice thing about this abstraction + model combo is that the model effectively becomes the image cache, and your UI is none the wiser. As you pass the model objects around your app, the UI will get the images without knowing that they didn't need downloaded.

But what happens if you repeat a fetch request and trash your model instances? You'd be wastefully downloading some of the same images over again potentially.

Well, at the risk of allowing this abstraction to outgrow itself, I think it's a pretty good place to abstract longer-term caching as well. I'm not going to go into depth on it here as I think it would betray the focus of this post. But this is how the public API could end up looking:

{% highlight swift %}
struct StoreItem: Decodable {
  ...

  lazy var primaryImageFuture: FutureImage = { 
    FutureImage(at: primaryImageUrl, cachePolicy: .toDisk(path: "unique/path/imageId.ext"))
  }()
  
  lazy var anotherImageFuture: FutureImage = {
    FutureImage(at: anotherImageUrl, cachePolicy: .using(cache: memcache, key:"unique/path/imageId.ext"))
  }()
}
{% endhighlight %}

Which would be provided by a new nested `CachePolicy` type in FutureImage that looks something like:

{% highlight swift %}
extension FutureImage {
  /// Encapsulates the desired caching behavior for a `FutureImage`.
  final class CachePolicy {
    /// The unique key to be used to store and retrieve the image from the cache.
    private var key: NSString?
    /// The cache object residing in memory where the image should be stored.
    private weak var cache: NSCache<NSString, UIImage>?
    /// The local disk path where the image should be stored.
    private var diskPath: URL?

    /// Convenience method for initializing a policy that only caches to disk.
    static func toDisk(path: URL) -> CachePolicy {
      CachePolicy(memCache: nil, diskPath: path)
    }

    /// Convenience method for initializing a policy that only caches to the 
    /// provided in-memory `NSCache` object.
    static func using(cache: NSCache<NSString, UIImage>, key: String) -> CachePolicy {
      CachePolicy(memCache: (key, cache), diskPath: nil)
    }

    /// The designated initalizer for this class.
    init(memCache: (key: String, cache: NSCache<NSString, UIImage>)?, diskPath: URL?) {
      // <Property setting omitted>
    }

    /// Retrieves the cached image, if any is available.
    fileprivate var cachedImage: UIImage? {
      ...
    }

    /// Caches the provided image and data according to this object's configuration.
    fileprivate func cache(_ image: UIImage, data: Data?) {
      ...
    }
  }
}
{% endhighlight %}

Inside `FutureImage`, you'd store this cache policy object and use it to retrieve a cached image if available, or to otherwise cache the image after it's downloaded. The FutureImage wouldn't need to be privy to the underlying details since the CachePolicy would hide them away. The FutureImage logic would stay tidy, aside from some checks for image presense in its initializer.

# Nothing new…

This was super fun to work on because I wasn't going off of any outside references. I was making it up as I went along, and kept thinking I'd hit a blocker at some point, but I didn't. And it ended up nicer to use than I ever thought it would.

That said, I expect this pattern has been done before, and probably in a popular library even. If you've read the other posts here, you know dependencies are not by bag. So while I did make this myself, and I did have a blast doing it, and I therefore wanted to share it, I do not lay claim to it.

If you use it, I hope it's as fun for you as it was for me. But maybe don't use it. Maybe see if you can come up with something even better. That's where the real fun is.
