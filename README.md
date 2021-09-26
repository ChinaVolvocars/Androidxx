# 最新Android aar发布到Maven Central

## 注册
前往`https://issues.sonatype.org`注册账号


## Gradle准备

### 新建`publish-mavencentral.gradle`文件

```
apply plugin: 'maven-publish'
apply plugin: 'signing'

task androidSourcesJar(type: Jar) {
    archiveClassifier.set('sources')
    from android.sourceSets.main.java.srcDirs
}

task androidJavadocs(type: Javadoc) {
    source = android.sourceSets.main.java.srcDirs
    excludes = ['**/*.kt'] // Exclude all kotlin files from javadoc file.
    classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
}

task androidJavadocsJar(type: Jar, dependsOn: androidJavadocs) {
    archiveClassifier.set('javadoc')
    from androidJavadocs.destinationDir
}

artifacts {
    archives androidSourcesJar
    archives androidJavadocsJar
}
//加载资源
Properties properties = new Properties()
InputStream inputStream = project.rootProject.file('local.properties').newDataInputStream();
properties.load(inputStream)

// Because the components are created only during the afterEvaluate phase, you must
// configure your publications using the afterEvaluate() lifecycle method.
afterEvaluate {
    publishing {
        publications {
            // Creates a Maven publication called "release".
            release(MavenPublication) {
                // Applies the component for the release build variant.
                from components.release

                // You can then customize attributes of the publication as shown below.
                groupId = properties.getProperty('GROUP_ID')
                artifactId = project.artifactId
                version = project.artifactVersion

                // Adds Javadocs and Sources as separate jars.
                artifact androidSourcesJar
                artifact androidJavadocsJar

                pom {
                    name = project.artifactName
                    packaging = 'aar'
                    description = project.artifactDescription
                    url = project.githubUrl

                    licenses {
                        license {
                            name = 'The Apache License, Version 2.0'
                            url = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                        }
                    }

                    developers {
                        developer {
                            id = 'OverWatch'
                            name = 'OverWatch'
                            email = 'OverWatch@126.com'
                        }
                    }

                    scm {
                        url = project.githubUrl
                        connection = "scm:git@${project.githubHttps}"
                        developerConnection = "scm:${project.githubSSH}"
                    }

                }
            }
        }

        repositories {
            maven {
                def releasesRepoUrl = "https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/"
                def snapshotsRepoUrl = "https://s01.oss.sonatype.org/content/repositories/snapshots/"
                url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl
                credentials {
                    username = properties.getProperty('OSSRH_USERNAME')
                    password = properties.getProperty('OSSRH_PASSWORD')
                }
            }
        }
    }

    signing {
        required { gradle.taskGraph.hasTask("publish") }
        useInMemoryPgpKeys(
                properties.getProperty('GPG_KEY_ID'),
                properties.getProperty('GPG_KEY'),
                properties.getProperty('GPG_KEY_PASSWORD')
        )
        sign publishing.publications.release
    }
}
```

### 在库的build.gradle添加下面的信息
```
apply from: "${rootProject.projectDir}/publish-mavencentral.gradle"

ext {
    artifactId = "helper"
    artifactName = 'helper'
    artifactDescription = "androidx helper"
    artifactVersion = "1.0.0"
    githubUrl = "https://github.com/ChinaVolvocars/OSSRH-73545"
    githubHttps = "https://github.com/ChinaVolvocars/OSSRH-73545.git"
    githubSSH = "git@github.com:ChinaVolvocars/OSSRH-73545.git"
}
```

### local.properties文件添加

```
GROUP_ID=xx.xxx.xxxxx
OSSRH_USERNAME=sonatype帐号
OSSRH_PASSWORD=sonatype密码
GPG_KEY_ID=xxx
GPG_KEY_PASSWORD=xxx
GPG_KEY=E\:\\Maven_0x486BD222_public.asc
```

## 签名密钥申请
下载GPG管理器客户端`https://www.gpg4win.org/get-gpg4win.html`
下载安装完成之后，执行下面的命令

### 生成秘钥
使用`gpg --gen-key`，输入名字和密码
```
PS C:\Users\Tiger> gpg --gen-key
gpg (GnuPG) 2.2.28; Copyright (C) 2021 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Note: Use "gpg --full-generate-key" for a full featured key generation dialog.

GnuPG needs to construct a user ID to identify your key.

Real name: Maven
Email address: atlantisspeed@gmail.com
You selected this USER-ID:
    "Maven <atlantisspeed@gmail.com>"

Change (N)ame, (E)mail, or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
generator a better chance to gain enough entropy.
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: key 67599438486BD222 marked as ultimately trusted
public and secret key created and signed.

pub   rsa3072 2021-09-26 [SC] [expires: 2023-09-26]
uid                      Maven <atlantisspeed@gmail.com>
sub   rsa3072 2021-09-26 [E] [expires: 2023-09-26]
```

### 密钥上传到GPG服务器
> 486BD222 是上面生成的`gpg: key 67599438486BD222 marked as ultimately trusted` 后八位

上传命令`gpg --keyserver hkp://keyserver.ubuntu.com --send-keys 486BD222`
检查是否上传成功` gpg --keyserver hkp://keyserver.ubuntu.com --search-keys 486BD222`

```
PS C:\Users\Tiger> gpg --keyserver hkp://keyserver.ubuntu.com --send-keys 486BD222
gpg: sending key 67599438486BD222 to hkp://keyserver.ubuntu.com
PS C:\Users\Tiger> gpg --keyserver hkp://keyserver.ubuntu.com --search-keys 486BD222
gpg: data source: http://162.213.33.9:11371
(1)     Maven <atlantisspeed@gmail.com>
          3072 bit RSA key 67599438486BD222, created: 2021-09-26
Keys 1-1 of 1 for "486BD222".  Enter number(s), N)ext, or Q)uit >
```

### 导出密钥
在软件上右键密钥->导出->保存
> 此文件的地址就是`local.properties`文件中的`GPG_KEY`的地址

## 发布
点击 `publish` 发布