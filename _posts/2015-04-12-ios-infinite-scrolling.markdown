---
title: iOS 实战 - Infinite Scrolling
author: 但江
avatar: danjiang
location: 成都
category: programming
---

![Weight Ruler](/images/weight-ruler.gif)

以上动画展示了应用 [每日体重记录][1] 中通过尺子来输入体重，要实现尺子的输入控件需要通过继承 UIScrollView 来做 Infinite Scrolling。

## 尺子中的刻度

![Weight Ruler Scale](/images/weight-ruler-scale.jpg)

首先我们需要通过继承 UIView 来实现上图中的刻度块，一个刻度块包含十个刻度和一个刻度值，不需要使用图片，通过 Quartz 框架来绘制，代码如下：

{% highlight objc %}
extern const CGFloat DTRulerScaleBlockWidth;
extern const CGFloat DTRulerScaleGap;

@interface DTRulerViewScale : UIView

@property (nonatomic) int weight;
- (id)initWithWeight:(int)weight frameHeight:(CGFloat)height;

@end
{% endhighlight %}

{% highlight objc %}
#import "DTRulerViewScale.h"
#import "DTThemeManager.h"
#import "DTTheme.h"

const CGFloat DTRulerScaleBlockWidth = 100;
const CGFloat DTRulerScaleGap = 10;

@implementation DTRulerViewScale

- (id)initWithWeight:(int)weight frameHeight:(CGFloat)height {
    self = [super initWithFrame:CGRectMake(0.0, 0.0, DTRulerScaleBlockWidth, height)];
    if (self) {
        self.backgroundColor = [[DTThemeManager sharedInstance] currentTheme].temporaryViewBackgroundColor;
        _weight = weight;
    }
    return self;
}

- (void)setWeight:(int)weight {
    if (_weight != weight) {
        _weight = weight;
        [self setNeedsDisplay];
    }
}

- (void)drawRect:(CGRect)rect {
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    [self doScaleWithWeight:self.weight rect:rect context:context];
}

- (void)doScaleWithWeight:(int)weight rect:(CGRect)rect context:(CGContextRef)context {
    CGFloat startX = CGRectGetMidX(rect) + DTRulerScaleGap / 2;
    CGFloat startY = CGRectGetMinY(rect);
    [[[DTThemeManager sharedInstance] currentTheme].temporaryViewTextColor setStroke];
    
    NSString *weightLabel = [NSString stringWithFormat:@"%d", weight];
    NSDictionary *attributes = @{NSFontAttributeName: [[DTThemeManager sharedInstance] rulerScaleTextFont],
                                 NSForegroundColorAttributeName: [[DTThemeManager sharedInstance] currentTheme].temporaryViewTextColor};
    
    CGSize textSize = [weightLabel sizeWithAttributes:attributes];
    [weightLabel drawInRect:CGRectMake(startX - textSize.width / 2, startY + self.frame.size.height * 0.46875 + 4, textSize.width, textSize.height)
             withAttributes:attributes];
    
    CGContextSetLineWidth(context, 2.0);
    CGContextMoveToPoint(context, startX, startY);
    CGContextAddLineToPoint(context, startX, self.frame.size.height * 0.46875);
    CGContextDrawPath(context, kCGPathStroke);
    
    [self doScaleFromMidToEgdeWithStartX:startX startY:startY counter:4 plus:YES context:context];
    [self doScaleFromMidToEgdeWithStartX:startX startY:startY counter:5 plus:NO context:context];
}

- (void)doScaleFromMidToEgdeWithStartX:(CGFloat)startX startY:(CGFloat)startY counter:(int)counter plus:(BOOL)plus context:(CGContextRef)context {
    CGContextSetLineWidth(context, 1);
    for (int i = 0; i < counter; i++) {
        if (plus) {
            startX += DTRulerScaleGap;
        } else {
            startX -= DTRulerScaleGap;
        }
        CGContextMoveToPoint(context, startX, startY);
        CGContextAddLineToPoint(context, startX, self.frame.size.height * 0.3125);
        CGContextDrawPath(context, kCGPathStroke);
    }
}

@end
{% endhighlight %}

## UIScrollView Infinite Scrolling

