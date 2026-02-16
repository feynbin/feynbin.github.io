---
title: maven基础
top_img: false
abbrlink: 195cf5d1
date: 2024-12-23 20:05:52
updated:
tags: "maven"
categories:
keywords:
description:
cover:
highlight_shrink:
---

> 内容来自 GeekHour 的一小时 Maven 教程和官方文档

## **Maven 简介**

`Maven` 是由 `Apache` 软件基金会开源的一个自动化构建工具，主要用来解决 `Java` 项目中最常见的两个问题：**依赖管理**和**项目构建**。

`Maven` 解决的第一个问题是**依赖管理**。我们只需要在一个叫做 `POM` 的 `XML` 文件中告诉 Maven 需要哪些依赖，Maven 就会将 `Jar` 包以及它所依赖的所有其他 `Jar` 包全部下载并导入到项目中。

解决的另一个问题是**构建管理**。在 Java 项目中，需要把 Java 源文件编译成字节码文件，然后再打包成一个可执行的 `Jar` 包或 `War` 包。如果没有自动化构建工具，这个过程会非常繁琐。Maven 提供了一个标准的项目结构和构建流程，只需要按照这个标准来组织项目，就可以轻松地执行构建、打包和部署等工作。

`Maven` 的核心概念是**项目对象模型**（POM），它是一个 XML 文件，也是 Maven 项目的核心文件，定义了项目的配置、依赖、插件以及构建过程。Maven 读取 `pom.xml` 文件后，会根据其中定义的规则下载依赖包，编译源代码，最后将工程打包成可执行的 `Jar` 包或 `War` 包。这个过程中有很多插件来协助完成工作，比如编译插件、打包插件、测试插件等，这些都是 Maven 提供的，只需要在 `pom.xml` 中配置即可。

### **仓库**

在 Maven 中有一个仓库的概念，简单来说就是存放 Jar 包的地方。按照作用范围的不同，可以分为三种：

- **本地仓库**：自己电脑上的一个目录，默认位于家目录下的 `.m2` 目录中，可以在 Maven 的配置文件中修改。
- **远程仓库（私服）**：公司或组织内部搭建的仓库，用来给内部项目提供统一的依赖管理，也可以存放私有的 Jar 包。常见的有 `Nexus Repository` 和 `Reposilite`。
- **中央仓库**：Maven 官方维护的仓库，所有开源的 Jar 包都可以在中央仓库中找到。

Maven 下载依赖时的查找顺序：**本地仓库** → **远程仓库** → **中央仓库**。下载完成后会缓存到本地仓库，下次直接从本地获取，避免重复下载。

---

## **安装与配置**

### **安装**

#### **Windows**

```powershell
# 使用 scoop
scoop install maven

# 使用 winget
winget install Apache.Maven
```

#### **macOS**

```bash
brew install maven
```

#### **Linux**

```bash
# Debian/Ubuntu
sudo apt install maven

# Arch Linux
sudo pacman -S maven
```

验证安装：

```shell
$ mvn --version
Apache Maven 3.9.9
```

### **核心配置文件**

Maven 的全局配置文件是 `settings.xml`，位于 Maven 安装目录的 `conf/` 下，也可以放在用户目录 `~/.m2/settings.xml` 中（用户级别优先）。

常见的配置项：

#### **修改本地仓库路径**

```xml
<settings>
    <localRepository>D:/maven-repo</localRepository>
</settings>
```

#### **配置镜像源**

国内使用 Maven 中央仓库速度较慢，可以配置阿里云镜像加速：

```xml
<settings>
    <mirrors>
        <mirror>
            <id>aliyun</id>
            <name>Aliyun Maven Mirror</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
    </mirrors>
</settings>
```

#### **配置默认 JDK 版本**

```xml
<settings>
    <profiles>
        <profile>
            <id>jdk-17</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <maven.compiler.source>17</maven.compiler.source>
                <maven.compiler.target>17</maven.compiler.target>
            </properties>
        </profile>
    </profiles>
</settings>
```

