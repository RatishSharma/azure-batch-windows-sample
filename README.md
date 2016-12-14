**Windows Batch Python 示例**

### **背景描述**
最近，在China Azure环境下运行Batch Python示例，遇到较多问题。为此，我们花费了较多的时间进行测试以解决这个问题，这里需要感谢微软的Allan、21V的George、Kevin共同的努力。

	1. https://docs.microsoft.com/zh-cn/azure/batch/batch-python-tutorial
	2. https://www.azure.cn/documentation/articles/batch-python-tutorial/


以上资料由微软官方提供，但是无法直接在China Azure环境下直接使用，主要问题如下

	1. SKU方式不支持中国区(中国无法访问Azure虚拟机应用商店映像列表，已经向产品组提交Bug)，因此无法基于这种方式创建Linux Batch Pool
	  

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

以下是本地运行实例、Batch节点依赖的运行时环境：

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
Sample start: 2016-12-14 05:34:52

Uploading file C:\Users\kevin\Desktop\george-python-test\azure-batch-samples-mas
ter\Python\Batch\article_samples\python_tutorial_task.py to container [applicati
on]...
Uploading file C:\Users\kevin\Desktop\george-python-test\azure-batch-samples-mas
ter\Python\Batch\article_samples\PrepPython35.ps1 to container [application]...
Uploading file C:\Users\kevin\Desktop\george-python-test\azure-batch-samples-mas
ter\Python\Batch\article_samples\data\taskdata1.txt to container [input]...
Uploading file C:\Users\kevin\Desktop\george-python-test\azure-batch-samples-mas
ter\Python\Batch\article_samples\data\taskdata2.txt to container [input]...
Uploading file C:\Users\kevin\Desktop\george-python-test\azure-batch-samples-mas
ter\Python\Batch\article_samples\data\taskdata3.txt to container [input]...
Creating pool [PythonTutorialPool01]...
Creating job [PythonTutorialJob01]...
Adding 3 tasks to job [PythonTutorialJob01]...
Monitoring all tasks for 'Completed' state, timeout in 0:30:00..................
................................................................................
................................................................................
................................................................................
................................................................................
....................................................................
  Success! All tasks reached the 'Completed' state within the specified timeout
period.
C:\Users\kevin
Downloading all files from container [output]...
  Downloaded blob [taskdata1_OUTPUT.txt] from container [output] to C:\Users\kev
in\taskdata1_OUTPUT.txt
  Downloaded blob [taskdata2_OUTPUT.txt] from container [output] to C:\Users\kev
in\taskdata2_OUTPUT.txt
  Downloaded blob [taskdata3_OUTPUT.txt] from container [output] to C:\Users\kev
in\taskdata3_OUTPUT.txt
  Download complete!
Deleting containers...

Sample end: 2016-12-14 05:42:20
Elapsed time: 0:07:28

Delete job? [Y/n]

*********************************************************************************************

Word	Count
------------------------------
to:	18
and:	17
you:	14
------------------------------
Node: tvm-884213370_1-20161214t053541z
Task: topNtask0
Job:  PythonTutorialJob01
Pool: PythonTutorialPool01
