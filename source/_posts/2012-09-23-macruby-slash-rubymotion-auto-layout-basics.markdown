---
layout: post
title: "MacRuby/Rubymotion Auto Layout Basics"
date: 2012-09-23 15:23
comments: true
categories: 
- MacRuby
- Rubymotion
- Ruby
- Grand Central Dispatch
- GCD
---

Until now whenever we used wanted to layout views on Rubymotion or MacRuby we harcoded the size and position of the UI elements. To achieve a little of dynamic we used the old style auto resizing masking.
With this article you'll get basic knowledge about the Cocoa/CocoaTouch Auto Layout architecture. For more on this topic you're strongly encouraged to check the sources[¹](#source).

The Auto Layout system let use defining layout __constraints__ for the user interface elements, these __constraints__ represent relationship between user interface elemets such as "these views line up head to tail" or "this button should move with this split view subview". When laying out the user interface, a constraint satisfaction system arranges the elements in a way that most closely meets the constraints. if you configure constraints that the system cannot be satisfy, an exception is thrown.

####What are constraints?
Constraints are rules for layout elements in your user interface. For example the help you to specify the a text label should be centered on its superview and keep the same proportional to its superview even though the superview sizes changed. Let's illustrate it by real life scenario:
- Localization example (image)
- Auto Rotation example (image)

Constraints themselves are objects, actually instances of [NSLayoutConstraint](https://developer.apple.com/library/ios/documentation/AppKit/Reference/NSLayoutConstraint_Class/NSLayoutConstraint/NSLayoutConstraint.html#//apple_ref/occ/cl/NSLayoutConstraint) that you can install on views objects (instance of UIView on iOS 6 or Instace of NSView on Mac OS X ≥ 10.7)
Typically, you specify the constraints in Interface Builder, well but you and and me can do it better :-), we will create it programmatically by using an ASCII-art[²](#ascii) inspired format string and by using a form that looks very much like an linear equations[³](#equations):
####pseudocode:
   - <code id="ascii">H:|-[input_field]-[action_button]-|<br/></code><br />
   {% img /images/posts/ascii.png %}<br />
   - <code id="linear">view1.attr1 < relation > view2.attr2 * multiplier + constant</code><br />
``` ruby
# view1.width == 0.5 * view2.width + 0
NSLayoutConstraint.constraintWithItem(view1,
                            attribute: NSLayoutAttributeWidth,
                            relatedBy: NSLayoutRelationEqual,
                               toItem: view2,
                            attribute: NSLayoutAttributeWidth,
                           multiplier: 0.5,
                             constant: 0)
``` 
####what we want to achieve:
{% img /images/posts/auto_layout_colored_portrait.png 200 300 iPhone layout #2 %}
{% img /images/posts/auto_layout_colored_landscape.png 300 200 Phone layout #4 %}<br />
#### Explanation:
We have a UIViewcontroller with 5 subviews:
1. UILabel: (Title label) should be directed attached to the top of it superview
2. UILabel: (subtitle label) should be direct attached to the bottom of the title label
3. UITextField: (Symbol input field) the input field is placed 5pts from the bottom of the subtitle label
4. UIButton: the (action button) is direct attached to the right size of the input text.
5. UILabel: (disclaimer label), it's bottom side is placed 5 pts above the bottom of the superview.
we want that all this relationship remains, don't matter how the superview ratio changes.

####how does it looks like in code?
``` ruby Discover if a number is prime https://github.com/seanlilmateus/MAStockPriceFetcher Source article
# first of all we need an hash with our views and the name that we want to use tho refer them 
views_dict = { 
	"title_label" => @title_label, 
	"subtitle_label" => @subtitle_label, 
	"button_action" => @button_action, 
	"text_field" => @text_field, 
	"info_label" => @info_label 
}
# we create two constraints: 
# First row with the title_label
constraints = NSLayoutConstraint.constraintsWithVisualFormat("H:|-[title_label]-|", 
                                                    options: 0, 
                                                    metrics: nil, 
                                                      views: views_dict)
self.view.addConstraints(constraints)
# second row with the subtitle_label
self.view.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[subtitle_label]-|", 
                                                                options: 0, 
                                                                metrics: nil, 
                                                                  views: views_dict))
# third row with an infolabel centered
self.view.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[info_label]-|", 
																options: 0, 
																metrics: nil, 
																views: views_dict))

metrics = {"width" => 100, "height"=> 80 }
self.view.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("V:[info_label(==height@1000)]-5-|", 
                                                                  options: 0, #  
                                                                  metrics: metrics, 
                                                                    views: views_dict))
# @button_action.width == 0.5 * @text_field.width + 0
self.view.addConstraint(NSLayoutConstraint.constraintWithItem(@button_action,
                                                    attribute: NSLayoutAttributeWidth,
                                                    relatedBy: NSLayoutRelationEqual,
                                                       toItem: @text_field,
                                                    attribute: NSLayoutAttributeWidth,
                                                   multiplier: 0.5,
                                                     constant: 0))
```

##Final result:
{% img /images/posts/auto_layout_original_portrait.png 200 300 iPhone layout #1 %}
{% img /images/posts/auto_layout_original_landscape.png 300 200 iPhone layout #3 %}<br />

This code might work on all devices the same [iphone, retina iphone and iphone 5]
the example code on [github](https://github.com/seanlilmateus/MAStockPriceFetcher) include some localization tweaks, it shows what are the benifits of autolayout for localization and other stuffs.

 <h2 id="source">Sources:</h2> 
- [Cocoa Autolayout WWDC 2011 video](https://developer.apple.com/videos/wwdc/2011/includes/cocoa-autolayout.html#cocoa-autolayout)<br/>
- [Cocoa Auto Layout Guide](http://developer.apple.com/library/mac/#documentation/UserExperience/Conceptual/AutolayoutPG/Articles/Introduction.html#//apple_ref/doc/uid/TP40010853-CH1-DontLinkElementID_2)<br/>
- [Beginning Auto Layout in iOS 6: Part 1 / 2](http://www.raywenderlich.com/20897/beginning-auto-layout-part-1-of-2)<br/>
- [Beginning Auto Layout in iOS 6: Part 2 / 2](http://www.raywenderlich.com/20897/beginning-auto-layout-part-2-of-2)<br/>
- [WWDC 2012: Best Practices for Mastering Auto Layout](https://developer.apple.com/videos/wwdc/2012/?id=228)<br/>