我们已经实现了尺子的刻度块，刻度块的长度为 100 点，如果以 iPhone 4s 的宽度，用户能够看到的刻度块也就是 3 个多一点，所以我们设定由 5 个刻度块首尾相接组成一把尺子，代码如下：

{% highlight objc %}
@interface DTRulerView : UIScrollView

- (instancetype)initWithFrame:(CGRect)frame weight:(float)weight;

@end
{% endhighlight %}

{% highlight objc %}
#import "DTRulerView.h"
#import "DTRulerView.h"
#import "DTRulerViewScale.h"

static const int DTRulerScaleBlockNumber = 5;
static const int DTRulerMinScale = 9;
static const int DTRulerMaxScale = 999;

@interface DTRulerView ()

@property BOOL isAnimating;
@property (strong, nonatomic) NSMutableArray *weights;
@property (strong, nonatomic) NSMutableArray *rulerScales;
@property (strong, nonatomic) NSMutableArray *reusedRulerScales;
@property (strong, nonatomic) UIView *container;

@end

@implementation DTRulerView

- (instancetype)initWithFrame:(CGRect)frame weight:(float)weight {
    self = [super initWithFrame:frame]; // frame is necessary for caculating scale position
    if (self) {
        self.contentSize = CGSizeMake(DTRulerScaleBlockWidth * DTRulerScaleBlockNumber, self.frame.size.height);
        self.showsHorizontalScrollIndicator = NO;
        self.delegate = self;
        
        _container = [[UIView alloc] init];
        _container.frame = CGRectMake(0, 0, self.contentSize.width, self.frame.size.height);
        _container.userInteractionEnabled = NO;
        [self addSubview:_container];
        
        _weights = [[NSMutableArray alloc] init];
        _rulerScales = [[NSMutableArray alloc] init];
        _reusedRulerScales = [[NSMutableArray alloc] init];
        
        int weightInt = roundf(weight * 10);
        int weightDecimal = weightInt % 10; // get weight`s decimal
        weightInt = weightInt / 10;
        int offsetGapNumber = 0;
        if (weightDecimal <= 4) {
            offsetGapNumber = weightDecimal + 5;
        } else {
            offsetGapNumber = weightDecimal - 5;
            weightInt++;
        }
        CGFloat startX = CGRectGetMidX(self.frame) - DTRulerScaleGap / 2; // point to middle scale block`s first scale
        startX -= offsetGapNumber * DTRulerScaleGap; // first scale with right gap offset
        
        int indexs[DTRulerScaleBlockNumber] = {0, -1, -2, 1, 2};
        for (int i = 0; i < DTRulerScaleBlockNumber; i++) {
            int index = indexs[i];
            [self placeWeight:((index == 0) ? [NSNumber numberWithInt:weightInt] : nil)
                orNewPrevious:(index < 0)
               calculateFrame:^CGRect(CGRect frame) {
                   frame.origin.x += startX + frame.size.width * index;
                   return frame;
               }];
        }
    }
    return self;
}

@end
{% endhighlight %}

现在我们已经可以展示出尺子了，但是我们还没有实现滑动，在滑动过程中 layoutSubviews 方法会被调用，看名字就知道是用来重新布局子视图，也就是刻度块，在 layoutSubviews 方法中需要将 UIScrollView 的 Content 再次居中，并且依次移动所有刻度块保持相同的偏移，滑动过后，有的刻度块就会超出最大或最小可视区域，需要将其移除，同样需要生成新的刻度块来显示在尺子中，其原理就像 UITableView 在上下滑动的过程中不断地重用 UITablewViewCell 来显示数据，这样才能保证内存使用量不会不断上升，代码如下：

{% highlight objc %}
- (void)layoutSubviews{
    [super layoutSubviews];
    
    [self recenterIfNecessary];
    
    CGRect visibleBounds = [self visibleBounds];
    CGFloat minimumVisibleX = CGRectGetMinX(visibleBounds);
    CGFloat maximumVisibleX = CGRectGetMaxX(visibleBounds);
    
    [self tileChildrensFromMinX:minimumVisibleX toMaxX:maximumVisibleX];
}

