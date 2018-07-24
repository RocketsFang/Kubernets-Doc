# 预备开始

## 步骤概览
1， 克隆这个代码仓库
2， 对代码做改动并且提交他们
3， 制作容器你也可以docker gradle插件来根据仓库中的文件制作一个容器亦或者使用例子Dockerfile制作你自己的docker容器
4， 将这个新容器下载到本地，然后执行并且测试

## 使用ECM小组的制品及其工作流程的例子
1， 克隆这个代码仓库
2， 在新的分支中做改动
3， 将代码提交到仓库后会自动触发一个制作容器的流程并将这个容器推送到容器仓库中
4， 将这个容器仓库下载到本地，运行并且测试

## 创建并使用一个你自己的制作容器的流程（准备长期开发的目的下推荐使用这种方式）
1， 克隆这个代码仓库
2， 对其中的代码做改动并且将改动提交到你自己的开发仓库中
3， 创建你自己的容器这个一点我们可以参考ECM小组的做法
4， 下载这个新的容器并且做测试，参阅一下的测试部分

## 如何为一个新的应用程序扩展IBACC框架
```
* 创建一个helm chart并且指出需要哪些用户输入
* 创建并且测试job容器，其中这个job容器就是用来执行部署helm release需要的步骤
```
我们会以一个例子来说明如何将一个新的应用加入到IBACC的详细上述步骤，这个例子将以ibm-nodejs-sample为例。

### 创建一个helm chart并且指出它需要输入的参数
这都是一个综合的步骤。 当你创建一个helm chart的时候你就会指出我们需要哪些参数，所以理想情况下当你的helm chart可用时，这个应用所需要的输入参数也就
定下来了。

### Helm chart的部署描述
helm chart有它自己特定的结构和架构。创建helm chart时可以参阅ICP小组定义的标准，这个例子我们是用了ibm-nodejs-sample helm chart。下表列出了所需要
的参数（大部分是ICP部署时需要的参数）：

### 使用linter 来测试helm chart
ICP小组开发了一套测试helm chart内容的工具，它可以用来保证所开发的helm chart符合标准并且没有技术问题。 总的来说我们要保证这个chart是遵守lint的标准
没有任何的错误抛出来。

### 在ICP中测试helm chart
我们应当把这个创建的helm chart导入到ICP中并做一个测试，这个是为了保证helm chart没有问题而且在把它搬进job container之前是可以部署它的。 将chart打包成包。
使用命令bx pr将chart 部署到ICP环境确保部署成功，而且能够成功创建出来实例。

## 创建job 容器
### 创建job 容器的步骤总览
大概如下，这个总览步骤可以根据具体的安装内容作必要的改动：
```
* job 容器是被ICP环境实例化，并且具有定义的入口
* 这个启动脚本将会去备份已有的日志文件到新的目录
* 然后启动脚本回去加载deploy-icp playbook
* 这个deploy-icp playbook会去初始化k8s/helm环境，然后调用deploy-product playbook并且传入所需要的参数
* deploy-product会利用已有的helm chart部署一个helm release
* deploy-icp playbook再去抽取目标服务的信息为部署做准备，并且将抽取结果序列化到jobs.xml文件去初始化和配置job容器。
```
### 为job 容器创建一系列文件
从这个仓库克隆job 容器模板，这是一个能够工作的例子主要是用来部署ibm-nodejs-sample helm chart

### 定制我们自己的启动脚本
job容器被定义为以指定启动脚本运行的容器--install.sh

### 把我们的helm chart放到指定的continer中
将我们做好的helm chart放到helm chart目录中比如在job container 模板中他被放到了这里

### 从jobs.yaml读取参数
默认情况下job 容器将会把job xml文件挂载到容器中，这样做的好处就是所有个job 容器都可以从统一的地方读取配置信息借以执行他们的目标任务。每一个job 容器
将会在启动时去挂在job xml文件到/opt/ibm/ibacc/cfg/jobs.xml。

在Ansible的playbooks中我们这样使用job配置文件中的值：vars_files: - /opt/ibm/ibacc/cfg/jobs.yaml。 在Ansible playbook中我们使用这样的标记来获得相应的值 {{sectionname.variablename}}

