# Chapter0_설치
| 부록 A 참조

## 1. Java 설치
```shell
java --version
```

## 2. Hadoop 설치
https://www.apache.org/dyn/closer.cgi#
```shell
export JAVA_HOME=/Library/Java/JavaVirtualMachines/adoptopenjdk-11.jdk/Contents/Home
export HADOOP_HOME=/Library/Hadoop/hadoop-3.3.5
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin

hadoop version
```

## 3. 환경 설정
### HDFS 파일 시스템 포맷하기
```shell
hdfs namenode -format
```

## 의사분산모드
의사 분산모드로 사용하려면,,
* config
* SSH
* 데몬의 시작과 중지
* 사용자 디렉토리 설정