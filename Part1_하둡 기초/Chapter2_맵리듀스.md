| Part1_하둡 기초
# Chapter2_맵리듀스
* 데이터 처리를 위한 프로그래밍 모델
# 2.1 기상 데이터셋

# 2.2 유닉스 도구로 데이터 분석하기

# 2.3 하둡으로 데이터 분석하기
## 2.3.1 맵과 리듀스
* 맵
  * 입력: 키-값 
  * 출력: 연도와 기온
* 리듀스
  * 입력: 키(연도)-값(기온 리스트)
  * 출력: 키(연도)-값(max)
## 2.3.2 자바 맵리듀스
### mapper
* 제네릭 타입
* 매개변수
    * 입력키: long integer의 오프셋(`LongWritable`)
    * 입력값: 한 행의 내용(`Text`)
    * 출력키: 연도(`Text`)
    * 출력값: 기온(`IntWritable`)
<details>
<summary>MaxTemperatureMapper.java</summary>

```java
// cc MaxTemperatureMapper Mapper for maximum temperature example
// vv MaxTemperatureMapper
import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

public class MaxTemperatureMapper
  extends Mapper<LongWritable, Text, Text, IntWritable> {

  private static final int MISSING = 9999;
  
  @Override
  public void map(LongWritable key, Text value, Context context)
      throws IOException, InterruptedException {
    
    String line = value.toString();
    String year = line.substring(15, 19);
    int airTemperature;
    if (line.charAt(87) == '+') { // parseInt doesn't like leading plus signs
      airTemperature = Integer.parseInt(line.substring(88, 92));
    } else {
      airTemperature = Integer.parseInt(line.substring(87, 92));
    }
    String quality = line.substring(92, 93);
    if (airTemperature != MISSING && quality.matches("[01459]")) {
      context.write(new Text(year), new IntWritable(airTemperature));
    }
  }
}
// ^^ MaxTemperatureMapper

```
</details>

### reducer
* 매개변수
    * 입력키: 연도(`Text`)
    * 입력값: 기온(`IntWritable`)
    * 출력키: 연도(`Text`)
    * 출력값: 최고기온(`IntWritable`)
<details>
<summary>MaxTemperatureReducer.java</summary>

```java
// cc MaxTemperatureReducer Reducer for maximum temperature example
// vv MaxTemperatureReducer
import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

public class MaxTemperatureReducer
  extends Reducer<Text, IntWritable, Text, IntWritable> {
  
  @Override
  public void reduce(Text key, Iterable<IntWritable> values,
      Context context)
      throws IOException, InterruptedException {
    
    int maxValue = Integer.MIN_VALUE;
    for (IntWritable value : values) {
      maxValue = Math.max(maxValue, value.get());
    }
    context.write(key, new IntWritable(maxValue));
  }
}
// ^^ MaxTemperatureReducer


```
</details>

### job
* job 명세서
    * 하둡 클러스터에서 잡 실행할 때 먼저 코드를 JAR파일로 묶어야 함
    * 하둡은 클러스터의 해당 머신에 JAR파일 배포
    * Job의 `setJarByClass()` method를 통해 class 지정
        * 하둡이 알아서 해당 class 포함한 JAR파일 찾아서 클러스터에 배치
* 입출력 경로 지정(`FileOutputFormat`)
    * 입력 경로: `addInputPath()`
    * 출력 경로: `setOutputPath()`
        * 리듀스 함수가 출력파일을 저장할 디렉토리
* 입출력 데이터 타입 지정
    * `setMapperClass()`
    * `setReducerClass()`
<details>
<summary>MaxTemperature.java</summary>

```java
// cc MaxTemperature Application to find the maximum temperature in the weather dataset
// vv MaxTemperature
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class MaxTemperature {

  public static void main(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.println("Usage: MaxTemperature <input path> <output path>");
      System.exit(-1);
    }
    
    Job job = new Job();
    job.setJarByClass(MaxTemperature.class);
    job.setJobName("Max temperature");

    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    
    job.setMapperClass(MaxTemperatureMapper.class);
    job.setReducerClass(MaxTemperatureReducer.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
// ^^ MaxTemperature


```
</details>

