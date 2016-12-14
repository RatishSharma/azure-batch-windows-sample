**本文章将介绍如何基于Cloud Service配置Windows Batch Python计算环境，并通过Python进行批处理作业**

### **背景描述**
我们在研究《Azure Batch Python 客户端入门》，参考了以下资料：
https://docs.microsoft.com/zh-cn/azure/batch/batch-python-tutorial
https://www.azure.cn/documentation/articles/batch-python-tutorial/

建立Batch Pool计算节点，主要有以下两种方式来：

	1. SKU方式不支持中国区(中国无法访问Azure虚拟机应用商店映像列表)，因此无法基于这种方式创建Linux Batch Pool

``` python
 	new_pool = batch.models.PoolAddParameter(
        	id=pool_id,
        virtual_machine_configuration=batchmodels.VirtualMachineConfiguration(
            image_reference=image_ref_to_use,
            node_agent_sku_id=sku_to_use),
        vm_size=_POOL_VM_SIZE,
        target_dedicated=_POOL_NODE_COUNT,
        start_task=batch.models.StartTask(
            command_line=common.helpers.wrap_commands_in_shell('linux',
                                                               task_commands),
            run_elevated=True,
            wait_for_success=True,
            resource_files=resource_files),
    )
```

	2. 基于CloudServiceConfiguration从云服务创建Windows Batch Pool是唯一可行的方案

``` python
new_pool = batch.models.PoolAddParameter(
        id=pool_id,
        cloud_service_configuration=batchmodels.CloudServiceConfiguration(
            os_family="4",
            target_os_version="*"),
        vm_size=_POOL_VM_SIZE,
        target_dedicated=_POOL_NODE_COUNT,
        start_task=batch.models.StartTask(
            command_line=common.helpers.wrap_commands_in_shell('windows', task_commands),
            run_elevated=True,
            wait_for_success=True,
            resource_files=resource_files),
    )
```

### **前置条件**

以下依赖环境是运行示例必须配置的环境，否则程序无法运行通过：

	1. python 2.7 或 3.3+ 
	2. pip 9.0.1  
	3. cryptography
	4. azure-batch
	5. azure-storage  

我们在尝试安装以上依赖环境时遇到了诸多问题，首要的问题就是如何在Batch节点中安装Python环境，因为Windows Batch Pool也是基于云服务建立的计算节点，因此我们参考了云服务中通过powershell安装Python的资料，参考：[云服务配置Python环境](https://docs.microsoft.com/zh-cn/azure/cloud-services/cloud-services-python-ptvs )

在测试过程中，我们发现China Azure网络环境访问[Python官网](https://www.python.org)较慢，因此将文件源上传到China Storage，会明显提升下载及安装速度，同时我们在安装pip、cryptography等依赖的时候，如果使用默认的命令（从Global镜像下载）会遇到超时的情况，因此我们调整了安装的镜像。完整的安装脚本如下：

``` powershell
$nl = [Environment]::NewLine
Write-Output "Download python to install...$nl"

$url = "https://devstorage.blob.core.chinacloudapi.cn/files/python-3.5.2-amd64.exe"
$outFile = "${env:TEMP}\python-3.5.2-amd64.exe"
Write-Output "Downloading $url to $outFile$nl"
Invoke-WebRequest $url -OutFile $outFile

Write-Output "Installing$nl"
Start-Process "$outFile" -ArgumentList "/quiet", "InstallAllUsers=1" -Wait

Write-Output "Update pip and add dependency"
py -m pip install -U pip -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com 
py -m pip install cryptography -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
py -m pip install azure-batch -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
py -m pip install azure-storage -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com
	
Write-Output "Done$nl" 

```


### **代码修正**

我们纠正了较多的代码，并编写了完整的依赖环境安装脚本，具体下载地址：

	1. python_tutorial_client.py中的create_pool定义修改如下：
``` python
def create_pool(batch_service_client, pool_id, resource_files):   
    print('Creating pool [{}]...'.format(pool_id))
    task_commands = [
        'copy %AZ_BATCH_TASK_WORKING_DIR%\* %AZ_BATCH_NODE_SHARED_DIR%',
        'powershell.exe -ExecutionPolicy Unrestricted -File %AZ_BATCH_NODE_SHARED_DIR%\PrepPython35.ps1']

    new_pool = batch.models.PoolAddParameter(
        id=pool_id,
        cloud_service_configuration=batchmodels.CloudServiceConfiguration(
            os_family="4",
            target_os_version="*"),
        vm_size=_POOL_VM_SIZE,
        target_dedicated=_POOL_NODE_COUNT,
        start_task=batch.models.StartTask(
            command_line=common.helpers.wrap_commands_in_shell('windows', task_commands),
            run_elevated=True,
            wait_for_success=True,
            resource_files=resource_files),
    )

    try:
        batch_service_client.pool.add(new_pool)
    except batchmodels.batch_error.BatchErrorException as err:
        print_batch_exception(err)
        raise
```

	2. python_tutorial_client.py中的add_tasks定义修改如下：
``` python
def add_tasks(batch_service_client, job_id, input_files,
              output_container_name, output_container_sas_token):
    print('Adding {} tasks to job [{}]...'.format(len(input_files), job_id))
    tasks = list()
    for idx, input_file in enumerate(input_files):

        command = ['"%ProgramFiles%\Python35\Python.exe" %AZ_BATCH_NODE_SHARED_DIR%\python_tutorial_task.py '
                   '--filepath {} --numwords {} --storageaccount {} '
                   '--storagecontainer {} --sastoken "{}"'.format(
                       input_file.file_path,
                       '3',
                       _STORAGE_ACCOUNT_NAME,
                       output_container_name,
                       output_container_sas_token)]

        tasks.append(batch.models.TaskAddParameter(
                'topNtask{}'.format(idx),
                common.helpers.wrap_commands_in_shell('windows', command),
                resource_files=[input_file]
                )
        )

    batch_service_client.task.add_collection(job_id, tasks)
```

	3. python_tutorial_client.py中的__main__主要修改Endopint，文件路径等：
``` python
	...
    blob_client = azureblob.BlockBlobService(
        account_name=_STORAGE_ACCOUNT_NAME,
        account_key=_STORAGE_ACCOUNT_KEY,
        endpoint_suffix='core.chinacloudapi.cn')

  	...
    application_file_paths = [os.path.realpath('./article_samples/python_tutorial_task.py'),
                              os.path.realpath('./article_samples/PrepPython35.ps1')]

    input_file_paths = [os.path.realpath('./article_samples/data/taskdata1.txt'),
                        os.path.realpath('./article_samples/data/taskdata2.txt'),
                        os.path.realpath('./article_samples/data/taskdata3.txt')]
	...

```



### **运行测试**





	```


