#IOS-AutoLayout Constraint及动画使用

###对已有Constraint更新、删除: 

```swift
@IBOutlet weak var closeBtn: UIButton!
@IBOutlet weak var closebtnWidthCon: NSLayoutConstraint!
@IBOutlet weak var closebtnHeightCon: NSLayoutConstraint!
@IBOutlet weak var closebtnTopCon: NSLayoutConstraint!

private func testUpdateAndDelete(){
    self.closeBtn.setTranslatesAutoresizingMaskIntoConstraints(false)

    // 删除原有的Constraint
    if closebtnWidthCon != nil{
        self.closeBtn.removeConstraint(closebtnWidthCon)
    }
    if closebtnHeightCon != nil{
        self.closeBtn.removeConstraint(closebtnHeightCon)
    }
    // 更改closeBtn宽高
    closeBtn.snp_makeConstraints { (make) -> Void in
        make.width.equalTo(150)
        make.height.equalTo(150)
    }
    
    // 从父View中移除closeBtn top约束
    self.view.removeConstraint(closebtnTopCon)
    // 更改closeBtn距离顶部距离top
    self.closeBtn.snp_makeConstraints { (make) -> Void in
        make.top.equalTo(self.view).offset(150)
    }
}
```

###采用Constraint的View动画:
####更改UIView的Frame:
* 更改UIView的Frame只会影响当前UIView，不会影响跟该UIView有约束关系的其它UIView
```swift
private func testAnim(){
        UIView.animateWithDuration(2.0, delay: 0.2, options: .CurveEaseInOut, animations: { () -> Void in
            
            // 更改位置坐标
            self.centerLabel.center.y = 100
            self.centerLabel.center.x = 100
            
            // 旋转动画
            let rotation = CGAffineTransformMakeRotation(CGFloat(M_PI))
            self.centerLabel.transform = rotation
            
            // 缩放动画
            let scale = CGAffineTransformMakeScale(0.5, 0.5)
            self.centerLabel.transform = scale
            
            
            }) { (isFinished:Bool) -> Void in
                
        }
}
```
###更改UIView的Constraints

```swift
private func modifyConstraints(){
    // 修改已有约束，会引起其它相关UIView约束同步变化
    self.closebtnWidthCon.constant = 200
    self.closebtnHeightCon.constant = 200
    
    UIView.animateWithDuration(2.0, animations: { () -> Void in
        // 刷新所有相关约束的UIView
        self.closeBtn.layoutIfNeeded()
        self.testSwitch.layoutIfNeeded()
        
        }) { (isFinished) -> Void in
    
    }
}
```    


