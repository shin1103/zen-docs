---
title: "Embulkを使ってMS Accessのデータを取得する方法"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Embulk", "Java", "Access"]
published: true
---
# 概要
Embulkの`embulk-input-jdbc`を使ってMS Accessのデータを取得する方法を記載します。

# 環境
- embulk 0.11.5
- openjdk 11.0.24 2024-07-16
- embulk-input-jdbc 0.13.2

# 取得方法
MS Access用のJDBCドライバとして[UCanAccess](https://sourceforge.net/projects/ucanaccess/)というドライバが公開されている(最近フォークで[Github](https://github.com/spannm/ucanaccess)にもライブラリが公開されていた。Java11以降サポートとのこと)。このJDBCドライバとEmbulkを組み合わせてアクセスできないか実験しました。

UCanAccessは`UCanaccess-5.0.1.jar`の本体のほかに以下4つのライブラリを参照。
- commons-lang3-3.8.1.jar
- commons-logging-1.2.jar
- hsqldb-2.5.0.jar
- jackcess-3.0.1.jar

Embulkの`embulk-input-jdbc`では`driver_path`を指定することで任意のJDBCドライバが指定可能です。（私が調べた限りでは）ドライバは単一のjarファイルを指定する必要があるので、fatjarを作成する必要があったので、以下の通りpom.xmlを作成しビルドしました。

```bash
mvn package
```

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>ucanaccess-fatjar</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>net.sf.ucanaccess</groupId>
            <artifactId>ucanaccess</artifactId>
            <version>5.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.hsqldb</groupId>
            <artifactId>hsqldb</artifactId>
            <version>2.5.0</version>
        </dependency>
        <dependency>
            <groupId>com.healthmarketscience.jackcess</groupId>
            <artifactId>jackcess</artifactId>
            <version>3.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.8.1</version>
        </dependency>
        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.2</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.4.2</version>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                            <mainClass>com.example.Main</mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
ビルドが完了するとtargetフォルダ以下に`ucanaccess-fatjar-1.0.0-jar-with-dependencies.jar`というファイルが出来上がるので、これをEmbulkの`driver_path`に指定してコンフィグファイルを指定しました。`driver_class`や`url`の指定方法は以下の通りに書けば指定できます。

```yml
in:
  type: jdbc
  driver_path: ./ucanaccess-fatjar-1.0.0-jar-with-dependencies.jar
  driver_class: net.ucanaccess.jdbc.UcanaccessDriver
  url: jdbc:ucanaccess:///user/ec2-user/Database1.accdb
  query: "SELECT * FROM my_tbl"
out:
  type: stdout
```

上記のように記載することでEmbulkをembulkを起動させると無事にデータ取得ができます。