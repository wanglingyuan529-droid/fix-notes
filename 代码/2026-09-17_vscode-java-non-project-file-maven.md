# VSCode Java 报 non-project file：配置写错了层级，pom.xml 才是解法

> 日期：2026-09-17 · 环境：Windows 11 (x86_64) · JDK 25 (Microsoft OpenJDK 25.0.4) · redhat.java 1.56.0

## 现象

从 GitHub 克隆下来的 Java 仓库在 VSCode 里每个文件顶部提示：

```
Test.java is a non-project file, only syntax errors are reported
```

同时没有类型检查、跳转定义、补全、CodeLens 运行按钮——只有最基本的语法高亮。

## 根因

三层，缺一层都不会出现这个现象：

1. **红帽 Java 插件不是语法高亮器，是完整 IDE 内核**（Eclipse JDT）。它只对「项目模型」内的文件做完整分析；
   项目模型的来源只有三种：`pom.xml`(Maven)、`build.gradle`(Gradle)、Eclipse 元数据(`.project`/`.classpath`)。
   仓库一个都没有 → 没有模型 → 文件是「非项目文件」，只给语法检查。

2. 插件对无构建文件的文件夹有**隐形项目（unmanaged folder）**兜底，但其配置
   （`java.project.sourcePaths`）属于 **VSCode 工作区设置**。

3. **工作区设置只从「打开的那个文件夹」的 `.vscode/settings.json` 读**。
   而实际打开的是父目录 `C:\study\java`（同级还有别的练习目录），
   配置却写在了子目录 `coding-diary\.vscode\settings.json` —— **从来没被读取过**，重载多少次都无效。

## 排查

VSCode 最近打开的工作区可以直接从磁盘读出来，不用猜：

```powershell
Get-ChildItem "$env:APPDATA\Code\User\workspaceStorage" -Directory | ForEach-Object {
  $wj = Join-Path $_.FullName 'workspace.json'
  if (Test-Path $wj) { (Get-Content $wj -Raw | ConvertFrom-Json).folder }
}
# → file:///c%3A/study/java      ← 打开的是这个，不是 coding-diary
```

## 步骤

### 1. 迁移到标准 Maven 布局（用 `git mv` 保留历史）

```powershell
mkdir src\main\java
git mv src\ArrayList        src/main/java/ArrayList
git mv src\oop              src/main/java/oop
git mv src\stringdemo       src/main/java/stringdemo
git mv src\studentmanagement src/main/java/studentmanagement
```

`git status` 里显示为 `R`（rename），历史不断。

### 2. 加最小 `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <groupId>io.github.wanglingyuan529</groupId>
    <artifactId>coding-diary</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.release>25</maven.compiler.release>
    </properties>
</project>
```

零依赖的 pom 不需要本机装 Maven，插件自己就能解析。

### 3. 清一次语言服务器缓存

`Ctrl+Shift+P` → **Java: Clean Java Language Server Workspace** → **Restart and delete**
（旧隐形项目模型有缓存，不清会继续按旧模型报错）

## 坑

1. **配置写在哪一层决定它有没有被读**：`.vscode/settings.json` 只在**工作区根**生效，
   写在子目录里等于没写。这个坑很隐蔽——文件在那儿躺着，看起来"配过了"。
2. **隐形项目的 sourcePaths 默认是工作区根**，不是 `src`；工作区根若是父目录，子仓库的文件就落在根之外。
3. **插件缓存要手动清**：改完配置只 Reload Window 不够，必须 Clean Workspace 重建模型。
4. **顺带纠正一个误判**：`static void main()`（无 `public`、无 `String[] args`）在 **JDK 25 是合法入口**，
   是 JEP 512 正式定稿的特性（Java 21 起预览），不要当 bug 去"修"。实测编译运行均通过。
   注意老环境（JDK ≤ 24、刷题平台多为 8/11/17）仍需写全 `public static void main(String[] args)`。

## 验证

```powershell
# 按新布局全量编译（模拟语言服务器的分析范围）
javac -encoding UTF-8 -d <临时目录> @<69个源文件列表>
# → exit 0，69 个源文件 → 69 个 class
```

VSCode 侧：警告消失、出现运行按钮、跳转/补全恢复。

## 教训

1. **让工具"自动识别"的配置，要放在工具真正扫描的层级**。构建文件（pom.xml）会被递归扫描，
   工作区设置只认根目录——前者天然更稳。
2. **优先选「跟仓库走」的方案**：`pom.xml` 提交进仓库后，任何机器 clone 下来都能识别，
   不依赖某个人的 VSCode 设置。
3. **报错信息要读到最后一层**：警告文本说的是"非项目文件"，但真正的答案是"你打开的文件夹不是我配置的那个"。
