---
title: iOS相册Moment功能的优化方案
subtitle: ""

# Summary for listings and search engines
summary: 最近在开发公司产品Perfect365的Gallery模块，包括按日期排序的Moment以及Album这两个模块。Moment功能和系统相册类似，就是根据图片的日期信息进行排序，然后按照不同日期分section显示。

# Link this post with a project
projects: []

# Date published
date: "2019-10-08T00:00:00Z"

# Date updated
lastmod: "2019-10-08T00:00:00Z"

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: ''
  focal_point: ""
  placement: 2
  preview_only: false

authors:
- 金小俊

tags:
- iOS
- 相册

categories:
- iOS
- 相册
---

最近在开发公司产品[Perfect365](https://itunes.apple.com/cn/app/perfect365-one-tap-makeover/id475976577?l=en&mt=8)的Gallery模块, 包括按日期排序的Moment以及Album这两个模块. Moment功能和系统相册类似, 就是根据图片的日期信息进行排序, 然后按照不同日期分section显示. 

![IMG_2897.jpg](http://upload-images.jianshu.io/upload_images/268805-1d651d17e7b0fe00.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Moment的实现思路很简单: 先遍历系统的所有相册, 然后获取每个相册内图片的日期信息, 根据日期进行分类和排序, 最后把枚举完的所有数据放到界面上来显示. 示例代码如下:
```
NSSortDescriptor *sort = [NSSortDescriptor sortDescriptorWithKey:@"date" ascending:NO];

[objects sortUsingDescriptors:@[sort]];

MomentCollection *lastGroup = nil; NSMutableArray *ds = [[NSMutableArray alloc] init];

for (ALAsset *asset in objects)
{
    @autoreleasepool
    {
        NSDateComponents *components = [[NSCalendar currentCalendar] components:NSDayCalendarUnit   |
                                                                                NSMonthCalendarUnit |
                                                                                NSYearCalendarUnit
                                                                       fromDate:[asset date]];
        NSUInteger month = [components month];
        NSUInteger year  = [components year];
        NSUInteger day   = [components day];
        
        if (!lastGroup || lastGroup.year!=year || lastGroup.month!=month || lastGroup.day!=day)
        {
            lastGroup = [MomentCollection new]; [ds addObject:lastGroup];
            
            lastGroup.month = month; lastGroup.year = year; lastGroup.day = day;
        }
        
        ALAsset *lPhoto    = [lastGroup.assetObjs lastObject];
        NSURL   *lPhotoURL = [lPhoto valueForProperty:ALAssetPropertyAssetURL];
        NSURL   *photoURL  = [asset  valueForProperty:ALAssetPropertyAssetURL];
        
        if (![lPhotoURL isEqual:photoURL])
        {
            [lastGroup.assetObjs addObject:asset];
        }
    }
}
```
So far so good, 接下来创建UICollectionView, 设置好dataSource就可以显示moment图片了. 起初我也是这么认为的, 但是对于开发一款拥有6500万用户的App来说everything is possible. 版本发布之后, 很多用户反馈打开相册后App直接freeze掉, what the hell is this? QA测试时一切OK的呀. 好吧, 继续骚扰用户询问到底是神马情况, 用户回复: 我手机里面放了30k+图片, 占了20G+的存储空间. OH MY GOD!!!

###优化方案

对于Moment功能, 肯定需要遍历完系统内的所有相册图片, 然后再按日期排序后显示给用户, 那优化就只能在枚举和排序这两部分来压榨了. 经过2天的苦思冥想决定采用**分批加载+取尾排序**的方案来优化. 具体思路为: 如果用户设备内的图片比较多, 不是等所有图片都枚举排序完了再显示, 而是枚举每隔一定数量的图片(e.g. 50张)后就抛出去(放到NSOperationQueue里)按日期分类并排序, 再显示给用户, 这样让用户看到我们动态加载图片的过程, 让他知道我们的程序still alive, 并且在不断的加载图片. 但是一般情况下排序的耗时会大于图片的枚举, 也就是第一个50张排完序后, 前面枚举放到Queue里面等待排序的已经有好几批了, 那么我们只对最后一批的图片再排序(也就是取尾)并清空当前的Queue, 因为中间的几批数据已经makes no sense了. 方案详细流程图如下:

![曲线流程图.jpg](http://upload-images.jianshu.io/upload_images/268805-1cc472e57a96694b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了最大程度的减轻动态加载后刷新显示对用户造成的突兀感, 在显示之前需要判断用户是否在滑动页面, 只有页面静止的时候刷新显示. 但对于全部图片枚举完成后的最后一批数据则要暂时保存住(否则就木有东东显示了), 待用户停止滑动后reloadData.

###分批加载

Moment需要按日期分类显示(最新的显示在最前面), 所以在枚举相册的时候可以先从camera roll开始(一般用户拍摄的照片相对导入的图片会早一点). 加载到50的倍数张后就抛到queue里面等待排序, 一个相册枚举完后再继续遍历其余的相册...
```
- (void)getPhotosWithGroupTypes:(ALAssetsGroupType)types
                    batchReturn:(BOOL)batch
                     completion:(void (^)(BOOL ret, id obj))completion
{
    self.batchBlock        = completion;
    NSMutableArray *tmpArr = [[NSMutableArray alloc] init];
    
    [self.assetLibary enumerateGroupsWithTypes:types
                                    usingBlock:^(ALAssetsGroup *group, BOOL *stop)
    {
        if (self.stopEnumeratePhoto) {*stop = YES; return;}
         
        NSInteger gType = [[group valueForProperty:ALAssetsGroupPropertyType] integerValue];
         
        if (group && (gType != ALAssetsGroupPhotoStream))
        {
            [group setAssetsFilter:[ALAssetsFilter allPhotos]];
             
            [group enumerateAssetsWithOptions:NSEnumerationReverse
                                   usingBlock:^(ALAsset *result, NSUInteger index, BOOL *stop)
            {
                if (self.stopEnumeratePhoto) {*stop = YES; return;}
                  
                if (result) [tmpArr addObject:result];
                  
                if (batch && !([tmpArr count]%50)) [self addQueueWithData:tmpArr final:NO];
            }];
        }
        else if (nil == group)
        {
            [self addQueueWithData:tmpArr final:YES];
        }
    }failureBlock:nil];
}
```

###取尾排序

每组批次的图片都加到一个串行queue队列里面等待排序, 某个批次的排序完成之后取当前queue最后一个(也就是最新过来的枚举图片)继续执行排序, 并清空当前的queue. 也就是在下面的sortMomentWithDate:final:函数里面调用cleanQueueAfterRoundOperation.
```
- (void)addQueueWithData:(NSMutableArray *)data final:(BOOL)final
{
    NSMutableArray *rawData = [NSMutableArray arrayWithArray:data];
    
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^
    {
        [self sortMomentWithDate:rawData final:final];
    }];
    
    [self.operQueue addOperation:op];
}

- (void)cleanQueueAfterRoundOperation
{
    if (self.operQueue == nil) return;
    
    if (self.operQueue.operationCount > 1)
    {
        NSArray *queueArr = self.operQueue.operations;
        NSMutableArray *opArr = [NSMutableArray arrayWithArray:queueArr];
        
        [opArr removeLastObject]; [opArr removeLastObject];
        [opArr makeObjectsPerformSelector:@selector(cancel)];
    }
}
```

###刷新CollectionView显示图片
中间批次按日期分类过的数据ready后, 在reloadData之前先判断一下当前用户是否在滑动collectionView, 如果是非scroll状态则刷新显示, 否则直接drop掉, 但是对于最后一批数据需要先存储着, 并在scrollViewDidEndDragging和scrollViewDidEndDecelerating里面判断, 一旦用户停止滑动了就立即刷新到collectionView上.
```
[[ImageDataAPI sharedInstance] getMomentsWithBatchReturn:YES
                                               ascending:NO
                                              completion:^(BOOL done, id obj)
{
    NSMutableArray *dArr = (NSMutableArray *)obj;
    
    if (dArr != nil && [dArr count])
    {
        if (!self.momentView.dragging && !self.momentView.decelerating)
        {
            dispatch_async(dispatch_get_main_queue(), ^
            {
                [self reloadWithData:dArr];
            });
        }
        else
        {
            if (done) {self.backupArr = dArr}
        }
    }
}];

- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate
{
    if (!decelerate && self.backupArr)
    {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self reloadWithData:self.backupArr];
            self.backupArr = nil; // done refresh
        });
    }
}

- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView
{
    if (self.backupArr)
    {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self reloadWithData:self.backupArr];
            self.backupArr = nil; // done refresh
        });
    }
}
```

###后续改进思路

按上述方案不管设备图片有多少, 基本可以正常打开相册并加载图片. 但还是有很多需要继续改进的地方: e.g. 
* 中间批次的数据ready后也可以先存储着, 待用户停止滑动后在reload上去, 而不是简单的drop掉. 
* 排序还需要再优化. 现在第一批50张图片排序后, 第二批进入排序的200张图片又需要重新分类排序, 中间批次数据只是为了先显示给用户看. 是不是第200张图片可以只对后面的150张进行排序, 也就是后面150张有新的日期, 则新建section, 相同日期直接insert到前面去. 这个还需要后面再研究...

以上只是自己的一些优化思路, 如果有更好的方案, 欢迎留言交流~~

***
###Photos.framework
iOS 8新引入了全新的PhotoKit API, 用来替代AssetsLibrary框架, PhotoKit提供了直接访问Moment数据的接口`+ (PHFetchResult<PHAssetCollection *> *)fetchMomentsWithOptions:(nullable PHFetchOptions *)options`该函数直接返回按日期分类的图片集合数据, 且速度非常快(猜想是不是Apple在用户拍摄图片或者导入图片时已mark日期信息并分类排序). 因此在iOS 8以上的系统可以直接采用PhotoKit框架来实现moment功能.
```
PHFetchOptions *options  = [[PHFetchOptions alloc] init];
options.sortDescriptors  = @[[NSSortDescriptor sortDescriptorWithKey:@"endDate"
                                                           ascending:ascending]];

PHFetchResult  *momentRes = [PHAssetCollection fetchMomentsWithOptions:options];
NSMutableArray *momArray  = [[NSMutableArray alloc] init];

for (PHAssetCollection *collection in momentRes)
{
    NSDateComponents *components = [[NSCalendar currentCalendar] components:NSDayCalendarUnit   |
                                                                            NSMonthCalendarUnit |
                                                                            NSYearCalendarUnit
                                                                   fromDate:collection.endDate];
    NSUInteger month = [components month];
    NSUInteger year  = [components year];
    NSUInteger day   = [components day];
    
    MomentCollection *moment = [MomentCollection new];
    moment.month = month; moment.year = year; moment.day = day;
    
    PHFetchOptions *option  = [[PHFetchOptions alloc] init];
    option.predicate = [NSPredicate predicateWithFormat:@"mediaType = %d", PHAssetMediaTypeImage];
    
    moment.assetObjs = [PHAsset fetchAssetsInAssetCollection:collection
                                                     options:option];
    if ([moment.assetObjs count]) [momArray addObject:moment];
}
```
So, 我们可以对外统一moment接口, 在内部(Gallery Model类)区分系统实现: iOS 7系统采用AssetsLibrary并使用上文的优化方案, iOS 8系统则直接调用Photos.framework的Moment接口.
 
但是这里面有个问题, AssetsLibrary的类型是ALAssetsGroup, 而PhotoKit的类型是PHFetchResult, 怎么在使用的时候统一呢? 难道还需要在外部调用的时候再区分一下系统么? 

解决方法很简单, 定义自己的数据类, 在数据结构内部再区分, 外部调用时使用的都是自己定义的数据类型:
e.g. 定义MomentCollection, 包括年月日信息, 对外的属性assetObjs则在内部区分系统并返回或设定相应的类型:
```
@interface MomentCollection : NSObject

@property (nonatomic, readwrite) NSUInteger     month;
@property (nonatomic, readwrite) NSUInteger     year;
@property (nonatomic, readwrite) NSUInteger     day;

@property (nonatomic, strong) id assetObjs;

@end

@property (nonatomic, strong) NSMutableArray *items;
@property (nonatomic, strong) PHFetchResult  *assets;

- (id)assetObjs
{
    return IS_IOS_8 ? self.assets : self.items;
}

- (void)setAssetObjs:(id)assetObjs
{
    if (IS_IOS_8)
    {
        self.assets = (PHFetchResult *)assetObjs;
    }
    else
    {
        self.items  = (NSMutableArray *)assetObjs;
    }
}
```
对于相册或者某个具体的图片也是类似的处理方法, 定义AlbumObj和PhotoObj数据类型. 这样外界(调用者)就不用管数据类型了, 所有的逻辑都在内部handle了...

另外, 对于对于其他功能, 比如相册的枚举, 相册Poster图片的获取, 图片URL的获取, 某个相册内所有thumbnail的获取等等都可以对外统一接口, 内部再区分是使用PhotoKit还是AssetsLibrary. 
```
- (void)getMomentsWithBatchReturn:(BOOL)batch // batch for iOS 7 only
                        ascending:(BOOL)ascending
                       completion:(void (^)(BOOL done, id obj))completion;

- (void)getThumbnailForAssetObj:(id)asset
                       withSize:(CGSize)size  // size for iOS 8 only
                     completion:(void (^)(BOOL ret, UIImage *image))completion;

- (void)getURLForAssetObj:(id)asset
                /*usingPH:(BOOL)usingPH*/
               completion:(void (^)(BOOL ret, NSURL *URL))completion;

- (void)getAlbumsWithCompletion:(void (^)(BOOL ret, id obj))completion;

- (void)getPosterImageForAlbumObj:(id)album
                       completion:(void (^)(BOOL ret, id obj))completion;

- (void)getPhotosWithGroup:(id)group
                completion:(void (^)(BOOL ret, id obj))completion;

- (void)getImageForPhotoObj:(id)asset
                   withSize:(CGSize)size
                 completion:(void (^)(BOOL ret, UIImage *image))completion;
```

完整的moment优化方案和PhotoKit/AssetsLibrary集成接口实现代码(**RJPhotoGallery**)已经上传到[GitHub](https://github.com/RylanJIN/RJPhotoGallery), 有兴趣的童鞋可以参考一下. 程序内封装的ImageDataAPI是图片加载的model类, 实现了Moment/Album功能, 有需要的可以直接copy过去使用.