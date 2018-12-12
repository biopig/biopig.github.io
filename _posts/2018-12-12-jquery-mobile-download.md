---
layout: post
title: JQuery Mobile下载文件
categories:Jquery Mobile
description: JQuery Mobile如何下载文件
keywords: Web App,Jquery Mobile
---

JQuery Mobile的请求基本上都是通过Ajax来完成的，但是ajax请求只是个“字符型”的请求，即请求的内容是以文本类型存放的,文件的下载是以二进制形式进行的也就是流，ajax是不能返回的，虽然可以读取到返回的response，但只是读取而已，是无法执行的。 

解决方案，不用ajax：


使用XHR 全称：XMLHttpRequest



废话不多说直接上代码吧：

**1. 前端请求**

         function downloadAttachments(attachmentId) {
    		   var url = 'http://localhost/KRISADMIN/downloadAttachment';
    		   var xhr = new XMLHttpRequest();
    		   xhr.open('POST', url);// 也可以使用POST方式，根据接口,也可以写get
    		  
			  xhr.responseType = "arraybuffer";
    		  
			  // 定义请求完成的处理函数，请求前也可以增加加载框/禁用下载按钮逻辑
    		   xhr.onload = function () {
		    		   // 请求完成
		    		   if (this.status === 200) {
			    		   // 返回200
			    		   var blob = new Blob([this.response], {type: "application/octet-stream"});
			    		   var reader = new FileReader();
			    		   reader.readAsDataURL(blob);// 转换为base64，可以直接放入a表情href
			    		   
							reader.onload = function (e) {
			    			 //手机浏览器 必须这么保存文件才能正确打开,因为当返回的流很
							//长的时候IE浏览器会报“传递给系统调用的数据区域太小”的错,同时手机浏览器会下载错误的文件
			    		   if(navigator.msSaveBlob){ // IE 10+ 
			    		   	navigator.msSaveBlob(new Blob([blob],{type:'application/octet-stream; charset = utf-8;'}),filename); 
			    		   }else{
			    			  // 转换完成，创建一个a标签用于下载
			    			  var a = document.createElement('a');
			    			   a.download = filename;
			    			   a.href = e.target.result;
			    			   
			    			   $("body").append(a);// 修复firefox中无法触发click
			    			   a.click();
			    			   $(a).remove();  
			    		  }
		    		   }
	    		     }
    		   };
    		   // 发送ajax请求
    		   xhr.send(attachmentId)
    	}

**2. 服务器端**

我项目是ssh的，用了rest的插件，接口都是restful接口暴露的。接口如下：

	@Action(value="/downloadAttachment")
	public void  downloadAttachment() throws IOException{
		ActionContext ctx = ActionContext.getContext();
		String attachmentId = getRequestBody(ctx);
		attachmentId=attachmentId.replaceAll("\"", "");
		WFAttachment wfattachment=workFlowService.findAttachmentByID(Integer.parseInt(attachmentId));
		fileName=wfattachment.getFileName();
		HttpServletRequest request = ServletActionContext.getRequest();
		HttpServletResponse response=ServletActionContext.getResponse();
		 
		String userAgent = request.getHeader("User-Agent");  
	       if (null != userAgent && -1 != userAgent.indexOf("Firefox")){ //Firefox  
	        	 fileName = new String(fileName.getBytes("UTF-8"),"iso-8859-1");     
	       }else{     
	    	   fileName = URLEncoder.encode(fileName, "UTF-8"); 
	    	   fileName = fileName.replace("+", "%20");
	       }
	       
		   String extension =fileName.substring(fileName.lastIndexOf(".")+1);
		   String path=wfattachment.getSavePath();
		   String strOldNmme = path +"/"+ attachmentId+"."+extension;
		 

		//上面代码都是在我项目中获取文件的过程，文件获取到之后才是重点
		 File f = new File(strOldNmme);
		 
		   
		//导出文件
		String agent = request.getHeader("User-Agent").toUpperCase();
		
		InputStream fis = null;
		OutputStream os = null;
		try {
		    fis = new BufferedInputStream(new FileInputStream(f.getPath()));
		    byte[] buffer;
		    buffer = new byte[fis.available()];
		    fis.read(buffer);
		    response.reset();
		   
			
			//我注释的这段代码是因为在我系统获取fileName时，已经做过这个操作了，就不需要了判断了直接addHeader就行了，你若果在你系统获取filename是没有做此操作，则需要这段代码进行判断添加
			
			/* //由于火狐和其他浏览器显示名称的方式不相同，需要进行不同的编码处理
		    if(agent.indexOf("FIREFOX") != -1){//火狐浏览器
		    	response.addHeader("Content-Disposition", "attachment;filename="+ new String(fileName.getBytes("GB2312"),"ISO-8859-1"));
		    }else{//其他浏览器
		    	response.addHeader("Content-Disposition", "attachment;filename="+ URLEncoder.encode(fileName, "UTF-8"));
		    }*/
		    response.addHeader("Content-Disposition", "attachment;filename="+ fileName);
		    

			//设置response编码
		    response.setCharacterEncoding("UTF-8");
		    response.addHeader("Content-Length", "" + f.length());
		    //设置输出文件类型(有很多具体可以看setContentType参数，application/octet-stream就是二进制流)  
		    response.setContentType("application/octet-stream");
		    //获取response输出流
		    os = response.getOutputStream();
		    // 输出文件
		    os.write(buffer);
		}catch(Exception e){
		    System.out.println(e.getMessage());
		} finally{
		    //关闭流
		    try {
		        if(fis != null){
		            fis.close();
		        }
		    } catch (IOException e) {
		        System.out.println(e.getMessage());
		    } finally{
		        try {
		            if(os != null){
		                os.flush();
		            }
		        } catch (IOException e) {
		            System.out.println(e.getMessage());
		        } finally{
		            try {
		                if(os != null){
		                    os.close();
		                }
		            } catch (IOException e) {
		                System.out.println(e.getMessage());
		            }
		        }
		    }
		}
		  
		 
	}
效果图：

![](/images/posts/jquerymobile/download-doc.png)

有什么不清楚的，留言一起学习

2018/12/12 18:05:42 