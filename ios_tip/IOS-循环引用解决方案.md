#循环引用解决方案：

###Block中循环引用问题：
* Block调用所在对象的方法或变量时，会带来循环引用的问题，继而引发内存泄露.
* 可以通过weak声明一个当前对象的弱引用来调用方法。

```swift
class LoginViewController: UIViewController {
    
    private weak var weakSelf:LoginViewController?//弱引用

    override func viewDidLoad() {
        super.viewDidLoad()

        weakSelf = self
        
        ParametersProxyFactory.getProxy().getAccountManager().login("", password: "") 
        { (userInfo) -> Void in
            
            weakSelf?.printlnData(userInfo)
        }                
    }
    
     private func printlnData(userInfo:UserInfo?){
        println(userInfo)
    }
}
```