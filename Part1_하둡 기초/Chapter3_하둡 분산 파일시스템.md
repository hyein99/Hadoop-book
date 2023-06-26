| Part1_하둡 기초
# Chapter3_하둡 분산 파일 시스템
* 네트워크로 연결된 여러 머신의 스토리지를 관리하는 파일 시스템
# 3.1 HDFS 설계
* HDFS 설계 특성
    * 매우 큰 파일
    * 스트리밍 방식의 데이터 접근
    * 범용 하드웨어
* HDFS가 적합하지 않은 분야
    * 빠른 데이터 응답 시간
    * 수많은 작업 파일
    * 다중 라이터와 파일의 임의 수정


# 3.2 HDFS 개념
## 3.2.1 블록
* 블록: 한 번에 읽고 쓸 수 있는 데이터의 최대량
* HDFS의 블록은 128MB로 굉장히 큰 단위
    * 탐색 비용 최소화 목적
* 블록 크기보다 작은 데이터일 경우 전체 블록 크기에 해당하는 하위 디스크를 모두 점유하지 않음
* 블록 추상화 장점
    * 파일 하나의 크기가 단일 디스크의 용량보다 커질 수 있음
    * 스토리지의 서브 시스템 단순화
    * fault tolerance, availability를 제공하는 데 필요한 Replication을 구현할 때 적합
        * 블록의 손상과 디스크 장애에 대처하기 위해 각 블록은 물리적으로 분리된 다수의 머신에 복제

## 3.2.2 네임노드와 데이터노드
* 네임노드(master)
    * 파일시스템의 네임스페이스 관리
    * 파일시스템 트리와 그 트리에 포함도니 모든 파일과 디렉터리에 해한 메타데이터 유지
    * 파일에 속한 모든 블록이 어느 데이터노드에 있는지 파악
* 데이터노드(worker)
    * 클라이언트나 네임노드의 요청이 있을 때 블록을 저장하고 탐색
    * 저장하고 있는 블록의 목록을 주기적으로 네임노드에 보고
* HDFS 클라이언트
    * 네임노드, 데이터 노드 사이에서 통신하여 파일시스템 접근
* 네임노드의 장애 복구 매커니즘

## 3.2.3 블록 캐싱
* 데이터노드가 디스크에 저장된 블록을 읽을 때, 빈번하게 접근하는 블록 파일은 off-heap 블록 캐시라는 데이터노드의 메모리에 명시적으로 캐싱
* 잡 스케줄러(맵리듀스, 스파크 등)는 블록이 캐싱된 데이터노드에서 테스크가 실행되도록 할 수 있음
* ex. 룩업 테이블

## 3.2.4 HDFS 페더레이션
* 네임노드는 파일시스템의 모든 파일과 각 블록에 대한 참조 정보를 메모리에서 관리
* HDFS 페더레이션
    * 네임노드의 확장성 문제 해결하기 위한 목적
    * 각각의 네임노드가 파일시스템의 네임스페이스 일부를 나누어 관리

## 3.2.5 HDFS 고가용성
* 네임노드의 단일 고장성(single point of failure) 문제
    * 네임노드 장애 > 맵리듀스 잡 포함 모든 클라이언트 오류
* 고가용성
    * 활성대기 상태로 설정된 한 쌍의 네임노드
    * 활성 네임노드에 장애 발생하면 대기 네임노드가 그 역할 처리
### 장애복구와 펜싱
* 장애복구 컨트롤러라는 새로운 객체로 관리


# 3.3 명령행 인터페이스
* 의사분산 모드(pseudo-distributed)
    * fs.defaultFS 속성
        * 기본 값: hdfs://localhost/
        * 파일시스템은 URI로 정의
        * HDFS 데몬
            * HDFS 네임노드의 호스트와 포트를 결정하기 위해 이 속성 사용
        * HDFS 클라이언트
            * 접속할 네임노드가 실행되는 주소를 얻기 위해 이 속성 사용
    * dfs.replication 속성
        * 기본 복제 계수 설정
## 3.3.1 기본적인 파일시스템 연산
* 도움말
    ```
    hadoop fs -help
    ```
* 1) 로컬 파일 시스템의 파일 하나를 HDFS로 복사
    ```
    hadoop fs -copyFromLocal input/docs/quangle.txt hdfs://localhost/user/tom/quangle.txt
    hadoop fs -copyFromLocal input/docs/quangle.txt /user/tom/quangle.txt # hdfs://localhost 생략 가능
    hadoop fs -copyFromLocal input/docs/quangle.txt quangle.txt  # 상대 경로
    ```
    * HDFS의 쉘 명령어 fs 호출
    * 로컬 파일 > HDFS 인스턴스의 /user/tom/quangle.txt로 복사
* 2) HDFS에 복사했던 파일을 다시 로컬로 가져와서 두 파일 같은지 확인
    ```
    hadoop fs -copyToLocal quangle.txt quangle.copy.txt
    md5 input/docs/quangle.txt quangle.copy.txt
    ```
* 3) HDFS 파일 목록 확인
    ```
    hadoop fs -mkdir books
    hadoop fs -ls .
    ```


# 3.4 하둡 파일 시스템
|파일 시스템|URL 스킴|자바 구현체|설명|
|--------|------|--------|---|
|Local|file|fs.LocalFileSystem|클라이언트 측의 체크섬을 사용하는 로컬 디스크를 위한 FS|
|HDFS|hdfs|hdfs.DistributedFileSystem|하둡분산파일 시스템|
|WebHDFS|webhdfs|hdfs.web.WebHdfsFileSystem|HTTP를 통해 HDFS에 인증|
* 로컬 파일 시스템의 루트 디렉터리 파일 목록
```
hadoop fs -ls file:///
```
## 3.4.1 인터페이스
### HTTP
* WebHDFS 프로토콜을 이용한 HTTP REST API를 사용하면 다른 언어도 HDFS에 쉽게 접근 가능
* HTTP로 HDFS에 접근하는 방법
    * 클라이언트의 HTTP 요청을 HDFS 데몬이 직접 처리
        * 네임노드와 데이터 노드에 저장된 웹서버가 WebHDFS의 말단으로 작용
    * 클라이언트 대신 DistributedFileSystem API로 HDFS에 접근하는 프록시 경유
        * 하나 또는 그 이상의 독립 프록시 서버 통하기
### C
### NFS
### FUSE


# 3.5 자바 인터페이스
## 3.5.1 하둡 URL로 데이터 읽기
```java
public class URLCat {
    static {
        // URL 클래스의 setURLStreamHandler Factory 메소드 호출
        // JVM 하나당 한 번씩만 호출 > static
        URL.setURLStreamHandlerFactory(new FsUrlStreamhandlerFactory());
    }
    public static void main(Stringp[] args) throws Exception{
        InputStream in = null;
        try {
            in = new URL(args[0]).openStream();
            // in 을 처리한다
            IOUtils.copyBytes(in, System.out 4096, false);
        } finally {
            IOUtils.closeStream(in);
        }
    }
}
```

```
export HADOOP_CLASSPATH=hadoop-examples.jar
hadoop URLCat hdfs://localhost/user/tom/quangle.txt
```

## 3.5.2 파일시스템 API로 데이터 읽기


## 3.5.3 데이터 쓰기

## 3.5.4 디렉터리

## 3.5.5 파일시스템 질의

## 3.5.6 데이터 삭제


# 3.6 데이터 흐름
## 3.6.1 파일 읽기 상세

## 3.6.2 파일 쓰기 상세

## 3.6.3 일관성 모델


# 3.7 distcp로 병렬 복사하기
## 3.7.1 HDFS 클러스터 균형 유지