比如在job.yaml文件中，我们想要引用的值是在icpConfiguration小结下的Product子小结的变量名为RELEASE_NAME，相应的引用名称为:
vars:
  ibacc_icp_ecm_release:
    "{{icpConfiguration.Product.RELEASE_NAME}}"

我们就是使用了jobs.yaml 来参数化helm的release部署。

### ICp中部署文件Playbooks
ICp中一共使用了两个playbooks来安装产品。deploy-icp.yaml playbook是用来协调所有的安装过程，而deploy-product.yml playbook则被用来定义参数化的流程来部署helm release到ICp环境。

简单理解这个过程就是deploy-icp.yaml playbook 告诉我们要安装什么，deploy-product.yml则是用来告诉我们怎么去安装。

这个概念可以很容易的扩展为去安装多个产品（ECM就是一个例子，我们在ECM中一共同时部署了4个不同的helm charts）。

为你的安装icp Ansible脚本作必要的改动。 默认这个playbook 会有一系列的安装产品的步骤。 它会为你的ICp/k8s环境配置helm和kubectl，在helm release小结中通过include：tasks/icp/deploy-product.yml语法去调用deploy-product文件完成对helm release的部署。

基于jobs.yml文件为deploy-product指定值，在这个例子中你可以看到helm chart需要哪些值我们就为这些属性从jobs yaml文件中取出相应的值。
```
- name: Deployment Helm release into ICP
        include: tasks/icp/deploy-product.xlm
        vars:
          ibacc_icp_prod_name: cpe
          ibacc_icp_prod_image_name: "{{ productConfiguration.Product.IMGAGE_NAME.split(':')[0]}}{{':'}}{{productConfiguration.CPE.IMAGE_NAME.split(':')[1]}}"
          ibacc_icp_prod_image_tag: "{{ productConfiguration.Product.IMAGE_NAME.split(':')[2] }}"
          ibacc_icp_prod_release_name: "{{ icpConfiguration.Product.RELEASE_NAME }}"
          ibacc_icp_prod_chart_name: "{{ icpConfiguration.Product.CHART_NAME }}"
          ibacc_icp_prod_name_space: "{{ sharedConfiguration.SC_ECC_TARGET_NAMESPACE }}"
          ibacc_icp_prod_chart_file: "ibm-nodejs-sample-1.2.0.tgz"
```

### 根据需要修改你的deploy product Ansible playbook
注意：这个playbook是一个参数化的文件它允许你传入参数，比如helm chart 名称 和release名称，这样就能在部署的时候检查是否已经有相同的helm chart relase被部署了， 我们应当尽可能多的传入参数到这个playbook中，这样我们就能在不同的地方使用这个playbook。

## Job 容器的日志
这里有三个地方需要为job 容器中的任务来写入日志信息， 这些log都是在重要的任务开始执行时打印的，比如初始化helm或者部署helm release，这些应该根据不同的任务被写入到不同的文件中。
```
- name: write output of cmd
  lineinfile: create=yes dest={{ icpConfiguration.ICP_LOG_PATH}}/ibacc_icp_deploy_{{ ibacc_icp_prod_name}}_helm_install.log line="{{shell_stdout.stdout}}" state=present
```

我们还应该保证写入OK或者FAIL，这样ibacc就能探测到job是否被成功的执行还是中途被错误终止。

一般来说遇到我们不期望继续执行的情况（比如：不再初始化或者验证）我们就需要写出一个fail文件，比如：‘ibacc-nodejs.all.FAILED’，一个通用的方法比如说我们可以查找日志文件中的关键字如果遇到Ansible任务被错误终止的情况
```
fail=$(cat /opt/ibm/ibacc/cfg/logs/ibacc-ICp/ibacc.log | grep -i failed=0)
echo $fail
if [ -z "$faile" ]; then
  touch /opt/ibm/ibacc/cfg/logs/ibacc-ICp/ibacc-prod-provision.FAILED
else
  touch /opt/ibm/ibacc/cfg/logs/ibacc-ICp/ibacc-prod-provision.all.OK
fi
```

