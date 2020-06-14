# sahara-extra

hadoop 에 기본으로 있는 [hadoop-openstack](https://github.com/apache/hadoop/tree/trunk/hadoop-tools/hadoop-openstack) 으로는 keystone v3 에서 인증을 실패하여 hdfs 에서 swift 로 접근할 수 없음

```text
Authentication Failure: Authenticate as tenant 'admin' PasswordCredentials{username='admin'}
POST http://controller.swift.local:5000/v3/auth/tokens => 400 : {"error":{"code":400,"message":"Expecting to find auth in request body. The server could not comply with the request since it is either malformed or otherwise incorrect. The client is assumed to be in error.","title":"Bad Request"}}
```

openstack swift 공식문서에서는 [sahara-extra](https://github.com/openstack/sahara-extra) 를 이용하라고 함

인증은 가능해졌지만 [SwiftFileStatus.java](https://github.com/openstack/sahara-extra/blob/master/hadoop-swiftfs/src/main/java/org/apache/hadoop/fs/swift/snative/SwiftFileStatus.java#L91) 부분에서 무한루프 돌면서 StackOverFlow 발생

```text
java.lang.StackOverflowError
  at org.apache.hadoop.fs.FileStatus.isDir(FileStatus.java:230)
  at org.apache.hadoop.fs.swift.snative.SwiftFileStatus.isDirectory(SwiftFileStatus.java:91)
  at org.apache.hadoop.fs.FileStatus.isDir(FileStatus.java:230)
  at org.apache.hadoop.fs.swift.snative.SwiftFileStatus.isDirectory(SwiftFileStatus.java:91)
...
```

~~super.isDir() || getLen() == 0 으로 수정하면 됨~~

위처럼 수정했더니 0바이트 파일이 directory 로 나오는 버그 발생함

isDirectory() 삭제 후 Store 에서 listDirectory 부분에서 directory 체크 부분에 headerNamr 이 Content-Type 이고 value 가 application/directory 인지 체크 추가함

file 은 Content-Type 이 application/octet-stream 등등..

swift 의 directory 조회시 하위 모든 object 의 metadata 를 가져오는데 너무 오래 걸림(733개 file이 있는 directory 조회시 10분 이상)

모든 object 에 head request 를 보내는 부분이 있는데 이 부분을 수정해주면 directory 조회속도 향상 가능

## HDP 3.0.1 branch

[3.1.1.3.0.1.0-187](https://github.com/HaNeul-Kim/sahara-extra/tree/3.1.1.3.0.1.0-187)

## Swift URI

hdfs dfs -ls swift://{CONTAINER_NAME}.{PROVIDER}/{OBJECT_KEY}

의 형태로 조회 가능함

CONTAINER_NAME, PROVIDER 는 java 의 URI 규칙을 따라야 함. [A-Za-z0-9\-\.] 만 가능함

container 여러개를 조회하려면 어떻게 해야할지 고민해야 함
