# AI Search和GPT Demo部署指南
这个demo是基于AI Search和GPT的实现的RAG问答系统，可以根据用户的输入，返回相应的答案。
<br/>
https://github.com/Azure-Samples/azure-search-openai-demo/
<br/>

![appcomponents](./img/appcomponents.png)

<br/>
这个demo会部署如下资源: <br/>

 - 一个AI Search实例 (可以提前建好)
 - 一个GPT实例 （可以提前建好），需要部署gpt-3.5(0613)和embedding模型
 - 一个Web App实例
 - Log Analytics工作区
 - Application Insights实例

## 环境准备

安装如下工具:

* [Azure Developer CLI](https://aka.ms/azure-dev/install)
* [Python 3.9, 3.10, or 3.11](https://www.python.org/downloads/)
  * **Important**: Python and the pip package manager must be in the path in Windows for the setup scripts to work.
  * **Important**: 确认python版本 `python --version` 
* [Node.js 14+](https://learn.microsoft.com/zh-cn/windows/dev-environment/javascript/nodejs-on-windows)
* [Git](https://git-scm.com/downloads)
* [Powershell 7+ (pwsh)](https://github.com/powershell/powershell) - For Windows users only.
  * **Important**: 可认可以运行命令 `pwsh.exe` from a PowerShell terminal. If this fails, you likely need to upgrade PowerShell.


## 部署步骤
1. 下载代码
``` bash
git clone https://github.com/Azure-Samples/azure-search-openai-demo/
cd azure-search-openai-demo
#先清空data目录， 这里是样例的pdf文件。部署完Azure资源后，可以把自己的pdf文件放到这个目录下

mv data data.bak
mkdir data
```

2. 配置参数
``` bash
# 配置开发环境名称
azd env list

azd auth login
azd env new
#输入环境名称

#如果换了环境
azd env refresh -e <env-name>

```
环境信息会保存在项目的.azure/\<env-name\>/.env中 <br/>
如果需要修改环境信息，可以直接修改.env文件，也可以使用azd env set命令


设置搜索对中文支持建议如下配置
``` bash
azd env set AZURE_SEARCH_QUERY_LANGUAGE zh-CN
azd env set AZURE_SEARCH_QUERY_SPELLER none
azd env set AZURE_SEARCH_ANALYZER_NAME zh-Hans.lucene
#如果是开发/Demo环境，可以设置为Basic SKU
azd env set AZURE_SEARCH_SERVICE_SKU basic
```
如果是全部资源重新创建，就可以开始部署，要注意openai所在区域可用的tpm要大于30. 如果想用现有的资源，如以设置完后面的参数再部署<br/>
部署，按提示选择订阅和区域。如果出错，可以重复部署，会自动跳过已经部署的资源.  
``` bash
azd up
```

部署过程 可以按提示打开URL，查看部署进度

 - 如果是现有的资源组

``` bash
#Existing resource group
azd env set AZURE_RESOURCE_GROUP rg-aidemo
azd env set AZURE_LOCATION japaneast
```


 - 如果是现有的Azure AI Search资源
``` bash
#Existing Azure AI Search resource

azd env set AZURE_SEARCH_SERVICE azsearch-jpe
azd env set AZURE_SEARCH_SERVICE_RESOURCE_GROUP cogsvc
azd env set AZURE_SEARCH_SERVICE_LOCATION japaneast
###如果是现有的Azure AI Search资源为basic，需要设置SKU为Basic. 默认是standard
azd env set AZURE_SEARCH_SERVICE_SKU basic
```

 - [不建议] 如果是现有的OpenAI资源。(这里比较容易出错，建设预留好gpt3.5和embedding的tpm, 重新创建完再改配置。否则需要改./infra/main.bicep里的参数与现有资源对应)
``` bash

#Existing OpenAI resource
#Azure OpenAI:
azd env set AZURE_OPENAI_SERVICE aoaixxx-swc
azd env set AZURE_OPENAI_RESOURCE_GROUP cogsvc
azd env set AZURE_OPENAI_CHATGPT_DEPLOYMENT gpt-3.5-0613
azd env set AZURE_OPENAI_EMB_DEPLOYMENT emb002

```

## 增加文件索引
1. 把pdf文件放到data目录下
2. 运行prepdocs.ps1 (Windows) 脚本，生成索引文件。大概过程如下:
    -  读取.env文件，获取环境信息
    -  列出data目录下的pdf文件
    - 循环读取pdf文件，使用Document Intelligence服务识别出内容。默认是prebuild-layout模型
    - 读取完的pdf文档会在目录下生成md5文件，下次运行时会跳过。
    - 将识别的内容分段(secionts), 做embedding.
    - 将分好段的内容添加到AI Search的索引文件(如果不存在，会自动创建索引文件)
    - 将pdf文件上传到Azure Blob Storage, 用于后续引用(citation)

``` bash

#Windows
cd <workspace_dir>/azure-search-openai-demo
./scripts/prepdocs.ps1
``` 