- (void)recenterIfNecessary {
    CGPoint currentOffset = self.contentOffset;
    CGFloat contentWidth = self.contentSize.width;
    CGFloat centerOffsetX = (contentWidth - self.bounds.size.width) / 2.0;
    CGFloat distanceFromCenterSign =  currentOffset.x - centerOffsetX;
    CGFloat distanceFromCenter = fabs(distanceFromCenterSign);
    if (distanceFromCenter > centerOffsetX) {
        self.contentOffset = CGPointMake(centerOffsetX, currentOffset.y);
        // move content by the same amount so it appears to stay still
        [self moveAllChildrensWithOffsetX:-distanceFromCenterSign animation:NO];
    }
}

- (void)moveAllChildrensWithOffsetX:(CGFloat)offsetX animation:(BOOL)animation{
    if (animation) {
        [UIView animateWithDuration:0.2 animations:^{
            for (DTRulerViewScale *rulerScale in self.rulerScales) {
                rulerScale.center = CGPointMake(rulerScale.center.x + offsetX, rulerScale.center.y);
            }
        }];
    } else {
        for (DTRulerViewScale *rulerScale in self.rulerScales) {
            rulerScale.center = CGPointMake(rulerScale.center.x + offsetX, rulerScale.center.y);
        }
    }
}

- (CGRect)visibleBounds {
    return [self convertRect:[self bounds] toView:self.container];
}

- (void)placeWeight:(NSNumber *)weight orNewPrevious:(BOOL)previous calculateFrame:(CGRect (^)(CGRect frame))calculateFrame {
    DTRulerViewScale *rulerScale = self.reusedRulerScales.lastObject;
    if (!rulerScale) {
        rulerScale = [[DTRulerViewScale alloc] initWithWeight:0 frameHeight:self.frame.size.height];
    }
    if (!weight && !previous) {
        weight = [self nextWeight];
    }
    if (!previous) {
        [_weights addObject:weight];
        [_rulerScales addObject:rulerScale];
        [_container addSubview:rulerScale];
    } else {
        weight = [self previousWeight];
        [_weights insertObject:weight atIndex:0];
        [_rulerScales insertObject:rulerScale atIndex:0];
        [_container insertSubview:rulerScale atIndex:0];
    }
    rulerScale.weight = weight.intValue;
    rulerScale.frame = calculateFrame(rulerScale.frame);
}

- (NSNumber *)previousWeight {
    NSNumber *weight = [self.weights objectAtIndex:0];
    if (weight.intValue > DTRulerMinScale) {
        return [NSNumber numberWithInt:weight.intValue - 1];
    } else {
        return [NSNumber numberWithInt:DTRulerMaxScale];
    }
}

- (NSNumber *)nextWeight {
    NSNumber *weight = self.weights.lastObject;
    if (weight.intValue < DTRulerMaxScale) {
        return [NSNumber numberWithInt:weight.intValue + 1];
    } else {
        return [NSNumber numberWithInt:DTRulerMinScale];
    }
}

- (void)tileChildrensFromMinX:(CGFloat)minimumVisibleX toMaxX:(CGFloat)maximumVisibleX {
    // add child that are missing on right side
    DTRulerViewScale *last = self.rulerScales.lastObject;
    CGFloat rightEdge = CGRectGetMaxX(last.frame);
    if(rightEdge < maximumVisibleX) {
        [self placeWeight:nil orNewPrevious:NO calculateFrame:^CGRect(CGRect frame) {
            frame.origin.x = rightEdge;
            return frame;
        }];
    }
    
    // add child that are missing on left side
    DTRulerViewScale *first = [self.rulerScales objectAtIndex:0];
    CGFloat leftEdge = CGRectGetMinX(first.frame);
    if (leftEdge > minimumVisibleX) {
        [self placeWeight:nil orNewPrevious:YES calculateFrame:^CGRect(CGRect frame) {
            frame.origin.x = leftEdge - frame.size.width;
            return frame;
        }];
    }
    
    if(self.rulerScales.count > DTRulerScaleBlockNumber){
        // remove child that have fallen off right edge
        last = self.rulerScales.lastObject;
        leftEdge = CGRectGetMinX(last.frame);
        if (last && leftEdge > maximumVisibleX) {
            [self.reusedRulerScales addObject:last];
            [self.weights removeLastObject];
            [self.rulerScales removeLastObject];
            [last removeFromSuperview];
        }
        // remove child that have fallen off left edge
        first = [self.rulerScales objectAtIndex:0];
        rightEdge = CGRectGetMaxX(first.frame);
        if (first && rightEdge < minimumVisibleX) {
            [self.reusedRulerScales addObject:first];
            [self.weights removeObjectAtIndex:0];
            [self.rulerScales removeObjectAtIndex:0];
            [first removeFromSuperview];
        }
    }
}
{% endhighlight %}