ibacc.summary日志文件被UI用来展示安装过程的实时状态。 这个log文件应该被谨慎使用，并且应当只用来在展示主要的job容器
```
- name: write to summary file
    lineinfile: create=yes dest="{{icpConfigureation.ICP_LOG_PATH}}/ibacc.summary" line="{{lookup('pipe','date +%Y-%m-%d-%H:%m:%S')}} Finished Initiallize Helm Chart Configuration" state=present
```
注意日志文件的输出路径必须遵循以下的规则：{{sharedConfiguration.SC_ECC_BASE_DIR}}/logs/<job id, such ibacc-ICp>, for example, /opt/ibm/ibacc/cfg/logs/ibacc-ICp

对于每一个job容器，我们必须达成一致保证使用一个唯一的job id。 ibacc-ODM, ibacc-BAI, 等等。
所以一个长的目录名称中就会有这样的目录：ibacc-ODM, ibacc-BAI, ibacc-ECM
对于这个领域的最佳实践我们也很期望集思广益。

## 测试容器
```
create PVC/PV 来为job容器存放配置文件
```
PV的定义，把以下的定义放到ibacc-pv.yml文件
```
{
 "Kind": "PersistentVolume",
 "apiVersion": "v1",
 "metadata": {
   "name": "ibacc-cfg-pv",
   "labels":{}
 },
 "spec": "
   storeageClassName: 'ecm-ibacc-pv',
   "capacity: {
      "storage": "1Gi"
    },
    accessModes":[
      "ReadWriteMany"
    ],
  "persistentVolumeReclaimPolicy": "Retain",
  "nfs": {
     "path": "/ibacc",
     "server": 172.16.248.55
```
把这两个文件放到同一个目录下面然后运行kubectl命令去创建pv/pvc
```
kubectl apply -f . -n <namespace>
```

从这里现在一个job.xml的副本，然后根据实际情况更新他的值。
```
sharedConfiguration:
  SC_K8S_ENVIRONMENT: https://9.30.57:8001
  SC_K8S_CLUSTER_NAME: mycluster.icp
  SC_K8S_TOKEN:eyJhbGci... ...
  SC_K8S_CLUSTER_CONTEXT: mycluster.icp-context
```
你也可以在jobs文件中定义其他的参数。 以nodejs为例，可以加入以下的productConfiguration小结到jobs文件中。

为kubectl创建一个部署你的job容器的yaml文件，参照这里

注意在步骤1中创建的PVC/PV，必须要挂载到job容器中去，挂载点如下。

拷贝PVC/PV到这个挂载点的主机目录下面。

下载job容器镜像文件（可以参考ecm是如何做的），然后把它推送到ICp本地仓库中去，我们也的需要把目标产品镜像文件推送到ICp本地仓库中去，在这个例子中是这样的：
```
docker pull ecm-containerization-docker-local.artifactory.swg-devops.com/ibacc-icpjobtemplate:latest
docker tag ecm-containerization-docker-local.artifactory.swg-devops.com/ibacc-icpjobtemplate:latest mycluster.icp:8500/ecm-icp-space/ibacc-icpjobtemplate:latest
docker push mycluster.icp:8500/ecm-icp-space/ibacc-icpjobtemplate:latest

docker pull ibmcom/icp-nodejs-sample:8
docker tag ibmcom/icp-nodejs-sample:8 mycluster.icp:8500/ecm-icp-space/icp-nodejs-sample:latest
docker push mycluster.icp:8500/ecm-icp-space/icp-nodejs-sample:latest
```

使用kubectl执行我们的job 容器在任何一个安装了kubectl和部署定义文件背靠背的机器上。

## 生成以及持续交付
在我们目前的编排中，我们自定了docker-gradle-plugin这个可以用来制作和发布docker images，而ecm-container-gradle-wrapper是一个自定义的gradle分布式文件它包含了基本的初始化步骤，比如定义image标签，同样那里还有Jenkins的运行时包里面包含一些基本的编排步骤，比如：下载基本镜像文件，制作docker镜像文件和推送他们到镜像仓库中去。

### 制作一个容器
为了制作一个指定的job容器，我们可以忽略编排逻辑而专注于特定的任务配置。参照这个例子，他只有几行代码。























