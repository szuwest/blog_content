Date: 2016-05-02
Title: iOS系统相册上传不得不说的那些事儿
Category: iOS
Tags: iOS网络编程 系统相册上传 导出视频 NSURLConnection NSURLSession 大文件上传 后台上传

最近在开发手机相册自动备份的功能，这就需要用到上传文件的功能。
其实我一年前也做过同样的功能，当时也做得不算很好。这次是别人做了，然后那人离职了。我来接受这块，然后发现有问题，然后准备认真的研究一下这个问题，顺便学习更多的iOS开发的知识。

## 系统相册概述

首先说一下iOS相册的问题。iOS的相片都是存放在系统库ALAssetLibrary中，开发者从这个库中读取到的是一个个ALAsset对象，而不是一个文件系统的File文件。当然iOS8系统新增了Photo Framework,但是只是增强了相关功能，还是不能直接取到File。 这个Android平台或者PC平台很不一样的。

我们知道一般的上传文件流程就是，将本地的文件转换为二进制数据或流，通过HTTP协议或者socket协议，本地和服务器之间建立一个连接，本地将流写入这个连接，服务器那边接收这个流，并将接收到的流写入文件，直接客户端那边将整个文件传完，传输就结束了。上传文件这个过程一般会被封装库，使用者只需传入必要的参数就可以了。我以前做自动备份相册的时候，就是使用了别人提供的静态库，它上传文件API就是需要传一个文件的本地路径。这就麻烦了，因为我们的应用从iOS系统库读取到的对象是ALAsset, 所以我需要先将ALAsset从系统中导出，写入我们应用的沙盒中，然后再把它在沙盒中路径传给上传库进行上传。

## 导出ALAsset

据我所知，iOS系统cocoa层网络传输基本上只有**NSURLConnection**和基于**NSURLSession**一套API。它们提供的接口基本上都是基于NSData，或者文件路径或者NSInputStream。网上有很多库或者框架都是基于这两套API来做上传，我们一般也使用第三方的框架来减少工作量。这里最有名的当然就是AFNetWorking。AFNetWorking库提供给使用的接口就是有NSData接口，文件路径接口，由于我们上传的照片或者视频可能很大，所以就不能直接使用NSData的接口，所以必须使用文件接口。那么就得考虑如何将ALAsset导出为一个文件了。

将ALAsset从系统中导出的方法如下：

````
- (BOOL)exportAsset:(ALAsset*)asset toFilePath:(NSString *)path {
    [[NSFileManager defaultManager] createFileAtPath:path contents:nil attributes:nil];
    NSFileHandle *handle = [NSFileHandle fileHandleForWritingToURL:fileUrl error:error];
    if (!handle) {
        return NO;
    }
    
    ALAssetRepresentation *rep = [asset defaultRepresentation];
    uint8_t *buffer = calloc(BufferSize, sizeof(*buffer));
    NSUInteger offset = 0, bytesRead = 0;
    
    do {
        @try {
            bytesRead = [rep getBytes:buffer fromOffset:offset length:BufferSize error:error];
            [handle writeData:[NSData dataWithBytesNoCopy:buffer length:bytesRead freeWhenDone:NO]];
            offset += bytesRead;
        } @catch (NSException *exception) {
            free(buffer);
            return NO;
        }
    } while (bytesRead > 0);
    
    free(buffer);
    return YES;
 }
 
````
这个方法有个地方是特别要说明的，就是将ALAsset从系统库中读出来不是一下子读到内存中的，而是每次读一个buffer大小的数据，然后将这个buffer中的数据写入文件中，不断读取和写入，直到读完。这主要是避免一个文件（如视频）一下子读到内存中导致内存问题。
相册里大部分都是图片，而图片大部分也就是几MB, 一般就算一次性直接读取二进制数据到内存，不拷贝文件出来，也不会产生内存问题。可是视频就不行了，因为一般视频都比较大，几百MB甚至几个GB。所以上面这个导出方法十分重要。

