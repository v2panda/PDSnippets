# 优化 TableView


### 1.UITableView 的简单认识
UITableView最核心的思想就是UITableViewCell的重用机制。简单的理解就是：UITableView只会创建一屏幕（或一屏幕多一点）的UITableViewCell，其他都是从中取出来重用的。每当Cell滑出屏幕时，就会放入到一个集合（或数组）中（这里就相当于一个重用池），当要显示某一位置的Cell时，会先去集合（或数组）中取，如果有，就直接拿来显示；如果没有，才会创建。这样做的好处可想而知，极大的减少了内存的开销。

### 2.UITableView 的回调方法

UITableView最主要的两个回调方法是`tableView:cellForRowAtIndexPath:`和`tableView:heightForRowAtIndexPath:`。理想上我们是会认为UITableView会先调用前者，再调用后者，因为这和我们创建控件的思路是一样的，先创建它，再设置它的布局。但实际上却并非如此，我们都知道，UITableView是继承自UIScrollView的，需要先确定它的contentSize及每个Cell的位置，然后才会把重用的Cell放置到对应的位置。所以事实上，UITableView的回调顺序是先多次调用`tableView:heightForRowAtIndexPath:`以确定contentSize及Cell的位置，然后才会调用`tableView:cellForRowAtIndexPath:`，从而来显示在当前屏幕的Cell。

举个例子来说：如果现在要显示100个Cell，当前屏幕显示5个。那么刷新（reload）UITableView时，UITableView会先调用100次`tableView:heightForRowAtIndexPath:`方法，然后调用5次`tableView:cellForRowAtIndexPath:`方法；滚动屏幕时，每当Cell滚入屏幕，都会调用一次`tableView:heightForRowAtIndexPath:`、`tableView:cellForRowAtIndexPath:`方法。


### 3.实践
1.计算缓存 Cell 高度
2.异步绘制 Cell

```
//异步绘制
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        CGRect rect = [_data[@"frame"] CGRectValue];
        UIGraphicsBeginImageContextWithOptions(rect.size, YES, 0);
        CGContextRef context = UIGraphicsGetCurrentContext();
	//整个内容的背景
        [[UIColor colorWithRed:250/255.0 green:250/255.0 blue:250/255.0 alpha:1] set];
        CGContextFillRect(context, rect);
	//转发内容的背景
        if ([_data valueForKey:@"subData"]) {
            [[UIColor colorWithRed:243/255.0 green:243/255.0 blue:243/255.0 alpha:1] set];
            CGRect subFrame = [_data[@"subData"][@"frame"] CGRectValue];
            CGContextFillRect(context, subFrame);
            [[UIColor colorWithRed:200/255.0 green:200/255.0 blue:200/255.0 alpha:1] set];
            CGContextFillRect(context, CGRectMake(0, subFrame.origin.y, rect.size.width, .5));
        }
        
        {
	    //名字
            float leftX = SIZE_GAP_LEFT+SIZE_AVATAR+SIZE_GAP_BIG;
            float x = leftX;
            float y = (SIZE_AVATAR-(SIZE_FONT_NAME+SIZE_FONT_SUBTITLE+6))/2-2+SIZE_GAP_TOP+SIZE_GAP_SMALL-5;
            [_data[@"name"] drawInContext:context withPosition:CGPointMake(x, y) andFont:FontWithSize(SIZE_FONT_NAME)
                             andTextColor:[UIColor colorWithRed:106/255.0 green:140/255.0 blue:181/255.0 alpha:1]
                                andHeight:rect.size.height];
	    //时间+设备
            y += SIZE_FONT_NAME+5;
            float fromX = leftX;
            float size = [UIScreen screenWidth]-leftX;
            NSString *from = [NSString stringWithFormat:@"%@  %@", _data[@"time"], _data[@"from"]];
            [from drawInContext:context withPosition:CGPointMake(fromX, y) andFont:FontWithSize(SIZE_FONT_SUBTITLE)
                   andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1]
                      andHeight:rect.size.height andWidth:size];
        }
	//将绘制的内容以图片的形式返回，并调主线程显示
	UIImage *temp = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        dispatch_async(dispatch_get_main_queue(), ^{
            if (flag==drawColorFlag) {
                postBGView.frame = rect;
                postBGView.image = nil;
                postBGView.image = temp;
            }
	}
	//内容如果是图文混排，就添加View，用CoreText绘制
	[self drawText];
}}
```


3.滑动UITableView时，按需加载对应的内容，滚动很快时，只加载目标范围内的Cell

```
//按需加载 - 如果目标行与当前行相差超过指定行数，只在目标滚动范围的前后指定3行加载。
- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset{
    NSIndexPath *ip = [self indexPathForRowAtPoint:CGPointMake(0, targetContentOffset->y)];
    NSIndexPath *cip = [[self indexPathsForVisibleRows] firstObject];
    NSInteger skipCount = 8;
    if (labs(cip.row-ip.row)>skipCount) {
        NSArray *temp = [self indexPathsForRowsInRect:CGRectMake(0, targetContentOffset->y, self.width, self.height)];
        NSMutableArray *arr = [NSMutableArray arrayWithArray:temp];
        if (velocity.y<0) {
            NSIndexPath *indexPath = [temp lastObject];
            if (indexPath.row+3<datas.count) {
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+1 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+2 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+3 inSection:0]];
            }
        } else {
            NSIndexPath *indexPath = [temp firstObject];
            if (indexPath.row>3) {
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-3 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-2 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-1 inSection:0]];
            }
        }
        [needLoadArr addObjectsFromArray:arr];
    }
}
```

记得在tableView:cellForRowAtIndexPath:方法中加入判断：

```
if (needLoadArr.count>0&&[needLoadArr indexOfObject:indexPath]==NSNotFound) {
    [cell clear];
    return;
}

```

### 4.总结

UITableView的优化主要从三个方面入手：

- 提前计算并缓存好高度（布局），因为heightForRowAtIndexPath:是调用最频繁的方法；
- 异步绘制，遇到复杂界面，遇到性能瓶颈时，可能就是突破口；
- 滑动时按需加载，这个在大量图片展示，网络加载的时候很管用！（SDWebImage已经实现异步加载，配合这条性能杠杠的）。

除了上面最主要的三个方面外，还有很多几乎大伙都很熟知的优化点：

- 通过正确的reuseIdentifier重用cells
- 尽量多的设置views 为不透明，包括cell本身。
- 避免渐变，图像缩放，屏幕以外的绘制。
- 如果cell显示的内容来自网络，确保异步和缓存。
- 使用shadowPath来建立阴影。
- 减少子视图的数目。
- cellForRowAtIndexPath:中做尽量少的工作，如果需要做相同的工作，那么只做一次并缓存结果。
- 使用适当的数据结构存储你要的信息，不同的结构有对于不同的操作有不同的代价。
- 使用rowHeight，sectionFooterHeight，sectionHeaderHeight为常数，而不是询问代理。


----



## Reference

[UITableView优化技巧](http://longxdragon.github.io/2015/05/26/UITableView%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/)

[VVeboTableViewDemo](https://github.com/johnil/VVeboTableViewDemo)

