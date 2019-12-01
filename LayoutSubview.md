# LayoutSubviews

### A->B->C 调用时机  
- 没有被添加到视图上的时候不会调用
- addsubview
    - A的frame不是zero，B的frame时zero，会调用A的，不会调用B的
    - A的frame是zero，B的frame也是zero。先调用A的，再调用B的
    - A和B的frame都不是zero的时候，同上！
    - 当A不是自己创建的，例如UIViewController的view,即使B的frame是zero，也会调用B的(猜测：应该是和UIViewController的生命周期有关)
    - 多层嵌套只会调用父视图和子视图两个，例如addC时，会调用B和C的layout，不会调用A的  
- 更改frame
    - 通过改变origin调用layoutsubview，最多5次，通过改变size调用，没有次数限制（所以会ANR）
    - 更改B的size, 先调用A的 再调用B的
    - 更改B的origin 先调用B的 再调用C的
    - origin和size都改  先调用A 再调用B 再调用C
    - 更改A的origin 先调用A 再调用B 再调用C
        - 结论：当更改size的时候，会调用父view和自己的，当更改origin的时候，会调用自己以及所有子视图的layout
    
### 结论   
- 触发layoutSubviews的操作不会立刻调用。（猜测：但是会立马对view进行标记）
- 但会在runloop休眠之前调用标记view的layoutSubview。（猜测：调用完成后会清除标记）    

### 实例
- 在A的layoutSubviews中更改B的size
    - 试图对A和B进行标记。
    - 因为此时A的标记没有清除，所以对A的标记失败。对B的成功。
    - 即在休眠前调用B的layoutSubviews，不会调用A的。
- 在B的layoutSubviews中又更改了自己的size
    - 试图对A和B进行标记。
    - 此时B的标记没有清除，对B标记失败，对A成功。
    - 即在休眠前调用A的layoutSubviews，不会调用B的。
- 此时产生循环调用（ANR)

