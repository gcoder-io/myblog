##添加点击事件：
* NSButton添加点击事件：

```objective-c
- (void) setListener:(LoginView *)loginView{
    [loginView.closeButton addTarget:self action:@selector(closeButtonClickAction:) forControlEvents:UIControlEventTouchUpInside];
}

- (void) closeButtonClickAction:(id)sender{
    NSLog(@"closeButtonClickAction=====>");
    [self.navigationController popViewControllerAnimated:YES];
    //[self dismissViewControllerAnimated:NO completion:^{}];
}
```
* UIView添加点击事件：

```objective-c
- (void) setListener:(MainView *)mainView{
    UITapGestureRecognizer *tapGestureTel = [[UITapGestureRecognizer alloc]initWithTarget:self action:@selector(testTextClickAction:)];
    mainView.testText.userInteractionEnabled = YES;
    [mainView.testText addGestureRecognizer:tapGestureTel];
}

- (void) testTextClickAction:(UITapGestureRecognizer *)recognizer{
    NSLog(@"testClick=======>");
}
```