---

## **项目结构**

Maven 定义了一套标准的项目目录结构：

```
my-project/
├── pom.xml                  # 项目配置文件
├── src/
│   ├── main/
│   │   ├── java/            # Java 源代码
│   │   └── resources/       # 资源文件（配置文件等）
│   └── test/
│       ├── java/            # 测试代码
│       └── resources/       # 测试资源文件
└── target/                  # 编译输出目录（自动生成）
```

---

## **POM 文件**

`pom.xml` 是 Maven 项目的核心文件。一个基本的 POM 文件结构如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- 项目坐标 -->
    <groupId>com.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <!-- 项目信息 -->
    <name>My Project</name>
    <description>A sample Maven project</description>

    <!-- 属性定义 -->
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <!-- 依赖管理 -->
    <dependencies>
        <!-- 具体依赖 -->
    </dependencies>

    <!-- 构建配置 -->
    <build>
        <plugins>
            <!-- 构建插件 -->
        </plugins>
    </build>
</project>
```

### **项目坐标**

Maven 通过三个元素唯一标识一个项目或依赖，称为**坐标**（GAV）：

| 元素 | 说明 | 示例 |
|---|---|---|
| `groupId` | 组织或公司的标识，通常是域名反写 | `com.example` |
| `artifactId` | 项目或模块的名称 | `my-project` |
| `version` | 版本号 | `1.0.0` |

### **打包类型**

`<packaging>` 指定项目的打包方式：

| 类型 | 说明 |
|---|---|
| `jar` | 默认值，打包为 Jar 文件 |
| `war` | 打包为 Web 应用 |
| `pom` | 不打包，作为父工程或聚合工程使用 |

---

## **Maven 命令**

Maven 通过命令行执行各种构建操作。常用命令：

```shell
# 编译项目（编译 src/main/java 中的代码）
mvn compile

# 编译测试代码
mvn test-compile

# 运行测试
mvn test

# 打包项目（生成 jar/war 文件到 target/ 目录）
mvn package

# 安装到本地仓库（供其他本地项目引用）
mvn install

# 部署到远程仓库
mvn deploy

# 清理 target 目录
mvn clean

# 常用组合：清理后打包
mvn clean package

# 清理后安装，跳过测试
mvn clean install -DskipTests
```

常用参数：

| 参数 | 说明 |
|---|---|
| `-DskipTests` | 跳过测试执行（但仍编译测试代码） |
| `-Dmaven.test.skip=true` | 跳过测试编译和执行 |
| `-P <profile>` | 激活指定的 profile |
| `-pl <module>` | 只构建指定的模块 |
| `-am` | 同时构建所依赖的模块 |
| `-o` | 离线模式，不从远程仓库下载 |
| `-U` | 强制更新快照版本依赖 |

---

## **生命周期与插件**

### **三套生命周期**

Maven 有三套独立的生命周期，每个生命周期包含若干有序的阶段（phase）：

#### **Clean 生命周期**

负责清理工作：

1. `pre-clean`
2. `clean` — 删除 `target` 目录
3. `post-clean`

#### **Default 生命周期**

核心生命周期，负责项目的构建和部署（主要阶段）：

1. `validate` — 验证项目结构是否正确
2. `compile` — 编译源代码
3. `test` — 运行单元测试
4. `package` — 打包
5. `verify` — 运行集成测试验证
6. `install` — 安装到本地仓库
7. `deploy` — 部署到远程仓库

> 执行某个阶段时，它前面的所有阶段都会被依次执行。例如执行 `mvn package` 时，`validate`、`compile`、`test` 都会先执行。

#### **Site 生命周期**

负责生成项目文档站点：

1. `pre-site`
2. `site` — 生成项目文档
3. `post-site`
4. `site-deploy` — 发布文档

### **插件**

Maven 的每个生命周期阶段都绑定了一个或多个插件目标（plugin goal）。插件是实际执行构建任务的组件。

常用插件：

| 插件 | 说明 |
|---|---|
| `maven-compiler-plugin` | 编译 Java 源代码 |
| `maven-surefire-plugin` | 运行单元测试 |
| `maven-jar-plugin` | 打包为 Jar 文件 |
| `maven-war-plugin` | 打包为 War 文件 |
| `maven-shade-plugin` | 打包为包含所有依赖的 Fat Jar |
| `maven-spring-boot-plugin` | Spring Boot 项目打包 |

在 `pom.xml` 中配置插件示例：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.13.0</version>
            <configuration>
                <source>17</source>
                <target>17</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## **依赖管理**

### **添加依赖**

在 `pom.xml` 的 `<dependencies>` 中添加所需的依赖：

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>3.3.0</version>
    </dependency>

    <!-- JUnit 5 测试框架 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.11.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

> 可以在 `https://mvnrepository.com/` 搜索需要的依赖坐标。