可是这个方法也有一个致命的问题，就是必须要有足够的剩余空间给你拷贝一份相片或者视频出来。特别是视频。空间问题是个大问题，因为正是因为用户的手机剩余空间不足了，才想要备份，而备份又需要额外的拷贝空间。设想一个用户拍了一个3GB的视频，他要备份这个视频，至少要预留3GB以上的剩余空间才能备份，这显然很不合理。那有没有别的不导出ALAsset直接上传的方法呢。我也一直在思考这个问题。我能想到的是：一种是边读取边上传，也就是通过ALAssetRepresentation的"getBytes:fromOffset:error"方法读取出来就上传。这样的话需要自己先建立连接，传输HTTPheader相关准备数据（boundary），然后就传输从ALAssetRepresentation读取的数据，读完之后再自己做一些结尾的工作。这一系列的操作都是自己来实现。貌似这样的工作量也挺大的，而且最重要的是我没想好这一整体过程实现起来是怎样的。还有一种就以chuck方式上传，即一段一段的上传给服务器，服务器那边收到之后自己组装成一个完成的文件。这样客户端这边传输可以利用已有的API，但是服务器那边需要额外的支持。我们的服务器这边比较弱，不能支持这种方式。

所以我最终还是选择了导出ALAsset的方式，开发相对来说容易一些。不过这种方式被其他人吐槽和耻笑。。

## 上传的坑
我们项目里用的是**AFNetWorking**库，所以我用它来上传。一开始我们用的是2.0版本，它有基于NSURLConnection的API，也有基于NSURLSession一套API。默认都是用NSURLConnection的API。但是iOS9不是不再推荐使用NSURLConnection了吗，所以AFN3.0版本直接就把NSURLConnection的相关API去掉了，全部都使用基于NSURLSession的API。本来我们用2.0版本好好的，但是更新到3.0版本后就出问题了。在iOS7.0系统上上传失败。经过和服务器调试，发现客户端的网络请求header里面没有contentLength，所以服务器那边失败了。就算你自己给它设置这个contentLength，它还是会丢弃掉，因为这是苹果API底层做的事！这实在是太蛋疼了，想改还改不了。所以我只能寻求别的方法。最终我用了ASIHTTPRequest库，为什么我没有采用旧的NSURLConnectionAPI呢。因为这个API也有它的缺陷，否则苹果也不会再最新的系统把它废弃掉。我自己在项目中遇到的问题是它改变不了HTTP header中connection连接方式，默认的是keep-alive方式，但是我们有一种情况是需要服务器中转HTTP请求，如果是keep-alive方式的话很容易出现问题，所以必须是设为close方式。蛋疼的问题来了，你改变不了！你在HTTP request头中设置了，到底层还是被系统覆盖掉，苹果实在是太霸道了。。

**ASIHTTPRequest**库是基于CFNetwork框架实现的，它很底层，你可以控制各种东西，所以我就采用了它，而且它也挺高效。唯一不足的就是这个库已经早就不维护了。只能自己维护。所以我最终的方式是，如果是iOS8以下系统，就采用ASI方式上传，否则就采用AFNetWorking方式上传。这里顺表ASI的另一个优点，它可以做到真正的断点续传下载，iOS系统提供NSURLSessionTask的断掉续传不是真的断掉续传，因为强退应用之后NSURLSessionTask会重新从头开始下载，它只能做到应用生命周期内的断掉续传。而ASI是真个下载过程你可以干涉，并且它本来已经把你保存好文件并且强退后重启也可以还原。虽然这个原理一点都不复杂，但是不知道为啥NSURLSessionTask就是不这样做，估计是外国人的使用习惯和思维和我们不同，人家不屑于实现这个？

## 后台上传
由于系统相册里有可能很多图片，上传又是一个比较慢的过程，所以很自然就想到要做后台上传。要做后台任务，iOS7以后当然是用基于NSURLSession的NSURLSessionTask，这也是苹果一直力推使用NSURLSession API的原因吧。可是NSURLSession相关API的坑也很多。
一般来说，只要生成一个具有后台任务配置的NSURLSession，然后由它来创建NSURLSessionUploadTask，然后基于这个task来上传就可以做到后台也能上传。实际的代码运行中崩溃了。
在网上搜了很多资料，最终发现说iOS的后台上传任务不支持NSData方式上传，它只支持file的方式上传。我们是基于AFN的mutiPart方式来上传，即你设置好相关的信息和文件地址，然后AFN会转换为输入流来上传，但是这种方式是后台上传不了的。

