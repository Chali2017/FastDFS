#### 使用FastFDS搭建文件服务平台



##### 一、docker搭建单机版fastFDS服务

###### 1.1拉取别人已经上传的镜像

docker pull qbanxiaoli/fastdfs

###### 1.2启动服务，IP要改为自己机子的ip，作者linux ip为192.168.99.100【请根据自己实际情况修改】

docker run -d --restart=always --privileged=true --net=host --name=fastdfs -e IP=192.168.99.100 -e WEB_PORT=80 -v ${HOME}/fastdfs:/var/local/fdfs qbanxiaoli/fastdfs

###### 1.3 测试服务-上传文件到fastDfs

docker exce -it fastdfs /bin/bash

echo "Hello FastDFS!">index.html

fdfs_test /etc/fdfs/client.conf upload index.html

##### 二、springboot使用fastFDS api

##### 2.1创建springboot工程

Pom.xml中导入依赖：

```
<dependency>
    <groupId>org.csource</groupId>
    <artifactId>fastdfs-client-java</artifactId>
    <version>1.27-SNAPSHOT</version>
</dependency>
```

可能内网中无法加载此依赖，可参考如下文档手动导入：

<https://www.cnblogs.com/mlq2017/p/10076084.html>

##### 2.2 resources文件夹下添加配置文件fdfs_client.conf：

```
connect_timeout = 60
network_timeout = 60
charset = UTF-8
http.tracker_http_port = 8080
http.anti_steal_token = no
http.secret_key = 123456

tracker_server = 192.168.99.100:22122
```

application.properties文件中添加**thymeleaf**模板属性配置，否则可能会出现找不到对应页面的错误：

```
spring.servlet.http.multipart.max-file-size=10MB
spring.servlet.http.multipart.max-request-size=10MB

spring.thymeleaf.content-type=text/html 
spring.thymeleaf.cache=false 
spring.thymeleaf.mode=LEGACYHTML5
```

##### 2.3创建fastDFS的代理类FastDFSClient.java，主要是利用fastFDS的api封装可用接口，重点介绍TrackerClient

```
package com.neo.fastdfs;

import org.csource.common.NameValuePair;
import org.csource.fastdfs.*;
import org.slf4j.LoggerFactory;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;

import java.io.*;

public class FastDFSClient {
	private static org.slf4j.Logger logger = LoggerFactory.getLogger(FastDFSClient.class);

	static {
		try {
			String filePath = new ClassPathResource("fdfs_client.conf").getFile().getAbsolutePath();;
			ClientGlobal.init(filePath);
		} catch (Exception e) {
			logger.error("FastDFS Client Init Fail!",e);
		}
	}

	public static String[] upload(FastDFSFile file) {
		logger.info("File Name: " + file.getName() + "File Length:" + file.getContent().length);

		NameValuePair[] meta_list = new NameValuePair[1];
		meta_list[0] = new NameValuePair("author", file.getAuthor());

		long startTime = System.currentTimeMillis();
		String[] uploadResults = null;
		StorageClient storageClient=null;
		try {
			storageClient = getTrackerClient();
			uploadResults = storageClient.upload_file(file.getContent(), file.getExt(), meta_list);
		} catch (IOException e) {
			logger.error("IO Exception when uploadind the file:" + file.getName(), e);
		} catch (Exception e) {
			logger.error("Non IO Exception when uploadind the file:" + file.getName(), e);
		}
		logger.info("upload_file time used:" + (System.currentTimeMillis() - startTime) + " ms");

		if (uploadResults == null && storageClient!=null) {
			logger.error("upload file fail, error code:" + storageClient.getErrorCode());
		}
		String groupName = uploadResults[0];
		String remoteFileName = uploadResults[1];

		logger.info("upload file successfully!!!" + "group_name:" + groupName + ", remoteFileName:" + " " + remoteFileName);
		return uploadResults;
	}

	public static FileInfo getFile(String groupName, String remoteFileName) {
		try {
			StorageClient storageClient = getTrackerClient();
			return storageClient.get_file_info(groupName, remoteFileName);
		} catch (IOException e) {
			logger.error("IO Exception: Get File from Fast DFS failed", e);
		} catch (Exception e) {
			logger.error("Non IO Exception: Get File from Fast DFS failed", e);
		}
		return null;
	}

	public static InputStream downFile(String groupName, String remoteFileName) {
		try {
			StorageClient storageClient = getTrackerClient();
			byte[] fileByte = storageClient.download_file(groupName, remoteFileName);
			InputStream ins = new ByteArrayInputStream(fileByte);
			return ins;
		} catch (IOException e) {
			logger.error("IO Exception: Get File from Fast DFS failed", e);
		} catch (Exception e) {
			logger.error("Non IO Exception: Get File from Fast DFS failed", e);
		}
		return null;
	}

	public static void deleteFile(String groupName, String remoteFileName)
			throws Exception {
		StorageClient storageClient = getTrackerClient();
		int i = storageClient.delete_file(groupName, remoteFileName);
		logger.info("delete file successfully!!!" + i);
	}

	public static StorageServer[] getStoreStorages(String groupName)
			throws IOException {
		TrackerClient trackerClient = new TrackerClient();
		TrackerServer trackerServer = trackerClient.getConnection();
		return trackerClient.getStoreStorages(trackerServer, groupName);
	}

	public static ServerInfo[] getFetchStorages(String groupName,
												String remoteFileName) throws IOException {
		TrackerClient trackerClient = new TrackerClient();
		TrackerServer trackerServer = trackerClient.getConnection();
		return trackerClient.getFetchStorages(trackerServer, groupName, remoteFileName);
	}

	public static String getTrackerUrl() throws IOException {
		return "http://"+getTrackerServer().getInetSocketAddress().getHostString()+":"+ClientGlobal.getG_tracker_http_port()+"/";
	}

	private static StorageClient getTrackerClient() throws IOException {
		TrackerServer trackerServer = getTrackerServer();
		StorageClient storageClient = new StorageClient(trackerServer, null);
		return  storageClient;
	}

	private static TrackerServer getTrackerServer() throws IOException {
		TrackerClient trackerClient = new TrackerClient();
		TrackerServer trackerServer = trackerClient.getConnection();
		return  trackerServer;
	}
}
```

启动项目，访问路径http://127.0.0.1:8080，上传文件到服务器

上图片成功：

Spring Boot - Upload Status
You successfully uploaded '企业微信截图_20190311145543.png'
file path url 'http://192.168.99.100:8080/group1/M00/00/00/wKhjZFzvZ5SAYQysAAB9ZwGWXpU903.png'

查看图片：

http://192.168.99.100/group1/M00/00/00/wKhjZFzvZ5SAYQysAAB9ZwGWXpU903.png

查看上传文件信息：

http://127.0.0.1:8080/getFile?groupName=group1&fileName=M00/00/00/wKhjZFzvZ5SAYQysAAB9ZwGWXpU903.png


特别鸣谢：
ityouknow/spring-boot-examples提供的许多springboot用例
