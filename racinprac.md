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

---

## Describing an algorithm with functions

### Example: Finding the best voucher to cover a purchase

- If there are vouchers with higher value than the purchase, use the lowest valued one
- Otherwise, use the highest valued voucher available

---

# Imperative approach

```objc
- (id<Voucher>)voucherForPurchaseAmount:(NSDecimalNumber *)purchaseAmount {

  NSArray *vouchers = [[self voucherLibrary] vouchers];

  NSArray *sortedVouchers = [vouchers sortedArrayUsingComparator:^NSComparisonResult(id<Voucher> voucher1, id<Voucher> voucher2) {
	NSDecimalNumber *voucher1Amount = voucher1.amount;
	NSDecimalNumber *voucher2Amount = voucher2.amount;

	if (voucher1Amount == nil && voucher2Amount == nil) {
	  return NSOrderedSame;
	} else if (voucher1Amount == nil) {
	  return NSOrderedDescending;
	} else if (voucher2Amount == nil) {
	  return NSOrderedAscending;
	} else {
	  return [voucher1Amount compare:voucher2Amount];
	}
  }];

  id<Voucher> bestVoucher = nil;
  for (id<Voucher> voucher in sortedVouchers) {
    NSDecimalNumber *voucherAmount = voucher.amount;
    if (!voucherAmount) continue;

    if ([voucherAmount cch_isLessThen:purchaseAmount]) {
      bestVoucher = voucher;
    } else if ([voucherAmount cch_isGreaterThanOrEqualTo:purchaseAmount]) {
      bestVoucher = voucher;
      break;
    }
  }

  return bestVoucher;
}
```

---

# Separate the nil filter?

```objc
NSMutableArray *vouchersWithValue = [NSMutableArray array];
for (id<Voucher> voucher in sortedVouchers) {
	if (voucher.amount) {
	  [vouchersWithValue addObject:voucher];
	}
}

id<Voucher> bestVoucher = nil;
for (id<Voucher> voucher in vouchersWithValue) {
  NSDecimalNumber *voucherAmount = voucher.amount;
  if ([voucherAmount cch_isLessThen:purchaseAmount]) {
    bestVoucher = voucher;
  } else if ([voucherAmount cch_isGreaterThanOrEqualTo:purchaseAmount]) {
    bestVoucher = voucher;
    break;
  }
}
```

---

# Functional

```objc
- (id<Voucher>)voucherForPurchaseAmount:(NSDecimalNumber *)purchaseAmount {
[[[[[[[[self voucherLibrary]
    vouchers]
    sortedArrayUsingComparator:^NSComparisonResult(id<Voucher> voucher1, id<Voucher> voucher2) {
      NSDecimalNumber *voucher1Amount = voucher1.amount;
      NSDecimalNumber *voucher2Amount = voucher2.amount;

      if (voucher1Amount == nil && voucher2Amount == nil) {
        return NSOrderedSame;
      } else if (voucher1Amount == nil) {
        return NSOrderedDescending;
      } else if (voucher2Amount == nil) {
        return NSOrderedAscending;
      } else {
        return [voucher1Amount compare:voucher2Amount];
      }
    }]
    rac_sequence]
    filter:^BOOL(id<Voucher> voucher) {
      return voucher.amount != nil;
    }]
    cch_takeUptoBlock:^BOOL(id<Voucher> voucher) {
      return [voucher.amount cch_isGreaterThanOrEqualTo:purchaseAmount];
    }]
    array]
    lastObject];
```

---

### 1. ~~A reactive container view controller (Me)~~
### 2. ~~Functional data processing (Mark Corbyn)~~
### 3. <something> (Jeames Bone)

---