既然后台只能支持文件方式上传，那就只能将所有的信息写入一个文件，最后将这个文件来上传，AFN确实也提供了这样一种方法。

支持后台上传的方法如下：

````
- (void)uploadFile3:(NSString *)filePath withURL:(NSString *)urlString withSaveName:(NSString *)saveName andFileItem:(FileItem *)fileItem {
    
    NSString* finalFile = [NSTemporaryDirectory() stringByAppendingPathComponent:[fileItem getFileName]];
    NSString *fileType = [fileItem contentType];
    
    // Prepare a temporary file to store the multipart request prior to sending it to the server due to an alleged
    // bug in NSURLSessionTask.
    // Create a multipart form request.
    AFHTTPRequestSerializer *requestSerizlizer = [AFHTTPRequestSerializer serializer];
    NSMutableURLRequest *multipartRequest = [requestSerizlizer multipartFormRequestWithMethod:@"POST"
                                                                                    URLString:urlString
                                                                                   parameters:nil
                                                                    constructingBodyWithBlock:^(id<AFMultipartFormData> formData)
                                             {
                                                 [formData appendPartWithFileURL:[NSURL fileURLWithPath:filePath] name:saveName fileName:saveName mimeType:fileType error:nil];
                                             } error:nil];
    multipartRequest.timeoutInterval = 60*30;
    // Dump multipart request into the temporary file.
    [requestSerizlizer requestWithMultipartFormRequest:multipartRequest
                                              writingStreamContentsToFile:[NSURL fileURLWithPath:finalFile]
                                                        completionHandler:^(NSError *error) {
                                                            // Once the multipart form is serialized into a temporary file, we can initialize
                                                            // the actual HTTP request using session manager.
                                                            _currentTask = [_manager uploadTaskWithRequest:multipartRequest fromFile:[NSURL fileURLWithPath:finalFile] progress:^(NSProgress * _Nonnull uploadProgress) {
                                                                NSLog(@"progress = %f", uploadProgress.fractionCompleted);
                                                                self.fileItem.uploadedSize = fileItem.fileSize*uploadProgress.fractionCompleted;
                                                                [self notifyStateChanged];
                                                            } completionHandler:^(NSURLResponse * _Nonnull response, id  _Nullable responseObject, NSError * _Nullable error) {
                                                                [[NSFileManager defaultManager] removeItemAtURL:[NSURL fileURLWithPath:finalFile] error:nil];
                                                                if (error) {
                                                                    NSLog(@"Upload image failure......\n%@",[error description]);
//                                                                    [[NSFileManager defaultManager] removeItemAtPath:filePath error:nil];//上传失败保留文件下次上传
                                                                    [self onUploadFailure:fileItem];
                                                                } else {
                                                                    NSLog(@"Upload image Success......\n%@",[[_currentTask response] description]);
                                                                    NSLog(@"Response Data:\n%@",[NSString stringWithData:responseObject encoding:@"UTF-8"]);//返回是空
                                                                    [[NSFileManager defaultManager] removeItemAtPath:filePath error:nil];
                                                                    Partition *partition = [self backupPartition];
                                                                    partition.usedSize = [NSNumber numberWithLongLong:(partition.usedSize.longLongValue + [fileItem getAsset].defaultRepresentation.size)];
                                                                    [self onUploadSuccess:fileItem];
                                                                }
                                                            }];
                                                            [_currentTask resume];

                                                        }];
}
````

这个方式在iOS7及以上的系统都有效，但是在iOS7系统还是有点小问题：上传进度不回调。这。。这。。我无话可说了，所以我项目中iOS7还是只能用ASI来上传。

后台上传可以了，但是实际测试中貌似上传效率也不高，最主要还是要将所有数据再写入一个文件，这个是硬伤。因为前面导出ASAsset的时候已经临时生成了一个文件，在AFN后台上传的时候再一次写多一个临时文件，这样一次上传就生成了两个临时文件，如果这个文件是个很大的视频，那必然很容易导致磁盘空间不足而失败。

所以，后台上传看上去很美好的事情，实际过程中却是那么残酷的过程。

**参考**

[AFN issue]( https://github.com/AFNetworking/AFNetworking/issues/1398)

[StackOverflow相关问题](http://stackoverflow.com/questions/27154006/how-to-upload-task-in-background-using-afnetworking)