### **依赖范围（Scope）**

`<scope>` 控制依赖在哪些阶段可用：

| Scope | 编译 | 测试 | 运行 | 打包 | 说明 |
|---|---|---|---|---|---|
| `compile` | ✓ | ✓ | ✓ | ✓ | 默认值，所有阶段都可用 |
| `provided` | ✓ | ✓ | ✗ | ✗ | 编译和测试时可用，运行时由容器提供（如 Servlet API） |
| `runtime` | ✗ | ✓ | ✓ | ✓ | 编译时不需要，运行时需要（如 JDBC 驱动） |
| `test` | ✗ | ✓ | ✗ | ✗ | 仅测试时可用（如 JUnit） |
| `system` | ✓ | ✓ | ✗ | ✗ | 类似 provided，但需要手动指定 Jar 路径，不推荐使用 |

### **依赖传递**

Maven 会自动处理**传递性依赖**。如果项目 A 依赖 B，B 依赖 C，那么 A 也会自动引入 C。

当传递性依赖导致版本冲突时，Maven 遵循以下原则：

- **最短路径优先**：依赖层级越浅的版本优先
- **先声明优先**：在同一层级中，`pom.xml` 中先声明的优先

### **排除依赖**

当传递性依赖引入了不需要的或冲突的包时，可以使用 `<exclusions>` 排除：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>3.3.0</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### **查看依赖树**

```shell
# 查看完整的依赖树
mvn dependency:tree

# 查看是否有依赖冲突
mvn dependency:analyze
```

---

## **父子工程**

在大型项目中，通常会拆分为多个模块。Maven 通过**父子工程**（多模块项目）来管理这种结构。

### **父工程**

父工程的 `pom.xml` 中 `<packaging>` 必须为 `pom`，通过 `<modules>` 声明子模块：

```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>parent-project</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <!-- 声明子模块 -->
    <modules>
        <module>module-common</module>
        <module>module-service</module>
        <module>module-web</module>
    </modules>

    <!-- 统一管理依赖版本 -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.3.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- 所有子模块都会继承的依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.34</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```

### **子工程**

子工程通过 `<parent>` 继承父工程，可以省略 `groupId` 和 `version`：

```xml
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>parent-project</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>module-service</artifactId>

    <dependencies>
        <!-- 引用父工程 dependencyManagement 中管理的依赖，无需指定版本 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 引用兄弟模块 -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>module-common</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</project>
```

### **dependencies 与 dependencyManagement 的区别**

| 特性 | `<dependencies>` | `<dependencyManagement>` |
|---|---|---|
| 作用 | 直接引入依赖 | 仅声明依赖版本，不实际引入 |
| 子模块是否自动继承 | 是 | 否，子模块需要手动声明依赖，但可省略版本号 |
| 使用场景 | 所有模块都需要的公共依赖 | 统一管理依赖版本，避免版本冲突 |

---

## **私服仓库**

私服仓库是部署在公司或组织内部的 Maven 仓库，用于：

