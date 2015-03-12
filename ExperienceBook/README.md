# 开发中出现的闭包循环总结
##错误一
``出错处``
```swift
// MARK: - 上拉刷新部分代码
    var parentView: UITableView?

    // 给 parentView 添加观察
    func addPullupOberserver(parentView: UITableView, pullupLoadData: ()->()) {

        // 1. 记录要观察的表格视图
        self.parentView = parentView

        // 2. 记录上拉加载数据的闭包
        self.pullupLoadData = pullupLoadData

        parentView.addObserver(self, forKeyPath: "contentOffset", options: NSKeyValueObservingOptions.New, context: nil)
    }

    // KVO 的代码
    deinit {
        parentView!.removeObserver(self, forKeyPath: "contentOffset")
    }
```
``正确的写法``
```swift
    // MARK: - 上拉刷新部分代码
    // parentView 默认是强引用，会对视图控制器中的tableView进行强引用
    // OC 中设置代理，代理必须要使用 weak
//    var parentView: UITableView?
    weak var parentView: UITableView?

    // 给 parentView 添加观察
    func addPullupOberserver(parentView: UITableView, pullupLoadData: ()->()) {

        // 1. 记录要观察的表格视图
        self.parentView = parentView

        // 2. 记录上拉加载数据的闭包
        self.pullupLoadData = pullupLoadData

        self.parentView!.addObserver(self, forKeyPath: "contentOffset", options: NSKeyValueObservingOptions.New, context: nil)
    }

    // KVO 的代码
    /***
        使用观察者的时候，被观察对象释放之前，一定要先注销观察者！
    */
    deinit {
        println("刷新视图 88")
        // 注意** 不一定写在 deinit 中就一定保险的
        // EXC_BAD_INSTRUCTION OC 中的野指针！
//        parentView!.removeObserver(self, forKeyPath: "contentOffset")
    }

```
##错误二
``出错处``
```swift
 override func viewDidLoad() {
        super.viewDidLoad()

        // 添加 tableView 的footer
        tableView.tableFooterView = pullupView

        loadData()

        // 让上拉刷新视图 监听 tableView 的 contentOffset 动作
        pullupView.addPullupOberserver(tableView) {
            println("上拉加载数据啦～～～～～")

            self.pullupView.isPullupLoading = false
//            self.pullupView.stopLoading()
        }
    }

    ///  加载微博数据
    /**
        Refresh控件高度是 60 点
    */
    @IBAction func loadData() {
        // 主动开始加载数据
        refreshControl?.beginRefreshing()

        StatusesData.loadStatus { (data, error) -> () in
            // 隐藏刷新控件
            self.refreshControl?.endRefreshing()

            if error != nil {
                println(error)
                SVProgressHUD.showInfoWithStatus("你的网络不给力")
                return
            }
            if data != nil {
                // 刷新表格数据
                self.statusData = data
                self.tableView.reloadData()
            }
        }
    }
}

///  表格数据源 & 代理扩展
extension HomeViewController: UITableViewDataSource, UITableViewDelegate {
    override func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return self.statusData?.statuses?.count ?? 0
    }

    ///  根据indexPath 返回微博数据&可重用标识符
    func cellInfo(indexPath: NSIndexPath) -> (status: Status, cellId: String) {
        let status = self.statusData!.statuses![indexPath.row]
        let cellId = StatusCell.cellIdentifier(status)

        return (status, cellId)
    }

    ///  准备表格的cell
    override func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {

        // 提取cell信息
        let info = cellInfo(indexPath)

        let cell = tableView.dequeueReusableCellWithIdentifier(info.cellId, forIndexPath: indexPath) as! StatusCell

        // 判断表格的闭包是否被设置
        if cell.photoDidSelected == nil {
            // 设置闭包
            cell.photoDidSelected = { (status: Status, photoIndex: Int)->() in
                println("\(status.text) \(photoIndex)")
                // 将数据传递给照片浏览器视图控制器
                // 使用类方法调用，不需要知道视图控制器太多的内部细节
                let vc = PhotoBrowserViewController.photoBrowserViewController()

                vc.urls = status.largeUrls
                vc.selectedIndex = photoIndex

                self.presentViewController(vc, animated: true, completion: nil)
            }
        }

        cell.status = info.status

        return cell
    }
```
``正确的写法``
```swift
    override func viewDidLoad() {
        super.viewDidLoad()

        // 添加 tableView 的footer，tableView会对 pullupView 强引用
        tableView.tableFooterView = pullupView

        loadData()

        // 让上拉刷新视图 监听 tableView 的 contentOffset 动作
        weak var weakSelf = self
        pullupView.addPullupOberserver(tableView) {
            println("上拉加载数据啦～～～～～")

            weakSelf!.pullupView.isPullupLoading = false
//            self.pullupView.stopLoading()
        }
    }

    deinit {
        println("主页视图控制器被释放!!!!!!")

        // 主动释放加载刷新视图对tableView的观察
        tableView.removeObserver(pullupView, forKeyPath: "contentOffset")
    }

    ///  加载微博数据
    /**
        Refresh控件高度是 60 点
    */
    @IBAction func loadData() {
        // 主动开始加载数据
        refreshControl?.beginRefreshing()

        weak var weakSelf = self
        StatusesData.loadStatus { (data, error) -> () in
            // 隐藏刷新控件
            weakSelf!.refreshControl?.endRefreshing()

            if error != nil {
                println(error)
                SVProgressHUD.showInfoWithStatus("你的网络不给力")
                return
            }
            if data != nil {
                // 刷新表格数据
                weakSelf!.statusData = data
                weakSelf!.tableView.reloadData()
            }
        }
    }
}

///  表格数据源 & 代理扩展
extension HomeViewController: UITableViewDataSource, UITableViewDelegate {
    override func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return self.statusData?.statuses?.count ?? 0
    }

    ///  根据indexPath 返回微博数据&可重用标识符
    func cellInfo(indexPath: NSIndexPath) -> (status: Status, cellId: String) {
        let status = self.statusData!.statuses![indexPath.row]
        let cellId = StatusCell.cellIdentifier(status)

        return (status, cellId)
    }

    ///  准备表格的cell
    override func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {

        // 提取cell信息
        let info = cellInfo(indexPath)

        let cell = tableView.dequeueReusableCellWithIdentifier(info.cellId, forIndexPath: indexPath) as! StatusCell

        // 判断表格的闭包是否被设置
        if cell.photoDidSelected == nil {
            // 设置闭包
            weak var weakSelf = self
            cell.photoDidSelected = { (status: Status, photoIndex: Int)->() in
                println("\(status.text) \(photoIndex)")
                // 将数据传递给照片浏览器视图控制器
                // 使用类方法调用，不需要知道视图控制器太多的内部细节
                let vc = PhotoBrowserViewController.photoBrowserViewController()

                vc.urls = status.largeUrls
                vc.selectedIndex = photoIndex

                weakSelf!.presentViewController(vc, animated: true, completion: nil)
            }
        }

        cell.status = info.status

        return cell
    }
```