## 用户体验在于细节

用户滑动尺子后，我们应该保证指向刻度的箭头指在刻度线上，而不是空隙上，同样的还需要将指向的刻度值显示出来，都是通过 UIScrollViewDelegate 来实现的，需要注意的是用户滑动频率比较快时，刻度值的变化就比较快，很有可能超过视图的渲染频率，所以需要 isEnoughTimeElapsed 方法来确认是不是过了足够的时间来降低改变刻度值的频率，视图渲染的原则如下：

> 60 frames per second is the gold standard, 16.67 milliseconds per frame

剩下的部分代码：

{% highlight objc %}
@class DTRulerView;

@protocol DTRulerViewDelegate

- (void)rulerViewDidChange:(DTRulerView *)rulerView weight:(float)weight;

@end

@interface DTRulerView : UIScrollView <UIScrollViewDelegate>

@property (nonatomic, weak) id<DTRulerViewDelegate> rulerViewDelegate;
- (instancetype)initWithFrame:(CGRect)frame weight:(float)weight;

@end
{% endhighlight %}

{% highlight objc %}
- (float)weightPointedToWithRevise:(BOOL)revise {
    float weightFloat = 0.0;
    CGFloat middleVisibleX = CGRectGetMidX([self visibleBounds]);
    for (int i = 0; i < self.rulerScales.count; i++) {
        DTRulerViewScale *rulerScale = [self.rulerScales objectAtIndex:i];
        CGFloat rulerScaleMinX = CGRectGetMinX(rulerScale.frame);
        CGFloat rulerScaleMaxX = CGRectGetMaxX(rulerScale.frame);
        // point to which scale block
        if (rulerScaleMinX <= middleVisibleX && middleVisibleX < rulerScaleMaxX) {
            NSNumber *weight = [self.weights objectAtIndex:i];
            for (int j = 0; j < 10; j++) {
                CGFloat rulerSingleScaleMinX = rulerScaleMinX + j * DTRulerScaleGap;
                CGFloat rulerSingleScaleMidX = rulerSingleScaleMinX + DTRulerScaleGap/2;
                CGFloat rulerSingleScaleMaxX = rulerScaleMinX + (j + 1) * DTRulerScaleGap;
                // point to which scale block`s single scale
                if (rulerSingleScaleMinX <= middleVisibleX && middleVisibleX < rulerSingleScaleMaxX) {
                    weightFloat = weight.intValue + (j - 5) * 0.1;
                    // point to scale without offset
                    if (revise) {
                        [self moveAllChildrensWithOffsetX:middleVisibleX - rulerSingleScaleMidX animation:YES];
                    }
                    break;
                }
            }
            break;
        }
    }
    return weightFloat;
}

- (BOOL)isEnoughTimeElapsed {
    static NSTimeInterval lastTime = 0;
    NSTimeInterval currentTime = [[NSDate date] timeIntervalSince1970];
    BOOL isEnough = NO;
    if (lastTime == 0) {
        isEnough = YES;
    } else {
        isEnough = currentTime - lastTime > 0.016; // 16.67 milliseconds per frame
    }
    lastTime = currentTime;
    return isEnough;
}

#pragma mark - Delegate Method

- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate {
    if (!decelerate) {
        [self.rulerViewDelegate rulerViewDidChange:(DTRulerView *)scrollView weight:[self weightPointedToWithRevise:YES]];
    }
}

- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    [self.rulerViewDelegate rulerViewDidChange:(DTRulerView *)scrollView weight:[self weightPointedToWithRevise:YES]];
}

- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    if ([self isEnoughTimeElapsed]) {
        [self.rulerViewDelegate rulerViewDidChange:(DTRulerView *)scrollView weight:[self weightPointedToWithRevise:NO]];
    }
}
{% endhighlight %}

[1]: http://danthought.com/weight
