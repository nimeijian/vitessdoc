本页解释如何在[Kubernetes](http://kubernetes.io)集群上运行Vitess。同时还提供了[Google Container Engine](https://cloud.google.com/container-engine/)启动Kubernetes集群的使用步骤。

如果你在其他平台上已经有Kubernetes v1.0+版本的支持，你可以跳过`gcloud`步骤。 `kubectl`可以应用于任何Kubernetes集群。

## 先决条件

为了更好的使用本指南，您必须在本地安装Go 1.7+，Vitess的`vtctlclient`工具和Google Cloud SDK。以下部分说明如何在您的环境中进行配置。
其中，Google Cloud SDK不是非必需的，如果您在自有环境中搭建Kubernetes，那么就可以忽略Google Cloud SDK。

### 安装Go 1.7+

您需要安装[Go 1.7+](http://golang.org/doc/install)才能编译安装`vtctlclient`工具， vtctlclient可以向Vitess发送相关管理命令。

go环境安装完成后，请确保您的环境变量中`GOPATH`指向您工作目录的根目录。最常用的设置是`GOPATH=$HOME/go`，
同时需要确保该非root用户对该目录具有读写权限。


另外，请确保把`$GOPATH/bin`路径增加到环境变量`$PATH`中；有关go工作路径的更多信息可以参阅[如何使用go语言开发](http://golang.org/doc/code.html#Organization)。

### 编译安装vtctlclient

`vtctlclient`工具可以用来向Vitess发送命令。
``` sh
$ go get github.com/youtube/vitess/go/cmd/vtctlclient
```
该命令在以下目录下载并且编译了Vitess源码：
``` sh
$GOPATH/src/github.com/youtube/vitess/
```

同时他也会把编译好的`vtctlclient`二进制文件拷贝到目录`$GOPATH/bin`下。

### 设置Google Compute Engine, Container Engine和Cloud工具

**注意:** 如果您在别处运行Kubernetes， 那么请跳转到[本地kubectl](#locate-kubectl)。

为了使用GCE在Kubernetes上运行Vitess， 我们必须有一个GCE账户来计费。下面就说明在Google Developers Console中构建项目如何开启计费功能以及如何关联账户。

1. 登陆到谷歌开发者中心来[开启计费](https://console.developers.google.com/billing)。
    1.  Click the **Billing** pane if you are not there already.
    1.  Click **New billing account**.
    1.  Assign a name to the billing account -- e.g. "Vitess on
        Kubernetes." Then click **Continue**. You can sign up
        for the [free trial](https://cloud.google.com/free-trial/)
        to avoid any charges.

1.  Create a project in the Google Developers Console that uses
    your billing account:
    1.  At the top of the Google Developers Console, click the **Projects** dropdown.
    1.  Click the Create a Project... link.
    1.  Assign a name to your project. Then click the **Create** button.
        Your project should be created and associated with your
        billing account. (If you have multiple billing accounts,
        confirm that the project is associated with the correct account.)
    1.  After creating your project, click **API Manager** in the left menu.
    1.  Find **Google Compute Engine** and **Google Container Engine API**.
        (Both should be listed under "Google Cloud APIs".)
        For each, click on it, then click the **"Enable API"** button.

1.  Follow the [Google Cloud SDK quickstart instructions]
    (https://cloud.google.com/sdk/#Quick_Start) to set up
    and test the Google Cloud SDK. You will also set your default project
    ID while completing the quickstart.

    **Note:** If you skip the quickstart guide because you've previously set up
    the Google Cloud SDK, just make sure to set a default project ID by running
    the following command. Replace `PROJECT` with the project ID assigned to
    your [Google Developers Console](https://console.developers.google.com/)
    project. You can [find the ID]
    (https://cloud.google.com/compute/docs/projects#projectids)
    by navigating to the **Overview** page for the project in the Console.

    ``` sh
    $ gcloud config set project PROJECT
    ```

1.  安装或者更新`kubectl`工具:
    ``` sh
    $ gcloud components update kubectl
    ```

### 本地kubectl
  检查`kubectl`是否已经正常安装并设置环境变量`PATH`:

  ``` sh
  $ which kubectl
  ### example output:
  # /usr/local/bin/kubectl
  ```

  如果`kubectl`不在您的`PATH`上，您可以通过设置`KUBECTL`环境变量来告诉脚本在哪里可以找到它：

  ``` sh
  $ export KUBECTL=/usr/local/bin/kubectl
  ```

## 启动容器集群

**注意:** 如果您在其他地方运行Kubernetes，请跳到[启动Vitess集群](#start-a-vitess-cluster)。
1. 设置[zone][zone](https://cloud.google.com/compute/docs/zones#overview)。
    使用以下命令安装：
    ``` sh
    $ gcloud config set compute/zone us-central1-b
    ```

1.  创建容器引擎集群：
    ``` sh
    $ gcloud container clusters create example --machine-type n1-standard-4 --num-nodes 5 --scopes storage-rw
    ### example output:
    # Creating cluster example...done.
    # Created [https://container.googleapis.com/v1/projects/vitess/zones/us-central1-b/clusters/example].
    # kubeconfig entry generated for example.
    ```

    **注意:** The `--scopes storage-rw` argument is necessary to allow
    [built-in backup/restore](http://vitess.io/user-guide/backup-and-restore.html)
    to access [Google Cloud Storage](https://cloud.google.com/storage/).

1.  创建一个云存储bucket:

    为了给backups使用云存储插件，首先需要创建一个[bucket](https://cloud.google.com/storage/docs/concepts-techniques#concepts)用来存储Vitess备份数据。详细信息可以查阅[bucket naming guidelines](https://cloud.google.com/storage/docs/bucket-naming)。

    ``` sh
    $ gsutil mb gs://my-backup-bucket
    ```

## 启动Vitess集群

1.  **跳转到本地Vitess源代码目录**

    此目录在安装`vtctlclient`的时候已经被创建成功：

    ``` sh
    $ cd $GOPATH/src/github.com/youtube/vitess/examples/kubernetes
    ```

2.  **配置本地站点设置**

    运行`configure.sh`脚本来生成`config.sh`文件，`config.sh`用于自定义您的集群设置.

    目前，我们对于在[Google云存储](https://cloud.google.com/storage/)的备份(http://vitess.io/user-guide/backup-and-restore.html)提供了现成的支持。
    如果使用GCS, 请填写configure脚本中所需要字段， 包括上面创建bucket的名称。

    ``` sh
    vitess/examples/kubernetes$ ./configure.sh
    ### example output:
    # Backup Storage (file, gcs) [gcs]:
    # Google Developers Console Project [my-project]:
    # Google Cloud Storage bucket for Vitess backups: my-backup-bucket
    # Saving config.sh...
    ```

    对于其他平台，您需要选择`文件`备份存储插件，并在`vttablet`和`vtctld` pod中安装一个读写网络卷。例如：您可以通过NFS将任何存储服务mount到Kubernetes中，然后在这里提供configure脚本的安装路径。

    可以通过实现[Vitess BackupStorage插件](https://github.com/youtube/vitess/blob/master/go/vt/mysqlctl/backupstorage/interface.go)来添加对其他云存储（如Amazon S3）的支持。如果您有其他定制的插件请求，请在[论坛](https://groups.google.com/forum/#!forum/vitess)上告诉我们。

3.  **启动etcd集群**
    Vitess[拓扑服务](http://vitess.io/overview/concepts.html#topology-service)存储Vitess集群中所有服务器的协作数据， 他可以将此数据存储在几个一致性存储系统中。本例中我们使用[etcd](https://github.com/coreos/etcd)来存储，注意，我们需要自己的etcd集群，与Kubernetes本身使用的集群分开。

    ``` sh
    vitess/examples/kubernetes$ ./etcd-up.sh
    ### example output:
    # Creating etcd service for global cell...
    # service "etcd-global" created
    # service "etcd-global-srv" created
    # Creating etcd replicationcontroller for global cell...
    # replicationcontroller "etcd-global" created
    # ...
    ```

    这条命令创建了两个集群， 一个是[全局数据中心](/user-guide/topology-service.html#global-vs-local)集群，另一个是[本地数据中心](http://vitess.io/overview/concepts.html#cell-data-center)集群。你可以通过运行以下命令来检查群集中[pods](http://kubernetes.io/v1.1/docs/user-guide/pods.html)的状态:

    ``` sh
    $ kubectl get pods
    ### example output:
    # NAME                READY     STATUS    RESTARTS   AGE
    # etcd-global-8oxzm   1/1       Running   0          1m
    # etcd-global-hcxl6   1/1       Running   0          1m
    # etcd-global-xupzu   1/1       Running   0          1m
    # etcd-test-e2y6o     1/1       Running   0          1m
    # etcd-test-m6wse     1/1       Running   0          1m
    # etcd-test-qajdj     1/1       Running   0          1m
    ```
    Kubernetes节点第一下下载需要的Docker镜像的时候会耗费较长的时间， 在下载镜像的过程中Pod的状态是Pending状态。

    **注意:** 本例中, 每个以`-up.sh`结尾的脚本都有一个以`-down.sh`结尾的脚本相对应。 你可以用来停止Vitess集群中的某些组件，而不会关闭整个集群；例如：移除`etcd`的部署可以使用一下命令：

    ``` sh
    vitess/examples/kubernetes$ ./etcd-down.sh
    ```

4.  **启动vtctld**
    `vtctld`提供了检查Vitess集群状态的接口， 同时还可以接收`vtctlclient`的RPC命令来修改集群信息。

    ``` sh
    vitess/examples/kubernetes$ ./vtctld-up.sh
    ### example output:
    # Creating vtctld ClusterIP service...
    # service "vtctld" created
    # Creating vtctld replicationcontroller...
    # replicationcontroller "vtctld" create createdd
    ```

5.  **使用vtctld web界面**

    在Kubernetes外面使用vtctld需要使用[kubectl proxy]
    (http://kubernetes.io/v1.1/docs/user-guide/kubectl/kubectl_proxy.html)在工作站上创建一个通道。

    **注意:** proxy命令是运行在前台， 所以如果你想启动proxy需要另外开启一个终端。

    ``` sh
    $ kubectl proxy --port=8001
    ### example output:
    # Starting to serve on localhost:8001
    ```

    你可以在`本地`打开vtctld web界面:

    http://localhost:8001/api/v1/proxy/namespaces/default/services/vtctld:web/

    同时，还可以通过proxy进入[Kubernetes Dashboard]
    (http://kubernetes.io/v1.1/docs/user-guide/ui.html), 监控nodes, pods和服务器状态:

    http://localhost:8001/ui

6.  **使用 vtctlclient 向vtctld发送命令**

    现在就可以通过在本地运行`vtctlclient`向Kubernetes集群中的`vtctld`发送命令。

    为了开启RPC访问Kubernetes集群，我们将再次使用`kubectl`设置一个验证的隧道。
    和HTTP代理不同的是， HTTP代理我们使用Web界面访问， 这次我们使用[端口转发](http://kubernetes.io/v1.1/docs/user-guide/kubectl/kubectl_port-forward.html)来访问vtctld的gRPC端口。

    由于通道需要针对特定​​的vtctld pod名称，所以我们提供了`kvtctl.sh`脚本，它使用`kubectl`命令来查找pod名称并且在运行`vtctlclient`之前设置通道。

    现在运行`kvtctl.sh help`可以测试到`vtctld`的连接，同时还会列出管理Vitess集群的`vtctlclient`命令。

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh help
    ### example output:
    # Available commands:
    #
    # Tablets:
    #   InitTablet ...
    # ...
    ```
    可以使用`help`获取每个命令的详细信息：

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh help ListAllTablets
    ```
    有关`vtctl help`输出的web格式版本，请参阅[vtctl参考](http://vitess.io/reference/vtctl.html)

7.  **启动vttablets**

    [tablet](http://vitess.io/overview/concepts.html#tablet)是Vitess扩展的基本单位。tablet由运行在相同的机器上的`vttablet` 和 `mysqld`组成。
    我们在用Kubernetes的时候通过将vttablet和mysqld的容器放在单个[pod](http://kubernetes.io/v1.1/docs/user-guide/pods.html)中来实现耦合。

    运行以下脚本以启动vttablet pod，其中也包括mysqld：

    ``` sh
    vitess/examples/kubernetes$ ./vttablet-up.sh
    ### example output:
    # Creating test_keyspace.shard-0 pods in cell test...
    # Creating pod for tablet test-0000000100...
    # pod "vttablet-100" created
    # Creating pod for tablet test-0000000101...
    # pod "vttablet-101" created
    # Creating pod for tablet test-0000000102...
    # pod "vttablet-102" created
    # Creating pod for tablet test-0000000103...
    # pod "vttablet-103" created
    # Creating pod for tablet test-0000000104...
    # pod "vttablet-104" created
    ```
    启动后在vtctld Web管理界面中很快就会看到一个名为`test_keyspace`的[keyspace](http://vitess.io/overview/concepts.html#keyspace)，其中有一个名为`0`的分片。点击分片名称可以查看
    tablets列表。当5个tablets全部显示在分片状态页面上，就可以继续下一步操作。注意，当前状态tablets不健康是正常的，因为在tablets上面还没有初始化数据库。

    tablets第一次创建的时候， 如果pod对应的node上尚未下载对应的[Vitess镜像](https://hub.docker.com/u/vitess/)文件，那么创建就需要花费较多的时间。同样也可以通过命令行使用`kvtctl.sh`查看tablets的状态。

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh ListAllTablets test
    ### example output:
    # test-0000000100 test_keyspace 0 spare 10.64.1.6:15002 10.64.1.6:3306 []
    # test-0000000101 test_keyspace 0 spare 10.64.2.5:15002 10.64.2.5:3306 []
    # test-0000000102 test_keyspace 0 spare 10.64.0.7:15002 10.64.0.7:3306 []
    # test-0000000103 test_keyspace 0 spare 10.64.1.7:15002 10.64.1.7:3306 []
    # test-0000000104 test_keyspace 0 spare 10.64.2.6:15002 10.64.2.6:3306 []
    ```

8.  **初始化MySQL数据库**

    一旦所有的tablets都启动完成， 我们就可以初始化底层数据库了。

    **注意:** 许多`vtctlclient`命令在执行成功时不返回任何输出。

    首先，指定tablets其中一个作为初始化的master。Vitess会自动连接其他slaves的mysqld实例，以便他们开启从master复制数据的mysqld进程； 默认数据库创建也是如此。 因为我们的keyspace名称为`test_keyspace`，所以MySQL的数据库会被命名为`vt_test_keyspace`。
    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh InitShardMaster -force test_keyspace/0 test-0000000100
    ### example output:
    # master-elect tablet test-0000000100 is not the shard master, proceeding anyway as -force was used
    # master-elect tablet test-0000000100 is not a master in the shard, proceeding anyway as -force was used
    ```

    **注意:** 因为分片是第一次启动， tablets还没有准备做任何复制操作， 也不存在master。如果分片不是一个全新的分片，`InitShardMaster`命令增加`-force`标签可以绕过应用的健全检查。

    tablets更新完成后，你可以看到一个**master**, 多个 **replica** 和 **rdonly** tablets:

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh ListAllTablets test
    ### example output:
    # test-0000000100 test_keyspace 0 master 10.64.1.6:15002 10.64.1.6:3306 []
    # test-0000000101 test_keyspace 0 replica 10.64.2.5:15002 10.64.2.5:3306 []
    # test-0000000102 test_keyspace 0 replica 10.64.0.7:15002 10.64.0.7:3306 []
    # test-0000000103 test_keyspace 0 rdonly 10.64.1.7:15002 10.64.1.7:3306 []
    # test-0000000104 test_keyspace 0 rdonly 10.64.2.6:15002 10.64.2.6:3306 []
    ```

    **replica** tablets通常用于提供实时网络流量, 而 **rdonly** tablets通常用于离线处理, 例如批处理作业和备份。
    每个[tablet type](http://vitess.io/overview/concepts.html#tablet)的数量可以在配置脚本`vttablet-up.sh`中配置。

9.  **创建表**

    `vtctlclient`命令可以跨越keyspace里面的所有tablets来应用数据库变更。以下命令创建定义在文件`create_test_table.sql`中的表：

    ``` sh
    # Make sure to run this from the examples/kubernetes dir, so it finds the file.
    vitess/examples/kubernetes$ ./kvtctl.sh ApplySchema -sql "$(cat create_test_table.sql)" test_keyspace
    ```

    创建表的SQL如下所示：

    ``` sql
    CREATE TABLE messages (
      page BIGINT(20) UNSIGNED,
      time_created_ns BIGINT(20) UNSIGNED,
      message VARCHAR(10000),
      PRIMARY KEY (page, time_created_ns)
    ) ENGINE=InnoDB
    ```

    我们可以通过运行此命令来确认在给定的tablet是否创建成功，`test-0000000100`是`ListAllTablets`命令显示
    tablet列表其中一个tablet的别名：

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh GetSchema test-0000000100
    ### example output:
    # {
    #   "DatabaseSchema": "CREATE DATABASE `{{.DatabaseName}}` /*!40100 DEFAULT CHARACTER SET utf8 */",
    #   "TableDefinitions": [
    #     {
    #       "Name": "messages",
    #       "Schema": "CREATE TABLE `messages` (\n  `page` bigint(20) unsigned NOT NULL DEFAULT '0',\n  `time_created_ns` bigint(20) unsigned NOT NULL DEFAULT '0',\n  `message` varchar(10000) DEFAULT NULL,\n  PRIMARY KEY (`page`,`time_created_ns`)\n) ENGINE=InnoDB DEFAULT CHARSET=utf8",
    #       "Columns": [
    #         "page",
    #         "time_created_ns",
    #         "message"
    #       ],
    # ...
    ```

10.  **执行备份**

    现在， 数据库初始化已经应用， 是执行第一次[备份](http://vitess.io/user-guide/backup-and-restore.html)的最佳时间。在他们连上master并且复制之前， 这个备份将用于自动还原运行的任何其他副本。

    如果一个已经存在的tablet出现故障，并且没有备份数据， 那么他将会自动从最新的备份恢复并且恢复复制。

    选择其中一个 **rdonly** tablets并且执行备份。因为在数据复制期间创建一致性快照tablet会暂停复制并且停止服务，所以我们使用 **rdonly** tablet代替 **replica**。
    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh Backup test-0000000104
    ```

    After the backup completes, you can list available backups for the shard:

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh ListBackups test_keyspace/0
    ### example output:
    # 2015-10-21.042940.test-0000000104
    ```

11. **初始化Vitess路由**  

    在本例中， 我们只使用了没有特殊配置的单个数据库。因此，我们只需要确保当前(空)配置处于服务状态。
    我们可以通过运行以下命令完成：

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh RebuildVSchemaGraph
    ```

    （因为在运行，此命令将不显示任何输出。）

12.  **启动vtgate**

    Vitess通过使用[vtgate](http://vitess.io/overview/#vtgate)路由每个客户端的查询到正确的`vttablet`。
    在KubernetesIn中`vtgate`服务将连接分发到一个`vtgate`pods池中。pods由[replication controller](http://kubernetes.io/v1.1/docs/user-guide/replication-controller.html)来制定。

    ``` sh
    vitess/examples/kubernetes$ ./vtgate-up.sh
    ### example output:
    # Creating vtgate service in cell test...
    # service "vtgate-test" created
    # Creating vtgate replicationcontroller in cell test...
    # replicationcontroller "vtgate-test" created
    ```

## 使用客户端app测试集群

The GuestBook app in the example is ported from the
[Kubernetes GuestBook example](https://github.com/kubernetes/kubernetes/tree/master/examples/guestbook-go).
The server-side code has been rewritten in Python to use Vitess as the storage
engine. The client-side code (HTML/JavaScript) has been modified to support
multiple Guestbook pages, which will be useful to demonstrate Vitess sharding in
a later guide.

``` sh
vitess/examples/kubernetes$ ./guestbook-up.sh
### example output:
# Creating guestbook service...
# services "guestbook" created
# Creating guestbook replicationcontroller...
# replicationcontroller "guestbook" created
```

As with the `vtctld` service, by default the GuestBook app is not accessible
from outside Kubernetes. In this case, since this is a user-facing frontend,
we set `type: LoadBalancer` in the GuestBook service definition,
which tells Kubernetes to create a public
[load balancer](http://kubernetes.io/v1.1/docs/user-guide/services.html#type-loadbalancer)
using the API for whatever platform your Kubernetes cluster is in.

You also need to [allow access through your platform's firewall]
(http://kubernetes.io/v1.1/docs/user-guide/services-firewalls.html).

``` sh
# For example, to open port 80 in the GCE firewall:
$ gcloud compute firewall-rules create guestbook --allow tcp:80
```

**Note:** For simplicity, the firewall rule above opens the port on **all**
GCE instances in your project. In a production system, you would likely
limit it to specific instances.

Then, get the external IP of the load balancer for the GuestBook service:

``` sh
$ kubectl get service guestbook
### example output:
# NAME        CLUSTER-IP      EXTERNAL-IP     PORT(S)   AGE
# guestbook   10.67.242.247   3.4.5.6         80/TCP    1m
```

If the `EXTERNAL-IP` is still empty, give it a few minutes to create
the external load balancer and check again.

Once the pods are running, the GuestBook app should be accessible
from the load balancer's external IP. In the example above, it would be at
`http://3.4.5.6`.

You can see Vitess' replication capabilities by opening the app in
multiple browser windows, with the same Guestbook page number.
Each new entry is committed to the master database.
In the meantime, JavaScript on the page continuously polls
the app server to retrieve a list of GuestBook entries. The app serves
read-only requests by querying Vitess in 'replica' mode, confirming
that replication is working.

You can also inspect the data stored by the app:

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT * FROM messages"
### example output:
# +------+---------------------+---------+
# | page |   time_created_ns   | message |
# +------+---------------------+---------+
# |   42 | 1460771336286560000 | Hello   |
# +------+---------------------+---------+
```

The [GuestBook source code]
(https://github.com/youtube/vitess/tree/master/examples/kubernetes/guestbook)
provides more detail about how the app server interacts with Vitess.

## 测试Vitess resharding

现在你有一个完整的Vitess堆栈运行，你可能想继续按照[Sharding in Kubernetes](http://vitess.io/user-guide/sharding-kubernetes.html)测试[dynamic resharding](http://vitess.io/user-guide/sharding.html#resharding)。

如果是这样， 你可以跳过关闭和清理， 因为sharding指南可以直接跳转查阅。 如果不是请继续下面的关闭和清理。

## 关闭和清理

在停止Container Engine集群之前，应该删除Vitess服务。对于那些服务Kubernetes会负责清理它创建的任何实体
， 比如外部负载均衡。

``` sh
vitess/examples/kubernetes$ ./guestbook-down.sh
vitess/examples/kubernetes$ ./vtgate-down.sh
vitess/examples/kubernetes$ ./vttablet-down.sh
vitess/examples/kubernetes$ ./vtctld-down.sh
vitess/examples/kubernetes$ ./etcd-down.sh
```

Then tear down the Container Engine cluster itself, which will stop the virtual
machines running on Compute Engine:

``` sh
$ gcloud container clusters delete example
```

It's also a good idea to remove any firewall rules you created, unless you plan
to use them again soon:

``` sh
$ gcloud compute firewall-rules delete guestbook
```

## 故障排除

### 服务日志

如果一个pod进入`Running`状态，但是服务并没有按照预期响应。 那么我可以通过使用`kubectl logs`命令来查看
pod输出：

``` sh
# show logs for container 'vttablet' within pod 'vttablet-100'
$ kubectl logs vttablet-100 vttablet

# show logs for container 'mysql' within pod 'vttablet-100'
# Note that this is NOT MySQL error log.
$ kubectl logs vttablet-100 mysql
```

发布日志在某个地方，并且发送一个链接到[Vitess邮件列表](https://groups.google.com/forum/#!forum/vitess)获
得更多的帮助。

### Shell访问

If you want to poke around inside a container, you can use `kubectl exec` to run
a shell.

For example, to launch a shell inside the `vttablet` container of the
`vttablet-100` pod:

``` sh
$ kubectl exec vttablet-100 -c vttablet -t -i -- bash -il
root@vttablet-100:/# ls /vt/vtdataroot/vt_0000000100
### example output:
# bin-logs   innodb                  my.cnf      relay-logs
# data       memcache.sock764383635  mysql.pid   slow-query.log
# error.log  multi-master.info       mysql.sock  tmp
```

### Root证书

If you see in the logs a message like this:

```
x509: failed to load system roots and no roots provided
```

It usually means that your Kubernetes nodes are running a host OS
that puts root certificates in a different place than our configuration
expects by default (for example, Fedora). See the comments in the
[etcd controller template](https://github.com/youtube/vitess/blob/master/examples/kubernetes/etcd-controller-template.yaml)
for examples of how to set the right location for your host OS.
You'll also need to adjust the same certificate path settings in the
`vtctld` and `vttablet` templates.

### vttablets状态页面

Each `vttablet` serves a set of HTML status pages on its primary port.
The `vtctld` interface provides a **STATUS** link for each tablet.

If you access the vtctld web UI through the kubectl proxy as described above,
it will automatically link to the vttablets through that same proxy,
giving you access from outside the cluster.

You can also use the proxy to go directly to a tablet. For example,
to see the status page for the tablet with ID `100`, you could navigate to:

http://localhost:8001/api/v1/proxy/namespaces/default/pods/vttablet-100:15002/debug/status

### 直连mysqld

Since the `mysqld` within the `vttablet` pod is only meant to be accessed
via vttablet, our default bootstrap settings only allow connections from
localhost.

If you want to check or manipulate the underlying mysqld, you can issue
simple queries or commands through `vtctlclient` like this:

``` sh
# Send a query to tablet 100 in cell 'test'.
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT VERSION()"
### example output:
# +------------+
# | VERSION()  |
# +------------+
# | 5.7.13-log |
# +------------+
```

If you need a truly direct connection to mysqld, you can [launch a shell]
(#shell-access) inside the mysql container, and then connect with the `mysql`
command-line client:

``` sh
$ kubectl exec vttablet-100 -c mysql -t -i -- bash -il
root@vttablet-100:/# export TERM=ansi
root@vttablet-100:/# mysql -S /vt/vtdataroot/vt_0000000100/mysql.sock -u vt_dba
```
