# [fit] RAC
# [fit] IN
# [fit] PRACTICE

#### Adam Sharp (@sharplet) — Mark Corbyn (@markcorbyn) — Jeames Bone (@jeamesbone)
#### August 2015

^ Last ~6 months team adopted ReactiveCocoa

^ Intro RAC as a framework for modelling problems as streams of values over time

---

# :gift:

^ We'd like to share with you 3 interesting examples of interesting ways we've been able to apply RAC in our app

---

![](http://cl.ly/image/1N2P053O0l3c/Screen%20Shot%202015-08-10%20at%208.26.41%20pm.png)

# [fit] WATCH THIS

#### Kevin O'neill - July 2015 - https://vimeo.com/133250189


---

# What have we learned?

- Signals represent streams of values over time

```objc
[textField.rac_textSignal subsribeNext:^(NSString *text) {
  NSLog(@"hey, the text changed! %@", text);
}];
```

---

- You can use operators to transform them

```objc
RACSignal *numberSignal = [textSignal map:^(NSString *text) {
  return @(text.integerValue);
}];
```

---

- They’ll solve all your problems
- Magic and stuff
- $$$

---

# What *haven’t* we learned?

^ All examples will be taken from real code my team's worked on

^ Maybe you didn't realise you could use ReactiveCocoa for this?

---

### 1. A reactive container view controller (Me)
### 2. Functional data processing (Mark Corbyn)
### 3. <something> (Jeames Bone)

---

# Something simple

### Authentication

```objc
/// Stores authentication credentials and tells us if we're authenticated.
@protocol AuthStore <NSObject>

@property (nonatomic, readonly, getter=isAuthenticated) BOOL authenticated;

- (void)storeAccessCredentials:(AccessCredentials *)accessCredentials;
- (nullable AccessCredentials *)retrieveAccessCredentials;
- (void)removeAccessCredentials;

@end
```

---

### Observe authentication status changes

```objc
// Boolean signal sends the current value on subscription, and a new value
// whenever we log in or out.
RACSignal *authenticatedSignal = RACObserve(authStore, authenticated);
```

---

### Select the right view controller

```objc
RACSignal *authenticatedSignal = RACObserve(authStore, authenticated);

// Pick a view controller
RACSignal *viewControllerSignal = [authenticatedSignal map:^(NSNumber *isAuthenticated) {
  if (isAuthenticated.boolValue) {
    return [HomePageViewController new];
  } else {
    return [LandingPageViewController new];
  }
}];
```

---

### Displaying it on screen

#### How do we get values out of a signal? We have to subscribe, like a callback.

```objc
- (void)viewDidLoad {
  [super viewDidLoad];

  // Whenever we get a new view controller, PUSH IT
  [[viewControllerSignal deliverOnMainThread]
    subscribeNext:^(UIViewController *viewController) {
      [self showViewController:viewController sender:self];
    }];
}
```

---

### Ok, so maybe that’s pushing it

#### Let’s make a custom view controller container!

---

### Switching to the latest view controller

```objc
/// Manages a signal of view controllers, always displaying the most recent one
/// as its content.
@interface SwitchingController : UIViewController

- (instanctype)initWithViewControllers:(RACSignal *)viewControllers;

@property (nonatomic, readonly) UIViewController *currentViewController;

@end
```

---

```objc
- (void)viewDidLoad {
  [super viewDidLoad];

  [[self.viewControllers deliverOnMainThread] subscribeNext:^(UIViewController *viewController) {
    [self transitionFrom:self.currentViewController to:viewController];
  }];
}

- (void)transitionFrom:(UIViewController *)fromViewController to:(UIViewController *)toViewController {
  if (!fromViewController) {
    [self addInitialViewController:toViewController];
    return;
  }

  [self addChildViewController:nextViewController];
  nextViewController.view.frame = self.view.bounds;
  [self.view addSubview:nextViewController.view];
  [previousViewController willMoveToParentViewController:nil];

  [self transitionFromViewController:fromViewController
                    toViewController:toViewController
                            duration:0.5
                             options:UIViewAnimationOptionTransitionCrossDissolve
                          animations:nil
                          completion:^(BOOL finished) {
                            [toViewController didMoveToParentViewController:self];
                            [fromViewController removeFromParentViewController];
                            self.currentViewController = toViewController;
                          }];
}
```

---

### Tying a 🎀 on it

---

### Tying a 🎀 on it

- What happens if the view controller changes rapidly (faster than the animation?)
- What happens when the view controller is not on screen? Or deallocated?

---

### Tying a 🎀 on it

- ~~What happens if the view controller changes rapidly (faster than the animation?)~~
- What happens when the view controller is not on screen? Or deallocated?

---

# v1: Simple Throttling

```objc
- (void)viewDidLoad {
  [super viewDidLoad];

  [[[self.viewControllers
    throttle:0.5]
    deliverOnMainThread]
    subscribeNext:^(UIViewController *viewController) {
      [self transitionFrom:self.currentViewController to:viewController];
    }];
}
```

^ First pass: use -throttle: to prevent multiple view controllers during an animation

---

# v2: Only throttle while animating

## 1) keep track of whether we're animating

```objc
@interface SwitchingController ()
@property (getter=isAnimating) BOOL animating;
@end
```

---

## 1) keep track of whether we're animating

```objc
@interface SwitchingController ()
@property (nonatomic, getter=isAnimating) BOOL animating;
@end

// ...

- (void)transitionFrom:(UIViewController *)fromViewController to:(UIViewController *)toViewController {
  // code

  self.animating = YES;

  [self transitionFromViewController:fromViewController
                    toViewController:toViewController
                            duration:0.5
                             options:UIViewAnimationOptionTransitionCrossDissolve
                          animations:nil
                          completion:^(BOOL finished) {
                            // more code
                            self.animating = NO;
                          }];
}
```

---

## 2) throttle with a huge timeout while animating

```objc
  NSDate *forever = [NSDate distantFuture];

  [[[self.viewControllers
    throttle:forever valuesPassingTest:^(id _) {
      return self.isAnimating;
    }]
    deliverOnMainThread]
    subscribeNext:^(UIViewController *viewController) {
      [self transitionFrom:self.currentViewController to:viewController];
    }];
```

---

### 1. ~~A reactive container view controller (Me)~~
### 2. Functional data processing (Mark Corbyn)
### 3. <something> (Jeames Bone)
