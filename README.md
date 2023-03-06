# CI

## 持续集成部署之Coding篇
- 使用Coding持续集成可以实现软件自动化构建，流水线的形式方便开发人员快速，灵活地构建产品软件包
- 可以快速定制多版本应用并一键发布至多环境平台

1. 在coding上创建代码仓库
2. 创建构建计划。进入持续集成，coding提供了很多构建模板，并没有合适unity的 所以选择自定义构建 勾选使用静态配置Jenkinsfile
3. 在Jenkinsfile中进行流程配置，提供了图形和文本两种编辑器，默认已经给出代码仓库检出stage,以下举例自动打包unity说明

### 构建计划
补充自定义构建过程，%xxx%为流程环境变量 可以在coding上批量添加，比如APP打包的路径，名称，版本号等
```
pipeline {
  agent any
  stages {
    stage("检出") {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: GIT_BUILD_REF]],
          userRemoteConfigs: [[
            url: GIT_REPO_URL,
            credentialsId: CREDENTIALS_ID
        ]]])
      }
    }
    stage('自定义构建过程') {
      steps {
        echo "自定义构建过程开始 "
        
        // 请在此处补充您的构建过程
    bat '''

        echo "正在生成exe文件..."
        %unity%  -batchmode -quit -executeMethod BuildEditor.BuildPackage -logFile %logFile% -projectPath %projectPath% %outpath% %scenes% %AppName%



    '''
      }
    }
  }
}
```

- 还可以在coding页面上设置触发的规则 略

### 构建节点

- 使用一台windows电脑作为构建主机
1. 创建节点池。
2. 接入新节点。选择接入平台为Windows,接入方式为Powershell，这样我们就可以使用coding下方的命令直接完成节点所有的安装和接入过程。如果出现报错可查看文档，看具体是缺少哪些依赖
    > 我自己在装的时候出现报错，后来查明是安装的Python版本过高的原因，Python版本必须小于3.6 3.7 3.8 3.9;
3. 接入完后 可以使用命令qci_worker up -d 后台启动，或者qci_worker stop 进行关闭 ###

### 立即构建

- 回到构建计划，在stage中调用bat 批处理 调用Unity编辑器接口进行打包,自定义打包过程参数解释
> **%unity%** 为要使用的unity软件版本Editor下Unity.exe文件路径
> **BuildEditor.BuildPackage** 前者是类名，后者是调用的打包方法名
> **%logFile%** 日志文件输出地址
> **%projectPath%** 需要打包的unity工程地址
> **%outpath% %scenes% %AppName%** 分别是打包后发布地址，打包的场景名称，包名

- unity编辑器代码举例：
```c#
using UnityEditor;
using UnityEngine;

public class BuildEditor
{
    private static string[] paramss;    //传入的参数

    public static void BuildPackage()
    {

        paramss = System.Environment.GetCommandLineArgs();    
        BuildForStandaloneWindows();
    }

    //windows打包
  
    static void BuildForStandaloneWindows()
    {
        string[] levels = GetSceneNames();    
        for (int i = 0; i < levels.Length; i++)    
        {
            levels[i] = "Assets\\Scenes\\" + levels[i] + ".unity";    //
        }
        string path = GetExportPath();   
        BuildPipeline.BuildPlayer(levels, @path, BuildTarget.StandaloneWindows, BuildOptions.None);   


    }


    //获取发布路径
    private static string GetExportPath()
    {
        string path = paramss[paramss.Length - 3];
        string exepath = @path + "\\" + GetAppName()+".exe";
        return exepath;
    }

    //获取场景名
    private static string[] GetSceneNames()
    {
        string names = paramss[paramss.Length - 2];    
        string[] sceneNames = names.Split(new char[] { ',' });    
        for (int i = 0; i < sceneNames.Length; i++)
        {
            return sceneNames;
        }
        return null;
    }

    //获取包名
    private static string GetAppName()
    {
        string name = paramss[paramss.Length - 1];
        
        return name;
    }
}

```

- 点击立即构建后 coding就按照我们自己定义的流程开始执行，如果有报错也会给出log方便我们定位问题


