# 文件上传

- 导入pom依赖：

  ```xml
  <!--文件上传-->
  <dependency>
      <groupId>commons-fileupload</groupId>
      <artifactId>commons-fileupload</artifactId>
      <version>1.3.3</version>
  </dependency>
  <!--servlet-api导入高版本的-->
  <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>javax.servlet-api</artifactId>
      <version>4.0.1</version>
  </dependency>
  ```

- FileServlet.java：

  ```java
  package com.it.servlet;
  
  import org.apache.commons.fileupload.FileItem;
  import org.apache.commons.fileupload.FileUploadException;
  import org.apache.commons.fileupload.disk.DiskFileItemFactory;
  import org.apache.commons.fileupload.servlet.ServletFileUpload;
  
  import javax.servlet.ServletException;
  import javax.servlet.http.HttpServlet;
  import javax.servlet.http.HttpServletRequest;
  import javax.servlet.http.HttpServletResponse;
  import java.io.*;
  import java.util.List;
  import java.util.UUID;
  
  public class FileServlet extends HttpServlet  {
  
      @Override
      protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException, ServletException {
  
  //        判断上传的文件是普通表单还是带文件的表单
          if (!ServletFileUpload.isMultipartContent(req)) {
              return; //终止方法运行，如果是一个普通文件直接返回；
          }
  //            创建文件上传的保存路径，最好在WEB-INF下，用户无法直接访问
          String uploadPath = this.getServletContext().getRealPath("/WEB-INF/upload");
          File uploadFile = new File(uploadPath);
          if (!uploadFile.exists()) {
              uploadFile.mkdir(); //如国文件不存在就创建一个。
          }
  
          //消息提示
          String msg = "";
  
  
  //        缓存临时文件，如果文件大小超过了限至，就转到缓存中，过一段时间后删除，或者提醒用户永久删除
          String tempPath = this.getServletContext().getRealPath("/WEB-INF/upload");
          File temp = new File(tempPath);
          if (!temp.exists()) {
              temp.mkdir(); //如国文件不存在就创建一个。
          }
  //        使用commons-fileupload上传组件实现上传
  //        1.创建DiskFileItemFactory对象，处理文件上传l路径或者大小的限至
          DiskFileItemFactory diskFileItemFactory = new DiskFileItemFactory();
  
          /*
  //        通过这个工厂的设置一个缓冲区，当文件上传大于这个缓冲区的时候就将它放到临时文件中
          diskFileItemFactory.setSizeThreshold(1024*1024); //缓存区大小
          diskFileItemFactory.setRepository(temp);  //临时文件的保存目录
          */
  
  //        2.获取ServletFileUpload
          ServletFileUpload servletFileUpload = new ServletFileUpload(diskFileItemFactory);
          //解决上传文件名的中文乱码
          servletFileUpload.setHeaderEncoding("UTF-8");
  //      3.处理上传文件
          List<FileItem> fileItems = null;
          try {
              fileItems = servletFileUpload.parseRequest(req);
          } catch (FileUploadException e) {
              e.printStackTrace();
          }
          //fileItem每一个表单对象
          for (FileItem fileItem : fileItems) {
              //如果fileitem中封装的是普通输入项的数据
              if (fileItem.isFormField()) {
                  String name = fileItem.getFieldName(); //getFieldName()表单对象的name
                  //解决普通输入项的数据的中文乱码问题
                  String value = fileItem.getString("UTF-8");
                  //value = new String(value.getBytes("iso8859-1"),"UTF-8");
                  System.out.println(name + "=" + value);
              }else {
  
                  //======================处理文件==========================//
                  String uploadFileName = fileItem.getName();
                  System.out.println(uploadFileName);
                  if(uploadFileName==null || uploadFileName.trim().equals("")){
                      continue;
                  }
                  //处理获取到的上传文件的文件名的路径部分，只保留文件名部分
                  uploadFileName = uploadFileName.substring(uploadFileName.lastIndexOf("\\")+1);
                  //处理获取到的上传文件的后缀名例如.jpg
                  uploadFileName = uploadFileName.substring(uploadFileName.lastIndexOf(".")+1);
                  //生成为一个一个识别的通用码
                  String uuidPath = UUID.randomUUID().toString();
                  //======================存放地址==========================//
                  //文件存在的真实路径
                  String realPath = uploadPath + "/" + uuidPath;
                  //给每个文件创建一个文件夹
                  File realFile = new File(realPath);
                  if (!realFile.exists()){
                      realFile.mkdir();
                  }
                  //======================文件输出==========================//
                  InputStream inputStream = fileItem.getInputStream();
                  FileOutputStream fileOutputStream = new FileOutputStream(realPath + "/" + uploadFileName);
                  byte[] bytes = new byte[1024 * 1024];
                  int len = 0;
                  while ((len=inputStream.read(bytes))>0){
                      fileOutputStream.write(bytes,0,len);
                  }
                  fileOutputStream.close();
                  inputStream.close();
                  msg = "文件上传成功";
                  fileItem.delete();
              }
                  msg="文件上传失败";
  
          }
  
          req.setAttribute("msg",msg);
          req.getRequestDispatcher("/info.jsp").forward(req, resp);
  
  
      }
          protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
          doGet(req, resp);
      }
  }
  ```

- web.xml注册servlet

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <web-app xmlns="http://java.sun.com/xml/ns/javaee"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                        http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
           version="3.0"
           metadata-complete="true">
  
  <servlet>
    <servlet-name>FileServlet</servlet-name>
    <servlet-class>com.it.servlet.FileServlet</servlet-class>
  </servlet>
    <servlet-mapping>
      <servlet-name>FileServlet</servlet-name>
      <url-pattern>/upload.do</url-pattern>
    </servlet-mapping>
  
  </web-app>
  ```

- index.jsp:

  ```jsp
  <%@ page contentType="text/html; charset=UTF-8" language="java" %>
  <html>
  <body>
  
  <%--get上传文件大小有限制，post没有大小限制.文件上传加一个enctype--%>
  <form action="${PageContext.request.contextPath}/upload" enctype="multipart/form-data" method="post">
      <input type="text"name="text">
      <input type="file" name="file1">
      <input type="file" name="file1">
      <input type="submit">
  </form>
  
  </body>
  </html>
  ```

- info.jsp:

  ```jsp
  <%@ page contentType="text/html;charset=UTF-8" language="java" %>
  <html>
  <head>
      <title>Title</title>
  </head>
  <body>
  
  ${msg}
  
  </body>
  </html>
  ```

  