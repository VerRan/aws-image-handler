# 如何快速将您的阿里图片处理功能迁移到AWS

# 引子

最近有不少客户问到如何在AWS上实现在线图片处理（如裁剪，锐化，压缩）？以及如何在不修改原有代码基础上构建高可用，弹性的图片处理服务呢？
通过本文的介绍，您可以得到如下收益：

* 可以快速掌握如何搭建基于AWS的图片处理环境，快速掌握相关组件和参数。[查看【部署方法&参数详解】章节](#deploy)
* 可以快速了解图片处理方案的源代码结构以及基本原理。[查看【源代码解析】章节](#source)
* 可以通过自定义Lambda的方式快速适配当前已有的图片处理方案，零代码改造集成AWS图片处理方案（当前仓库的代码已支持Ali云的图片服务格式）。[查看【如何适配已有应用】章节](#adapter)
* 如果您擅长使用java可参考：https://github.com/VerRan/image-serverless-java 是一个java版处理的demo当前只实现了图片裁剪功能

# 背景

针对在线图片的裁剪需求非常的广泛，如您的应用包括App，Web，Pad等因为客户端的不同在现实图片信息的时候页需要相应的适配，从而针对同一个图片源需要支持在线裁剪来适配不同的客户端，此外可能还涉及包括压缩，锐化，或者图片的智能识别， 下面介绍一下AWS Serverless Image Handler 解决方案，以及如何定制该方案已符合您的需求。

# AWS Serverless-image-handler方案存在的不足

如果客户是新开发的项目，遵循方案中的配置和URL访问方式是可以满足的，但是针对一些迁移项目如某云厂商提供的图片服务已经规定了访问路径和参数格式，这种场景下标准的解决方案将无法直接满足。后面章节将从 AWS ServerlessImage Handler 解决方案中的代码架构和实现原理，同时会已某云厂家的图片处理方式为示例介绍如何进行适配，已实现图片处理功能的平滑迁移


# Serverless-image-handler方案与Lambda Edge方案的选择
* 如果当前您已经使用了cloudfront或者计划同时使用clodfront，那么可以直接采用将lambda与Cloudfront结合来实现图片的裁剪。
* 如果当前您使用的是其他CDN，那么可以考虑使用Serverless-image-handler方案，与现有CDN集成只需要将Apigateway的域名配置与CDN域名关联。
* 如果您希望将图片处理能力作为api服务提供给不同的应用使用时，那么可以考虑使用Serverless-image-handler方案。

# AWS Serverless Image Handler 解决方案介绍
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

# <span id="deploy">部署方法&参数详解</span>

1. 点击[cloudformation模版](https://s3.amazonaws.com/solutions-reference/serverless-image-handler/latest/serverless-image-handler.template)
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

# <span id="source">源代码解析</span>
## 处理过程
  ![](https://github.com/VerRan/aws-image-handler/blob/master/image6.png)
* Index.html: DemoUI 的首页
* script.js :DemoUI 用于处理请求参数，包括封装参数针对参数做Base64编码，请求apiGateway等
*  APIGateway: 这里不涉及代码表示访问Apigateway服务
* Lambda(index.js): index.js 是部署在lambda中的图片处理入口类
* image-request.js：是访问请求处理类，包括Base64编码解析，处理类型，访问路径的解析，桶以及图片的解析，访问参数的解析等
* image-handler.js: 是请求的具体处理类通过解析后的参数然后调用sharp.js 进行图片的具体处理
* sharp.js: 一款高性能的图片处理库
## 请求参数编码
   前端代码，Script.js实现请求参数的编码以及后端Apigateway的调用
   const encRequest = btoa(strRequest);//请求编码为Base64 编码
## 请求参数处理，如路径匹配
*   image-requests.js 实现请求路径的解析
*   lambda 环境变量配置： RewriteMatchPattern 可以配置该参数，来适配当前的请求路径（REWRITE_MATCH_PATTERN）
*   ImageHandler.js 调用sharp.js对图片进行具体处理

## 个性化定制方法
* 配置RewriteMatchPattern 参数，将现有路径匹配出来，已适配已有的图片处理访问路径
* 如果通过RewriteMatchPattern参数无法满足需求时，可以通过修改源代码匹配当前环境

# <span id="adapter">如何适配您的已有应用</span>
## 适配路径
下面是aLI云上的图片处理路径格式，下面我们已此为例介绍如何进行定制代码已适配您当前应用。
* 访问路径格式：http://<endpoint>/object@action.format
* endpoint：访问的url端点
* object：图片文件
* action：图片处理操作
* format：图片格式
## 访问路径举例：
  https://www.example.com/xxx/xx/123/abc.jpg@!w420-h560 此路径表示将abc.jpg文件裁剪为宽420*高560的图片然后返回
  
## 需求分析：
由于访问路径与AWS解决方案默认提供的方式不一致，同时通过RewriteMatchPattern配置无法满足，考虑通过定制代码实现。
，下面将已如上路径为例进行代码说明，同时我已提交了此部分代码适配了其它格式后面在代码中介绍。
## 代码修改：
* 查找路径匹配入口
  上文提到代码的处理过程，首先需要通过index.js 入手找到图片路径匹配部分代码，可以找到这部分的处理是通过ImageRequest类的setup方法统一处理了，代码片段如下：
```sync setup(event) {
        try {
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
* 路径匹配代码片段，通过分析代码发现此部分代码在this.parseRequestType 方法中实现
```parseRequestType(event) {
        //自定义图片处理
          const matchAli = new RegExp(/(\/?)(.*)(jpg|png|webp|tiff|jpeg)@!(.*)/i);
        // ----
        if (matchDefault.test(path)) {  // use sharp
            return 'Default';
        } else if (matchCustom.test(path) && definedEnvironmentVariables) {  // use rewrite function then thumbor mappings
            return 'Custom';
        } else if (matchThumbor.test(path)) {  // use thumbor mappings
            return 'Thumbor';
        } else if (matchAli.test(path)) {  //自定义图片处理
            return 'Ali';
        } 
    }
```
   我们增加自定义图片处理部分代码已增加Ali 适配类型，matchAli 定义了图片访问路径的正则表达式
* 桶解析部分适配，根据上一步自定义的请求类型进行适配处理，parseImageBucket 方法代码如下
```else if (requestType === "matchAli") {
            // Use the default image source bucket env var
            const sourceBuckets = this.getAllowedSourceBuckets();
            return sourceBuckets[0];
        } 
 ```
* 图片名称解析自定义处理代码 parseImageKey片段如下，
```//自定义图片处理
        if(requestType === "matchAli") {
            const path = event["path"];
            if(path.startsWith('/')) {
                return path.substring(1, path.indexOf('@!'));
            }
            return path.substring(0, path.indexOf('@!'));
        }
```
  需要根据您自己的访问路径进行适配，当前代码支持的是abc.jpg@!w420-h560路径
* 图片处理请求参数适配处理，代码片段如下：
```parseImageEdits(event, requestType) {
        if (requestType === "Default") {
            const decoded = this.decodeRequest(event);
            return decoded.edits;
        } else if (requestType === "Thumbor") {
            const thumborMapping = new ThumborMapping();
            thumborMapping.process(event);
            return thumborMapping.edits;
        } else if (requestType === "Custom") {
            const thumborMapping = new ThumborMapping();
            const parsedPath = thumborMapping.parseCustomPath(event.path);
            thumborMapping.process(parsedPath);
            return thumborMapping.edits;
        } else if (requestType === "matchAli") { //自定义图片处理
            const thumborMapping = new ThumborMapping();
            thumborMapping.processMatchMy(event);
            return thumborMapping.edits;
        }
```
  自定义ThumborMapping类已实现自定义处理逻辑
* ThuborMapping,processMatchAli代码片段，方案默认提供的参数是json格式的同时使用了Base64编码，为了适配访问路径通过该类实现参数的解析，并将参数转换为标准的参数格式。
```processMatchAli(event) {
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
通过儒商代码匹配您当前不同的图片处理规则，这里只需要将当前的规则，同时参数设置到edits参数中即可。当前代码会自动根据参数去通过sharp进行处理。

# 新版本部署
* 方法1
1. 修改代码后，代码打包针对node.js 使用 npm install ,然后 zip 整个目录为xx.zip 文件（也可以使用方法2中的打包脚本进行打包）
2. 通过部署章节已经部署好的环境，在console中选择lambda 服务，找到类似xx-ServerlessImageHandler-ImageHandlerFunction的函数，然后将打包好的代码上传到lambda
3. lambda console 保存就可以验证了。
4. 日志查看，通过lambda 监控页面转到 cloudwatch log查看日志

* 方法2
1. git clone https://github.com/awslabs/serverless-image-handler.git 克隆方案原始代码
2. 根据上文中描述的方法，已适配当前逻辑默认已提供支持阿里图片处理服务的路径，可以直接覆盖clone后的代码
3. 参考 https://github.com/awslabs/serverless-image-handler/blob/master/README.md 的方法进行代码打包和部署

* java版本
针对java版本代码可以使用SAM进行本地测试和打包部署
具体参考：https://github.com/VerRan/image-serverless-java 

# 参考

* 解决方案地址：
https://aws.amazon.com/solutions/serverless-image-handler/
* 源代码地址：
https://github.com/awslabs/serverless-image-handler
