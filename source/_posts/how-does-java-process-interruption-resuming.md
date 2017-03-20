---
title: java如何正确处理断点续传
date: 2011-12-16 21:27:41
description: 
categories:
- Java
tags:
- Java
toc: true
author: John Tang
comments:
original:
permalink: 
---

关于JAVA如何正确的处理静态文件，数据流的断点续传，查阅许多资料后，还是在tomcat中找到了灵感，
具体请参考[org.apache.catalina.servlets.DefaultServlet ](http://www.docjar.com/html/api/org/apache/catalina/servlets/DefaultServlet.java.html)

简单说明下，关于tomcat处理资源请求的主要包括：

1. 处理流程在 protected void serveResource(HttpServletRequest request,HttpServletResponse response,boolean content) 中
2. 使用缓存进行处理 CacheEntry cacheEntry = resources.lookupCache(path); 
3. 解析Range的正确方式请参考 


		// Parse range specifier
		ranges = parseRange(request, response, cacheEntry.attributes);
		
		// ETag header
		response.setHeader("ETag", cacheEntry.attributes.getETag());
		 
		// Last-Modified header
		response.setHeader("Last-Modified",
		cacheEntry.attributes.getLastModifiedHttp());
		
		// Get content length
		contentLength = cacheEntry.attributes.getContentLength();
		// Special case for zero length files, which would cause a
		// (silent) ISE when setting the output buffer size
		if (contentLength == 0L) {
		 content = false;


这个方法才是重点 **protected Range parseContentRange(HttpServletRequest request,HttpServletResponse response)**

经测试，完美支持CHROME,IE,FF,ANDROID,IPAD,IPHONE,SAFARI。

大家可以参考来处理静态资源或是数据流
下面的是整合后的代码，有删改


	 public String readFile(){
		   long start = System.currentTimeMillis();
		   //open virtual dir file in cloud 
		   mongodbfile cloudStore = new mongodbfile();
		   BufferedOutputStream bos = null;
		   String fileId = "";
		   getResponse().reset();
		   getResponse().resetBuffer();
		   int bufferSize = 1024;
		   getResponse().setBufferSize(bufferSize);
		   try {
			 getResponse().setContentType("image/");//请相应修改
			 getResponse().setHeader("Accept-Ranges", "bytes");
			   log.debug("readCloudStoreFile fileId = {}",fileId);
			   int readResult = cloudStore.ReadBegin(fileId);
			   if(readResult == 0){//文件读取成功
				   long contentLength = cloudStore.GetFileLength(fileId);
	                           //模拟Etag的信息
				   getResponse().setHeader("Etag", "W/\""+contentLength+"-1316052976000\"");
				   getResponse().setHeader("Last-Modified", "Thu, 15 Sep 2011 02:16:16 GMT");
				   byte[] buffer = new byte[bufferSize];
				   bos = new BufferedOutputStream(getResponse().getOutputStream());
				   long pos = 0;			   
				   ArrayList ranges = null;			   
				   long s = System.currentTimeMillis();
				   // Parse range specifier
	                           ranges = parseRange(request, response, contentLength);
	                           long requestLength = 0;			 
	               if ( (((ranges == null) || (ranges.isEmpty()))
	                               && (request.getHeader("Range") == null) )
	                       || (ranges == FULL) ){
	            	   if (contentLength < Integer.MAX_VALUE) {
	    				   getResponse().setContentLength((int) contentLength);
	                   } else {
	                       // Set the content-length as String to be able to use a long
	                	   getResponse().setHeader("content-length", "" + contentLength);
	                   }
	            	   requestLength = contentLength;
	               }else{
			// 若客户端传来Range，说明之前下载了一部分，设置206状态(SC_PARTIAL_CONTENT)
					   getResponse().setStatus(HttpServletResponse.SC_PARTIAL_CONTENT);
					   log.debug("=======ranges.size() {}",ranges.size());
					   if (ranges.size() == 1) {
	
			                Range range = (Range) ranges.get(0);
			                response.addHeader("Content-Range", "bytes "
			                                   + range.start
			                                   + "-" + range.end + "/"
			                                   + range.length);
			                log.debug("Content-Range:  {}","bytes "
	                                + range.start
	                                + "-" + range.end + "/"
	                                + range.length);
	//重点在这里对实时请求的数据长度和数据流剩余长度，和写回Response数据长度的判断
			                long length = range.end - range.start + 1;
			                if (length < Integer.MAX_VALUE) {
			                    response.setContentLength((int) length);
			                } else {
			                    // Set the content-length as String to be able to use a long
			                    response.setHeader("content-length", "" + length);
			                }
			                log.debug("request range length = {}",length);    
			                pos = range.start;
			                requestLength = length;
			            } else {
	                                //这里未处理range数组的情况
			                response.setContentType("multipart/byteranges; boundary="
			                                        + mimeSeparation);
		                    log.warn("@@@@@@@@@@@ for multipart transfer todo");
			            }   
				   }
	               log.debug("###########skip pos = {}",pos);
				   if (pos != 0) {
					   // 略过已经传输过的字节
					   long sk = System.currentTimeMillis();
					   cloudStore.Skip(pos);				  
				   }
				   long ws = System.currentTimeMillis();
				   if(requestLength < bufferSize){
					   bufferSize = (int)requestLength;
				   }
				   int readedLen = 0;
				   for(long len = 0;len<requestLength;){
					   readedLen = cloudStore.Read(buffer, bufferSize);
					   bos.write(buffer,0,readedLen);
					   len+=readedLen;
				   }
				   bos.flush();
				   log.debug("write data spent {}",System.currentTimeMillis()-ws);
			   }
			   log.debug("write to response file success fileId = {}",fileId);
		   } catch (Exception e) {
		   }finally{
			   //关闭资源	   
		   return null;
	   }
	        /**
		 * MIME multipart separation string
		 */
		protected static final String mimeSeparation = "CATALINA_MIME_BOUNDARY";
	
	   /**
	    * Full range marker.
	    */
	   protected static ArrayList FULL = new ArrayList();
	   /**
	    * Parse the range header.
	    *
	    * @param request The servlet request we are processing
	    * @param response The servlet response we are creating
	    * @return Vector of ranges
	    */
	   protected ArrayList parseRange(HttpServletRequest request,
	                               HttpServletResponse response,long length)
	       throws IOException {
	
	       // Checking If-Range
	       String headerValue = request.getHeader("If-Range");
	       log.debug("headerValue = ",headerValue);
	       if (headerValue != null) {
	
	           long headerValueTime = (-1L);
	           try {
	               headerValueTime = request.getDateHeader("If-Range");
	           } catch (IllegalArgumentException e) {
	               ;
	           }
	
	           String eTag = "W/\""+length+"-1316052976000\"";
	           long lastModified = 1316052976000L;
	
	           log.debug("headerValueTime = ",headerValueTime);
	           if (headerValueTime == (-1L)) {
	
	               // If the ETag the client gave does not match the entity
	               // etag, then the entire entity is returned.
	               if (!eTag.equals(headerValue.trim())){
	            	   log.debug("If the ETag the client gave does not match the entity etag, "
	                            +"then the entire entity is returned.");
	            	   return FULL;
	               }
	           } else {
	
	               // If the timestamp of the entity the client got is older than
	               // the last modification date of the entity, the entire entity
	               // is returned.
	               if (lastModified > (headerValueTime + 1000)){
	            	   log.debug("If the timestamp of the entity the client got is older "
	            +"than the last modification date of the entity, the entire entity is returned");
	            	   return FULL;
	               }
	           }
	
	       }
	
	       long fileLength = length;
	
	       if (fileLength == 0)
	           return null;
	
	       // Retrieving the range header (if any is specified
	       String rangeHeader = request.getHeader("Range");
	
	       log.debug("rangeHeader = ",rangeHeader);
	       if (rangeHeader == null){
	    	   return null;
	       }    
	       // bytes is the only range unit supported (and I don't see the point
	       // of adding new ones).
	       if (!rangeHeader.startsWith("bytes")) {
	           response.addHeader("Content-Range", "bytes */" + fileLength);
	           response.sendError
	               (HttpServletResponse.SC_REQUESTED_RANGE_NOT_SATISFIABLE);
	           return null;
	       }
	
	       rangeHeader = rangeHeader.substring(6);
	
	       // Vector which will contain all the ranges which are successfully
	       // parsed.
	       ArrayList<range> result = new ArrayList</range><range>();
	       StringTokenizer commaTokenizer = new StringTokenizer(rangeHeader, ",");
	
	       // Parsing the range list
	       while (commaTokenizer.hasMoreTokens()) {
	           String rangeDefinition = commaTokenizer.nextToken().trim();
	
	           Range currentRange = new Range();
	           currentRange.length = fileLength;
	
	           int dashPos = rangeDefinition.indexOf('-');
	
	           if (dashPos == -1) {
	               response.addHeader("Content-Range", "bytes */" + fileLength);
	               response.sendError
	                   (HttpServletResponse.SC_REQUESTED_RANGE_NOT_SATISFIABLE);
	               return null;
	           }
	
	           if (dashPos == 0) {
	
	               try {
	                   long offset = Long.parseLong(rangeDefinition);
	                   currentRange.start = fileLength + offset;
	                   currentRange.end = fileLength - 1;
	               } catch (NumberFormatException e) {
	                   response.addHeader("Content-Range",
	                                      "bytes */" + fileLength);
	                   response.sendError
	                       (HttpServletResponse
	                        .SC_REQUESTED_RANGE_NOT_SATISFIABLE);
	                   return null;
	               }
	
	           } else {
	
	               try {
	                   currentRange.start = Long.parseLong
	                       (rangeDefinition.substring(0, dashPos));
	                   if (dashPos < rangeDefinition.length() - 1)
	                       currentRange.end = Long.parseLong
	                           (rangeDefinition.substring
	                            (dashPos + 1, rangeDefinition.length()));
	                   else
	                       currentRange.end = fileLength - 1;
	               } catch (NumberFormatException e) {
	                   response.addHeader("Content-Range",
	                                      "bytes */" + fileLength);
	                   response.sendError
	                       (HttpServletResponse
	                        .SC_REQUESTED_RANGE_NOT_SATISFIABLE);
	                   return null;
	               }
	
	           }
	
	           if (!currentRange.validate()) {
	               response.addHeader("Content-Range", "bytes */" + fileLength);
	               response.sendError
	                   (HttpServletResponse.SC_REQUESTED_RANGE_NOT_SATISFIABLE);
	               return null;
	           }
	
	           result.add(currentRange);
	       }
	
	       return result;
	   }
	   // ------------------------------------------------------ Range Inner Class
	
	
	   protected class Range {
	
	       public long start;
	       public long end;
	       public long length;
	
	       /**
	        * Validate range.
	        */
	       public boolean validate() {
	           if (end >= length)
	               end = length - 1;
	           return ( (start >= 0) && (end >= 0) && (start < = end)
	                    && (length > 0) );
	       }
	
	       public void recycle() {
	           start = 0;
	           end = 0;
	           length = 0;
	       }
	
	   }