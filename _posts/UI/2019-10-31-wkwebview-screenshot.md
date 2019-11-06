---

layout: post

title: 'WKWebView在屏幕外时无法截图'

subtitle: 'WKWebView在屏幕外时无法截图'

date: 2019-10-31

categories: 技术

cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'

tags: iOS开发，WKWebView，截图

---

##  一、背景

苹果今年推出了最新的iOS13系统，在最新的API中正式废弃UIWebView，需要升级到WKWebView，目前过渡阶段可能会审核被拒：

> - ITMS-90809: Deprecated API Usage - Apple will stop accepting submissions of apps that use UIWebView APIs . See [**here**](https://developer.apple.com/documentation/uikit/uiwebview) for more information.

鉴于WKWebView相对于UIWebView性能提升明显，因此需要对App内所有的UIWebView进行升级替换成WKWebView，而升级过程中发现**WKWebView在屏幕外时无法截图成功**

## 二、解决方案

### 1、我的场景

我有一个UIScrollView嵌套了多个UIView，而每个UIView有一个WKWebView，因此滑动的时候可能会造成部分webView在屏幕外。

### 2、原因分析

通过查看视图层级结构发现(推荐使用腾讯iOS团队推出的[Lookin](https://lookin.work/)查看Layer，主要免费），当移动到屏幕外后，真实的渲染层会被移除，当回到屏幕上后，渲染层又添加上了，因此，猜测是WKWebView内部做了优化导致。

### 3、解决思路

既然需要在屏幕内才能截图，肯定是要在截图之前将webView移动到屏幕内，经多次实验发现只要是WKWebView和屏幕存在相交区域，
即使被别的视图完全盖住也是可以成功截图的，因此可以将webview移到一个黑色背景视图后盖住。
我们可以使用**UIScrollViewDelegate**的滑动回调方法分别对WKWebview进行移出屏幕和移回屏幕的操作

### 4、代码

- 首先给一下我的截图代码（该方法对UIWebView和WKWebView均有效）

```java
/**
 给视图截图
 */
@interface UIView (Screenshot)

/**
 高清截图
 @return 返回截图后的UIImage
 */
- (UIImage *)kep_HDScreenshotImage;
@end

@implementation UIView (Screenshot)
- (UIImage *)kep_HDScreenshotImage {
    UIImage *image = nil;
    UIGraphicsBeginImageContextWithOptions(self.bounds.size, NO, 0);
    if ([self respondsToSelector:@selector(drawViewHierarchyInRect:afterScreenUpdates:)]) {
        for (UIView *subview in self.subviews) {
            if (subview.subviews.count > 0) {
                for (UIView *view in subview.subviews) {
                    CGRect rect = [view convertRect:view.bounds toView:self];
                    [view drawViewHierarchyInRect:rect afterScreenUpdates:NO];
                }
            } else {
                CGRect rect = [subview convertRect:subview.bounds toView:self];
                [subview drawViewHierarchyInRect:rect afterScreenUpdates:NO];
            }
        }
    } else {
        CGContextRef context = UIGraphicsGetCurrentContext();
        [self.layer renderInContext:context];
    }
    image = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return image;
}

@end
```
- 滑动手势停止时移动WKWebView到屏幕内，滑动手势开始时移动WKWebView回屏幕外

```java
#pragma mark - For WKWebView Screenshot

/// 移动所有的WKWebview到屏幕内，因为WKWebView不能在屏幕外截图
- (void)moveAllWebViewToScreen {
    UIView *currentBoxView = self.currentWebView.superview;
    [self.scrollView bringSubviewToFront:currentBoxView];
    for (WKWebView *webView in self.webViews) {
        [webView mas_remakeConstraints:^(MASConstraintMaker *make) {
            make.center.equalTo(currentBoxView);
            make.size.mas_equalTo(webView.frame.size);
        }];
    }
    [self.scrollView layoutIfNeeded];
}

/// 移动所有的WKWebview到scrollView的对应位置
- (void)resetAllWebViewForCanScroll {
    UIView *currentBoxView = self.currentWebView.superview;
    [self.scrollView bringSubviewToFront:currentBoxView];
    for (WKWebView *webView in self.webViews) {
        [webView mas_remakeConstraints:^(MASConstraintMaker *make) {
            make.center.equalTo(webView.superview);
            make.size.mas_equalTo(webView.frame.size);
        }];
    }
    [self.scrollView layoutIfNeeded];
}

#pragma mark - UIScrollViewDelegate

- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView {
    [self resetAllImageViewForCanScroll];
}

- (void)scrollViewDidEndScrollingAnimation:(UIScrollView *)scrollView {
    [self moveAllImageViewToScreen];
}

- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate {
    if (!decelerate) {
        [self moveAllImageViewToScreen];
    }
}

- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    [self moveAllImageViewToScreen];
}
```

- 获取截图的代码

```java
UIImage *image = [webView kep_HDScreenshotImage];
```

### 5、注意点

- 移动WKWebView到屏幕内后如果立即截图会无法成功，需要等一会儿，不同机型等待时长不同，等待时间经试验会小于0.5秒
- 笔者暂未找到能让WKWebView在屏幕外而渲染层不被移除的方法。