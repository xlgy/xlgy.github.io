---
layout:     post
title:      iOS开发记录——UILabel 文字垂直方向 置顶/置底 实现
subtitle:   iOS开发记录——UILabel 文字垂直方向 置顶/置底 实现date:       2019-05-05
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - iOS
---


iOS UILabel只提供了水平方向的文字位置设置，也就是我们常用的textAlignment属性(NSTextAlignmentLeft、NSTextAlignmentCenter、NSTextAlignmentRight)，但是**垂直方向并没有提供响应的API**给我们使用。我们经常碰到这样的需求，希望**文字向上置顶，或者向下置底**，这种场景我们如何实现呢？


------------

这里介绍一个我平时常用的小技巧，虽然不是很高级，但是用起来还可以，哈哈!!

利用 **分类(category）**为UILabel添加属性textVerticalAlignType来控制文字是垂直方向位置类型(置顶、中间、置底)

**实现原理**：利用往文字后面活前面下面添加”\n”来实现文字填充满整个UILable控件实现置顶/置顶效果

### 1、新建分类(UILabel+TextAlign)，添加枚举类型属性textVerticalAlignType

```
#import <UIKit/UIKit.h>

NS_ASSUME_NONNULL_BEGIN

@interface UILabel (TextAlign)

typedef NS_ENUM(NSInteger,TextVerticalAlignType){
    ///>垂直居中
    TextVerticalAlignTypeCenter,
    ///>垂直置顶
    TextVerticalAlignTypeTop,
    ///>垂直置底
    TextVerticalAlignTypeBottom,
};


@property(nonatomic,assign) TextVerticalAlignType textVerticalAlignType;


@end

NS_ASSUME_NONNULL_END
```


熟悉iOS的runtime都知道，给分类添加属性不能直接添加，需要重写set/get方法，并使用对象相关联保存值


```

static char * textVerticalAlignTypeKey = "textVerticalAlignTypeKey";

- (TextVerticalAlignType)textVerticalAlignType{
    NSNumber *num = objc_getAssociatedObject(self, &textVerticalAlignTypeKey);
    return num.integerValue;
}


- (void)setTextVerticalAlignType:(TextVerticalAlignType)textVerticalAlignType{
    objc_setAssociatedObject(self, &textVerticalAlignTypeKey, @(textVerticalAlignType), OBJC_ASSOCIATION_ASSIGN);
}
```

### 2、更新文本内容，实现置顶/置底

```

-(void)upadteTextAlignType{
    if (self.textVerticalAlignType == TextVerticalAlignTypeCenter) {
        return;
    }else{
        CGSize fontSize = [self.text sizeWithAttributes:@{NSFontAttributeName:self.font }];
        //控件的高度除以一行文字的高度
        int num = self.frame.size.height/fontSize.height;
        //计算需要添加换行符个数
        long newLinesToPad = num - self.numberOfLines;
        self.numberOfLines = 0;

        if (self.textVerticalAlignType == TextVerticalAlignTypeTop) {
            for(int i=0; i<newLinesToPad; i++){
                //在文字后面添加换行符"/n"
                self.text = [self.text stringByAppendingString:@"\n"];
            }
            return;
        }

        if (self.textVerticalAlignType == TextVerticalAlignTypeBottom) {
            for(int i=0; i<newLinesToPad; i++){
                //在文字前面添加换行符"/n"
                self.text = [NSString stringWithFormat:@" \n%@",self.text];
            }
            return;
        }
    }
}
```
在setTextVerticalAlignType方法中，调用upadteTextAlignType，遍实现置顶/置底效果。



### 3、测试代码及效果

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    
    UILabel *lb = [UILabel new];
    lb.text = @"测试文字";
    lb.textAlignment = NSTextAlignmentCenter;
    lb.frame = CGRectMake(100, 100, 200, 200);
    lb.backgroundColor = [UIColor yellowColor];
    
    lb.textVerticalAlignType = TextVerticalAlignTypeTop;
    [self.view addSubview:lb];
}
```


![置顶](https://youke2.picui.cn/s1/2025/12/13/693d2fd51db3d.png)

```
lb.textVerticalAlignType = TextVerticalAlignTypeBottom;
```

![置底](https://youke2.picui.cn/s1/2025/12/13/693d2fd52c1b4.png)
