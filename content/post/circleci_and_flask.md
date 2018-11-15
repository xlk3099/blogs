---
title: "用circle ci 跟flask做自动化集成测试的一次经历"
date: 2018-11-14T16:35:40+08:00
draft: false
tags: ["CI/CD", "circle ci", "flask"]
---

### **需求**

最近手头工作的项目代码是放在公网github，作为private repo，采用的CI是circle ci，但需要跑的**集成测试**，是部署在AWS EC2上面，内网环境。

总结下需求:

* 每次当CIRCLE CI 成功build最新的代码时，需要向内网发送通知 build成功，使用最新的commit来跑集成测试。
* 内网收到通知，下载最新代码，编译，运行最新的集成测试代码。
* 生成测试报告，返回结果给CIRCLE CI。

第一直觉就是在内网搭一个CI/CD不就了事，可以找个EC2 instance，搭建一个circle ci或者jenkins服务即可。

但这次想着试试自己reinvent wheel，看看能不能自己写个服务来handle自动build测试部署等等。

### **Flask web 服务**

为啥用flask？跑个小服务而已，人生苦短，我用python。

flask服务需求：

* 能handle来自circle ci的request，兄台，XXXbuild结束，该跑它的测试了。
* 能pull + build 指定的代码
* 能运行集成测试
* 能显示测试报告
* 返回集成测试结果给circle ci
* 给bearychat推送测试结果

CIRCLE CI 能获取到当前build的`user`， `branch`， `commit id` 等信息。
集成测试是用pytest写的，pytest使用pytest-html plugin能自动生成报告。

所以主要是两个接口:
* POST. 用来handle build request，利用来自circl ci的project的信息， 用来pull，build最新commit的代码。
* GET. 用来handle 展示test report的request。


Python代码:
```python
@app.route('/report', methods=['GET'])
def test_report():
    """
    返回测试报告
    :return:
    """
    return send_from_directory(TEST_RESULT_DIR, "result.html")

@app.route('/circle/build', methods=['POST'])
def build():
    """
    接受build request
    :return:
    """
    req = request.get_json()
    user = req["user"]
    branch = req["branch"]
    commit = req["commit"]
    os.chdir(PROJECT_ROOT)  
    try:
        subprocess.run("make clean;\
                       git fetch --all;\
                       git checkout {0}".format(req['commit']), shell=True, check=True)
        subprocess.run("make build", shell=True, check=True)
        ethx_pid = start_ethx() # 启动项目服务
        pytest.main([PROJECT_TEST_PASH, "-Wi", "--html={0}/result.html".format(TEST_RESULT_DIR)])
        stop_ethx(ethx_pid)     # 停止项目
        send_notification(user, branch, commit) # 发送推送消息
        if result["failed"] > 0:
            return TEST_REPORT_URL
        else:
            return "success"
    except subprocess.CalledProcessError as err:
        print("subprocess error: {0}".format(err))
```

### **小结**
其实可以做的更复杂，比如可以

* 展示历史test result
* 监听多个project变动（我实际项目里是监听3个）
* 跟circle ci 互动，执行完毕启动下一个job
* 插件化

但越做越像jenkins，也没啥必要，不过这次经历还是不错的。