- 缓存中央仓库的依赖，加速构建
- 存放组织内部的私有依赖包
- 统一管理依赖的来源和版本

常见的私服工具有 **Nexus Repository** 和 **Reposilite**。

### **配置私服**

在 `settings.xml` 中配置私服认证信息：

```xml
<settings>
    <servers>
        <server>
            <id>my-nexus</id>
            <username>admin</username>
            <password>password</password>
        </server>
    </servers>
</settings>
```

在 `pom.xml` 中配置发布地址：

```xml
<distributionManagement>
    <repository>
        <id>my-nexus</id>
        <url>http://nexus.example.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>my-nexus</id>
        <url>http://nexus.example.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

使用 `mvn deploy` 即可将项目发布到私服仓库。

> `<server>` 中的 `<id>` 必须与 `<repository>` 中的 `<id>` 一致，Maven 才能找到对应的认证信息。

---

## **Profile**

Profile 用于在不同环境下使用不同的配置（如开发、测试、生产环境）：

```xml
<profiles>
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <env>dev</env>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <env>prod</env>
        </properties>
    </profile>
</profiles>

<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

使用方式：

```shell
# 使用 dev 环境配置（默认）
mvn clean package

# 使用 prod 环境配置
mvn clean package -P prod
```

---

## **版本管理**

Maven 的版本号遵循约定：

- **SNAPSHOT 版本**：开发阶段的不稳定版本，如 `1.0.0-SNAPSHOT`。每次构建 Maven 都会检查远程仓库是否有更新。
- **RELEASE 版本**：正式发布的稳定版本，如 `1.0.0`。一旦发布就不应修改。

版本号推荐遵循**语义化版本**（Semantic Versioning）：

```
主版本号.次版本号.修订号
  1    .  0   .  0
```

- **主版本号**：不兼容的 API 变更
- **次版本号**：向下兼容的功能新增
- **修订号**：向下兼容的问题修复

---

## **常用技巧**

### **使用属性统一管理版本**

```xml
<properties>
    <spring-boot.version>3.3.0</spring-boot.version>
    <mybatis.version>3.5.16</mybatis.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>${mybatis.version}</version>
    </dependency>
</dependencies>
```

### **使用 Maven Wrapper**

Maven Wrapper 可以让项目自带 Maven，无需团队成员单独安装：

```shell
# 生成 Maven Wrapper
mvn wrapper:wrapper

# 使用 Wrapper 构建（Linux/macOS）
./mvnw clean package

# 使用 Wrapper 构建（Windows）
mvnw.cmd clean package
```

### **查看有效 POM**

当继承关系复杂时，可以查看最终生效的完整 POM：

```shell
mvn help:effective-pom
```

---

## **展望 Maven 4**

Maven 3 发布于 2010 年，至今已经服役超过 15 年。Maven 4 是这十多年来最大的一次架构升级，截至目前（2026 年 2 月）已经推进到 RC-5 阶段，正式版发布在即。以下是 Maven 4 中值得关注的重要变化。

### **Java 17 基线**

Maven 4 要求 Java 17 作为最低运行环境。这不影响项目本身的编译目标——你仍然可以通过 `maven-compiler-plugin` 或 Toolchains 编译到 Java 8 等旧版本，但 Maven 自身的运行需要 Java 17。这使得 Maven 内部可以使用现代 Java 特性进行重构和优化。

### **构建 POM 与消费者 POM 分离**

这是 Maven 4 最具实际意义的改进之一。在 Maven 3 中，发布到仓库的 POM 和用于构建的 POM 是同一个文件，其中包含大量消费者不需要的信息（构建插件配置、Profile 等）。许多项目不得不依赖 `flatten-maven-plugin` 来生成精简的 POM，配置繁琐且容易出错。

Maven 4 原生支持生成**消费者 POM**（Consumer POM），发布到远程仓库的是一个自动精简的版本：父 POM 引用被展开、BOM 导入被平铺、非传递性依赖被移除。这从根本上解决了长期困扰社区的 POM 膨胀问题。

