## NSURLSession 发送Get、Post请求

下面我们主要在OC与Swift两个环境中学习使用NSURLSession进行网络请求。

### 一 、OC中进行Get请求

使用NSURLSession发送Get请求过程如下：
 
 *  确定请求路径，使用NSRUL定义。
 *  创建一个请求对象（NSURLRequest）
 *   创建一个会话对象（NSURLSession）
 *   使用NSURLSession 以及requst对象，创建一个请求任务（NSURLSessionDataTask）
 *   执行task
 
 
 	   
 	    //定义请求路径URL
	    NSURL * url = [NSURL URLWithString:@"https://yt-adp.ws.126.net/channel4/300130_aznh_20190313.jpg"];
	    //创建request
	    NSURLRequest * request = [NSURLRequest requestWithURL:url];
	    //创建session
	    _session = [NSURLSession sharedSession];
	    //创建task
	    NSURLSessionDataTask * task = [_session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
	        NSLog(@"下载完成，%@",[NSThread currentThread]);
	        dispatch_async(dispatch_get_main_queue(), ^{
	            if (self.comBlock) {
	                self.comBlock(data);
	            }
	        });
	    }];
	    [task resume];


### 二、OC中进行Post请求

Post请求与Get相类似，

 *  确定请求路径，使用NSRUL定义。
 *  创建一个`可变`请求对象（NSMutableURLRequest）
 *  修改request的请求方式为“POST”
 *   创建一个会话对象（NSURLSession）
 *   使用NSURLSession 以及requst对象，创建一个请求任务（NSURLSessionDataTask）
 *   执行task
 
 
		 //1.创建会话对象
		NSURLSession *session = [NSURLSession sharedSession];
		 
		 //2.根据会话对象创建task
		 NSURL *url = [NSURL URLWithString:@"http://120.25.226.186:32812/login"];
		//3.创建可变的请求对象
		NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
		//4.修改请求方法为POST
		request.HTTPMethod = @"POST";
		 
		 //5.设置请求体
		 request.HTTPBody = [@"username=520it&pwd=520it&type=JSON" dataUsingEncoding:NSUTF8StringEncoding];
		
		//6.根据会话对象创建一个Task(发送请求）
		 /*
		  第一个参数：请求对象
		  第二个参数：completionHandler回调（请求完成【成功|失败】的回调）
		            data：响应体信息（期望的数据）
		               response：响应头信息，主要是对服务器端的描述
		               error：错误信息，如果请求失败，则error有值
		    */
		 NSURLSessionDataTask *dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
		  
		    //8.解析数据
		      NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil];
		       NSLog(@"%@",dict);
		       
		    }];
		
		 //7.执行任务
		[dataTask resume];

### 三 、 NSURLSession代理方法

有的时候，我们可能需要监听网络请求的过程（如下载文件需监听文件下载进度），那么就需要用到代理方法。

接下来通过代码简单说明NSURLSession中普通网络请求会涉及代理方法的使用

	-(void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler{
	    completionHandler(NSURLSessionResponseAllow);
	}
	
	//2.接收到服务器返回数据的时候会调用该方法，如果数据较大那么该方法可能会调用多次
	 -(void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data
	{
	 NSLog(@"didReceiveData--%@",[NSThread currentThread]);
	
	    //拼接服务器返回的数据
	    [self.data appendData:data];
	 }
	
	 //3.当请求完成(成功|失败)的时候会调用该方法，如果请求失败，则error有值
	 -(void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
	{
	   NSLog(@"didCompleteWithError--%@",[NSThread currentThread]);
	    if(error == nil)
	    {
	        if (self.comBlock) {
	            self.comBlock(self.data);
	        }
	    }
	 }