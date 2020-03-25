# 项目介绍
针对在线图片的裁剪需求非常的广泛，如您的应用包括App，Web，Pad等因为客户端的不同在现实图片信息的时候页需要相应的适配，从而针对同一个图片源需要支持在线裁剪来适配不同的客户端，此外可能还涉及包括压缩，锐化，或者图片的智能识别。
当前AWS Image Handler 解决方案,无法直接适配已有项目。本文将从AWS Image Handler 解决方案部署以及使用，代码分析，自定义代码等方面介绍AWS的图片处理解决方案。

# Image Handler方案存在的不足
如果客户是新开发的项目，遵循方案中的配置和URL访问方式是可以满足的，但是针对一些迁移项目如某云厂商提供的图片服务已经规定了访问路径和参数格式，这种场景下标准的解决方案将无法直接满足。本文将在“图片处理过程源代码介绍”章节介绍Image Handler 解决方案中的代码架构和实现原理，同时在“自定义代码适配图片处理路径” 章节已某云厂家的图片处理方式为示例介绍如何进行适配，已实现图片处理功能的平滑迁移。

# 解决方案介绍
![](https://github.com/VerRan/aws-image-handler/blob/master/architecture.png)
1. 客户通过客户端访问cloudfront endpoint，如果请求的内容在cloudfront cache中不存在的话，则将客户请求转发给Apigateway，如果存在的话则直接返回给客户端。
2. Apigateway接收到请求后，将客户请求直接转发给lambda
3. Lambda接到请求后，根据客户请求内容使用nodejs的图片处理库sharp进行图片处理，比如客户请求为裁剪图片，这lambda 会根据请求的参数进行裁剪并返回结果。
4. lambda 根据请求访问S3中的图片文件
5. lambda获取s3中的图片同时根据客户端的请求进行图片处理，返回处理后的图片给apigateway，然后通过cloudfront返回给客户端
6. 客户端收到处理后的图片信息并展示
该方案部署完成后包括以下可选组件：
1. *Demo UI 组件：*一个可选的组件，Demo UI部署在S3中，S3中的 js通过访问Apigateway访问图片处理方案，您可以通过UI上传图片，设置要处理的参数并提交，查看处理后的结果同时可以直接看到处理参数以及处理图片的endpoint
2. *智能裁剪：*一个可选组件可以借助AWS Rekogntion 实现自动的人脸识别，当您在UI中选择了智能裁剪，lambda 会调用 AWS Rekogntion 服务自动识别图片中的人脸信息。

# 部署方法

1. 点击cloudformation模版 (https://s3.amazonaws.com/solutions-reference/serverless-image-handler/latest/serverless-image-handler.template)自动下载模版文件
2. 登录AWS console ，选择Cloudformation
3. 在Cloudformation界面选择创建堆栈（with new Resource(stanard)），将上面的链接复制到 Amazon
![](https://github.com/VerRan/aws-image-handler/blob/master/image1.png)
4. 选择下一步，输入堆栈名称如“image-process-solution”，注意 Image Sources 参数需要设置您访问图片桶的名称，同时可以设置多个桶的名称，用逗号分隔（如桶名称为lht-ServerlessImageHandler），后续如果有变更可以通过lambda 环境变量修改。
![](https://github.com/VerRan/aws-image-handler/blob/master/image2.png)
5. 选择下一步，可以设置标签信息用于标记模版信息（可选）
6. 选择下一步，选择上本页最下方的复选框（允许cloudformation创建IAM资源并使用自定义名称），选择创建堆栈
7. 点击左侧菜单中的堆栈菜单，进入Cloudformation 堆栈页面，当堆栈的状态变更为“CREATE COMPLETE” 
![](https://github.com/VerRan/aws-image-handler/blob/master/image3.png)
8. 点击上图中的堆栈名称列（lht-ServerlessImageHandler）进入详情页面，选择“输出”tab页
![](https://github.com/VerRan/aws-image-handler/blob/master/image4.png)
9. 输出tab页面下的信息介绍如下：

* APIEndpoint: 用于处理图片的api接口，后面集成了lambda，DemoUI，Cloudfront就是通过调用该endpoint来实现图片处理功能的
* CorsEnabled ： 这个是在创建堆栈时指定的参数（当前为默认值），是否允许跨域访问的设置
* DemoUrl：这个是上文提到的可选组件DemoUI的访问链接
* LogRetentionPeriod：这个是在创建堆栈时指定的参数（当前为默认值），lambda上传到cloudwatch日志的保留时间
* SourceBuckets：访问图片的存储桶名称

10. 点击上图中DemoUrl的地址，效果如下图
![](https://github.com/VerRan/aws-image-handler/blob/master/image5.png)
* Image Source：第一个参数是桶名称 需要包含在 堆栈建立中的参数 SourceBuckets中存在，并且名称必须保持一致
* Image Source：第二个参数时要处理的图片，这个图片必须在上一步中桶中存在
* Editor：width，需要裁剪图片的目标宽度，Height：目标高度，Resize Mode：cover，覆盖也就是直接裁剪；contain，包含也就是图片包含在目标尺寸图中；fill，将裁剪后的图片填充为目标尺寸；fill color ：填充颜色；Backgound color：填充的北京颜色
* Smart Cropping：智能裁剪，自动识别人像，focus index：当图片包含多个人时可以指定裁剪出哪个人（通过索引指定）；Crop Padding，指定裁剪的填充大小
* Preview ： 展示裁剪后的效果图
* Request Body：前端UI发送给ApiGateway的参数
* Encoded URL：将请求参数进行了Base64编码后的请求链接
图片处理过程源代码介绍

源代码地址 (https://github.com/awslabs/serverless-image-handler)
*1.处理过程
   Index.html→script.js→apigatewat→lambda(index.js)→image-handler.js&image-request.js->sharp.js

* Index.html: DemoUI 的首页
* script.js :DemoUI 用于处理请求参数，包括封装参数针对参数做Base64编码，请求apiGateway等
*  APIGateway: 这里不涉及代码表示访问Apigateway服务
* Lambda(index.js): index.js 是部署在lambda中的图片处理入口类
* image-request.js：是访问请求处理类，包括Base64编码解析，处理类型，访问路径的解析，桶以及图片的解析，访问参数的解析等
* image-handler.js: 是请求的具体处理类通过解析后的参数然后调用sharp.js 进行图片的具体处理

*2.请求参数编码*
   前端代码，Script.js实现请求参数的编码以及后端Apigateway的调用
   const encRequest = btoa(strRequest);//请求编码为Base64 编码
*3.请求参数处理，如路径匹配*

*   image-requests.js 实现请求路径的解析
*   lambda 环境变量配置： RewriteMatchPattern 可以配置该参数，来适配当前的请求路径（REWRITE_MATCH_PATTERN）
* ImageHandler.js 通过sharp 对图片进行具体处理

*4.个性化定制方法*

* 配置RewriteMatchPattern 参数，将现有路径匹配出来，已适配已有的图片处理访问路径
* 如果通过RewriteMatchPattern参数无法满足需求时，可以通过修改源代码匹配当前环境

# 自定义代码适配图片处理路径

*比如您当前的请求路径如下：*

通过三级域名访问：http:// (http://channel/)*<endpoint>/object@action.format*
endpoint：访问的url端点
object：图片文件
action：图片处理操作
format：图片格式

## 需求分析：

由于访问路径与AWS解决方案默认提供的方式不一致，同时通过RewriteMatchPattern配置无法满足，考虑通过定制代码实现。

## 代码修改：

* 查找路径匹配入口
  上文提到代码的处理过程，首先需要通过index.js 入手找到图片路径匹配部分代码，可以找到这部分的处理是通过ImageRequest类的setup方法统一处理了，代码片段如下：
```sync setup(event) {
            this.requestType = this.parseRequestType(event);
            console.log('Parsed Request Type: ' + this.requestType);
            this.bucket = this.parseImageBucket(event, this.requestType);
            console.log('Parsed Image Bucket: ' + this.bucket);
            this.key = this.parseImageKey(event, this.requestType);
            console.log('Parsed Image Key: ' + this.key);
            this.edits = this.parseImageEdits(event, this.requestType);
            console.log('Parsed Image Edits: ' + JSON.stringify(this.edits));
            this.originalImage = await this.getOriginalImage(this.bucket, this.key);
```
* 路径匹配代码片段，通过分析代码发现此部分代码在t his.parseRequestType 方法中实现
```parseRequestType(event) {
        //自定义图片处理
          const matchMy = new RegExp(/(\/?)(.*)(jpg|png|webp|tiff|jpeg)@!(.*)/i);
        // ----
        if (matchDefault.test(path)) {  // use sharp
            return 'Default';
        } else if (matchCustom.test(path) && definedEnvironmentVariables) {  // use rewrite function then thumbor mappings
            return 'Custom';
        } else if (matchThumbor.test(path)) {  // use thumbor mappings
            return 'Thumbor';
        } else if (matchMy.test(path)) {  //自定义图片处理
            return 'Ali';
        } 
    }
```
* 桶解析部分适配，根据上一步自定义的请求类型进行适配处理，parseImageBucket 方法代码如下
```else if (requestType === "matchMy") {
            // Use the default image source bucket env var
            const sourceBuckets = this.getAllowedSourceBuckets();
            return sourceBuckets[0];
        }
```        
* 图片名称解析自定义处理代码 parseImageKey片段如下，
```//自定义图片处理
        if(requestType === "matchMy") {
            const path = event["path"];
            if(path.startsWith('/')) {
                return path.substring(1, path.indexOf('@!'));
            }
            return path.substring(0, path.indexOf('@!'));
        }
```
* ThuborMapping,processMatchMy 代码片段，方案默认提供的参数是json格式的同时使用了Base64编码，为了适配访问路径通过该类实现参数的解析，并将参数转换为标准的参数格式。
```
processMatchMy(event) {
        // Setup
        this.path = event.path;
        const ruleName = this.path.split('@!')[1];

        this.edits.resize = {};
        if (ruleName.match(/^w(\d+)$/)) { //w200
            this.edits.resize.width = Number(ruleName.match(/^w(\d+)$/)[1]);
            this.edits.resize.fit = 'inside';
        } else if (ruleName.match(/^h(\d+)\-w(\d+)$/)) { //h250-w250
            const dims = ruleName.match(/^h(\d+)\-w(\d+)$/);
            this.edits.resize.width = Number(dims[2]);
            this.edits.resize.height = Number(dims[1]);
            this.edits.resize.fit = 'contain';
            this.edits.resize.background = Color('#FFFFFF').object();
        } else if (ruleName.match(/^w(\d+)\-h(\d+)$/)) { //w90-h120
            const dims = ruleName.match(/^w(\d+)\-h(\d+)$/);
            this.edits.resize.width = Number(dims[1]);
            this.edits.resize.height = Number(dims[2]);
            this.edits.resize.fit = 'contain';
            this.edits.resize.background = Color('#FFFFFF').object();
        } else if (ruleName === 'banner_pc') {
            this.edits.resize.width = 1480;
            this.edits.resize.fit = 'inside';
        } else if (ruleName === 'banner_block_pc') {
            this.edits.resize.width = 650;
            this.edits.resize.fit = 'inside';
        } else if (ruleName === 'banner_block_m') {
            this.edits.resize.width = 200;
            this.edits.resize.fit = 'inside';
        } else if (ruleName === 'banner_m') {
            this.edits.resize.width = 600;
            this.edits.resize.fit = 'inside';
        } else if (ruleName === 'gif_t') {
            this.edits.resize.width = 200;
            this.edits.resize.fit = 'inside';
        }

        return this;
    }
```   
    
# 总结

通过本文针对AWS Image Handler解决方案的介绍，您可以获得如下收益：

1. 可以快速掌握如何搭建基于AWS的图片处理环境，同时针对方案中涉及的参数以及使用方法进行了介绍。
2. 可以快速了解图片处理方案的源代码结构以及基本原理
3. 可以通过自定义Lambda的方式快速适配当前已有的图片处理方案

# 参考

*解决方案地址：*
https://aws.amazon.com/solutions/serverless-image-handler/
*源代码地址：*
https://github.com/awslabs/serverless-image-handler