### **多模块项目的全面改善**

Maven 4 对多模块（现在更名为"多子项目"）的支持有了质的飞跃：

- **自动版本推断**：子项目的 `<parent>` 中可以省略 `groupId`、`artifactId` 和 `version`，只需声明 `<relativePath>` 即可自动推断，甚至可以简写为 `<parent/>`
- **自动子项目发现**：当父 POM 的打包类型为 `pom` 时，不再需要手动列出 `<modules>`，Maven 会自动扫描包含 `pom.xml` 的子目录
- **CI 友好版本原生支持**：`${revision}` 等变量可以直接使用，不再需要 `flatten-maven-plugin`
- **项目本地仓库**：多模块构建时，已构建的模块产物缓存在根项目的 `target` 下，为增量构建铺平了道路

这些改进对于管理大型多模块项目有着直接的帮助，大幅减少了样板配置。

### **生命周期架构升级**

Maven 3 的生命周期是一个线性的有序列表，Maven 4 将其升级为**树形结构**，引入了更细粒度的阶段依赖关系。每个阶段现在都有对应的 `before:` 和 `after:` 变体，可以精确控制执行时机和顺序。配合 `mvn -b concurrent` 并发构建器，多模块项目的构建速度可以得到显著提升。

### **新的依赖解析器**

Maven 4 集成了 Resolver 2.0，包含 150 多项修复和改进，采用了基于 Java 17 的原生 HTTP 客户端替代旧实现，在大型依赖树的解析性能上有明显提升。同时引入了新的制品类型（`classpath-jar`、`modular-jar` 等），为 Java 模块系统提供了更好的支持。

### **其他值得注意的改进**

- **专用 BOM 打包类型**：新增 `bom` 打包类型，明确区分父 POM 和依赖管理用的 BOM
- **密码加密增强**：用真正的加密替代了 Maven 3 中实质上只是混淆的"加密"方案，提供了独立的 `mvnenc` 工具
- **插件版本警告**：未锁定版本的插件会产生警告，防止构建不可复现
- **`--fail-on-severity` 参数**：可以设定在特定日志级别（如 WARN）时中断构建
- **Maven Daemon（mvnd）和 Maven Shell（mvnsh）**：通过常驻进程消除启动开销，提升交互式开发体验

### **迁移建议**

Maven 4 对 Maven 3 项目保持了良好的向后兼容性。官方提供了迁移工具 `mvnup`，可以通过 `mvnup check` 检测潜在问题，`mvnup apply` 自动应用推荐的修复。对于大多数项目来说，升级过程应该比较平滑，但建议提前通过 `mvn clean install -Dmaven.plugin.validation=verbose` 检查所用插件的兼容性。

总的来看，Maven 4 是一次务实的现代化升级。它没有追求颠覆性的变革，而是集中解决了社区多年来反馈最强烈的痛点：多模块项目的样板配置过多、POM 发布不够干净、构建性能不够理想。对于仍在使用 Maven 的 Java 项目来说，这是一个值得期待的版本。

---

## **总结**

Maven 作为 Java 生态中最主流的构建工具，其核心价值在于：

- **标准化**：定义了统一的项目结构和构建流程，任何 Maven 项目拿到手都能快速理解和构建
- **依赖管理**：通过坐标系统和传递性依赖解析，自动处理复杂的依赖关系，不再需要手动管理 Jar 包
- **生命周期**：compile → test → package → install → deploy 的标准流程，覆盖了项目从编译到部署的全过程
- **生态丰富**：中央仓库收录了几乎所有主流的 Java 开源库，插件体系可以扩展几乎任何构建需求

尽管 Gradle 等新一代构建工具在灵活性和性能上有所优势，Maven 凭借其简洁的声明式配置、庞大的社区基础和稳定可靠的表现，仍然是大量 Java 项目的首选。理解 Maven 的核心概念——POM、坐标、生命周期、依赖范围、父子工程——是每个 Java 开发者的基本功。