### hadoop 설치
* JAVA 설치
  * bash_profile
  ```
  JAVA_HOME=/Library/Java/JavaVirtualMachines/adoptopenjdk-11.jdk/Contents/Home
  PATH=$PATH:$JAVA_HOME/bin
  export JAVA_HOME
  export PATH
  ```
* mvn 설치
  * bash_profile
  ```
  M3_HOME=/Library/apache-maven-3.9.2
  PATH=$PATH:$M3_HOME/bin
  export M3_HOME
  export PATH
  ```
* hadoop 설치
  * 참조 https://key4920.github.io/docs/bigdata_platform/Hadoop/hadoop_install_M1/

### 테스트 수행
* 독립 모드로 하둡 설치(부록A 참조)
  * 독립모드: 로컬 파일 시스템과 로컬 잡 수행자로 맵리듀스 잡 실행
* 웹 사이트 예제 설치하고 컴파일
```
export HADOOP_CLASSPATH=hadoop-examples.jar
hadoop MasTemperature input/ncdc/sample.txt output
```

# 2.4 분산형으로 확장하기
## 2.4.1 데이터 흐름
* 맵리듀스 job
  * 클라이언트가 수행하는 작업의 기본 단위
  * 입력 데이터, 맵리듀스 프로그램, 설정 정보로 구성
* YARN
  * task scheduling
* 하둡의 job 실행 방법
  * map task, reduce task로 나누어 실행
    * map task
      * input split 또는 split이라고 부르는 고정 크기 조각으로 분리
        * split 크기는 일반적으로 HDFS 블록의 기본 크기인 128MB가 적당
      * 각 split마다 하나의 map task를 생성하여 split의 각 레코드를 사용자 정의 map 함수로 처리
      * map task의 결과는 로컬 디스크에 저장(map 결과는 중간 결과물. job이 완료되면 버려짐)
    * reduce task
      * 모든 map task는 하나의 reduce task에 연결
      * 정렬된 map의 모든 결과는 reduce task가 실행중인 노드로 전송

## 2.4.2 컴바이너 함수
* combiner function
  * map task와 reduce task 사이 데이터 전송 최소화 목적
  * combiner function의 출력 = reduce function의 입력
  * 제한적 사용
    * reduce function을 완전히 대체할 수 없음
      * ex. min 은 되는데 mean 은 안됨
### 컴바이너 함수 작성하기
* reduce 클래스 사용하여 정의
* job 설정에 컴바이너 클래스 추가 지정 필요
<summary>MaxTemperatureWithCombiner.java</summary>

```java

// vv MaxTemperatureWithCombiner
public class MaxTemperatureWithCombiner {

  public static void main(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.println("Usage: MaxTemperatureWithCombiner <input path> " +
          "<output path>");
      System.exit(-1);
    }
    
    Job job = new Job();
    job.setJarByClass(MaxTemperatureWithCombiner.class);
    job.setJobName("Max temperature");

    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    
    job.setMapperClass(MaxTemperatureMapper.class);
    /*[*/job.setCombinerClass(MaxTemperatureReducer.class)/*]*/;
    job.setReducerClass(MaxTemperatureReducer.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
// ^^ MaxTemperatureWithCombiner

```
</details>

## 2.4.3 분산 맵리듀스 잡 실행하기

# 2.5 하둡 스트리밍
* 하둡은 java외에도 다른 언어로 맵과 리듀스 함수 작성할 수 있는 map reduce API 제공
## 2.5.1 루비

## 2.5.2 파이썬
* max_temperature_map.py
```python
#!/usr/bin/env python

import re
import sys

for line in sys.stdin:
  val = line.strip()
  (year, temp, q) = (val[15:19], val[87:92], val[92:93])
  if (temp != "+9999" and re.match("[01459]", q)):
    print "%s\t%s" % (year, temp)

```
* max_temperature_reduce.py
```python
#!/usr/bin/env python

import sys

(last_key, max_val) = (None, -sys.maxint)
for line in sys.stdin:
  (key, val) = line.strip().split("\t")
  if last_key and last_key != key:
    print "%s\t%s" % (last_key, max_val)
    (last_key, max_val) = (key, int(val))
  else:
    (last_key, max_val) = (key, max(max_val, int(val)))

if last_key:
  print "%s\t%s" % (last_key, max_val)
```