---
layout: post
title:  "典型的 Serverless 架构是怎样的"
date:   2021-06-27 19:16:18 +0800
categories: serverless
tags:  ["serverless"]
author: zhaojizhuang
---


先看下 CNCF 2020 \(2021年的还没出\) Serverless 调查报告显示的数据，云计算厂商中的 Serverless 产品使用率中 AWS Lambda 占 57% ；开源 Serverless 平台平台使用率中 Knative 占 27% 。 

![](/images/serverless001.png)



![](/images/serverless002.png)





本文主要介绍 Lambda 上的 Serverless 典型架构

那么AWS Lambdas 吸引人的地方在于：自动扩缩容及按使用量计费。当然还有一个主要原因，AWS 提供的其他服务也是相当丰富（API gateway，鉴权，数据库，工作流等等）。给中小型企业充分发挥的余地，充分利用 AWS 提供的这些能力组建自己的 Serverless 形态产品架构。

本文以 Theodo 公司的一个 Web 项目为例，介绍典型的 Serverless 架构是怎样的，首先看下项目的全景图，看不懂没关系，接下来分解开来讲解。

在上述图表中， 方框代表的是存在于大多数 Serverless 架构中的典型技术领域或技术功能。


# 一、Serverless 实践思路

此项目的目标是拥有一个强大的完全托管的系统，并提供友好的开发人员体验。 为了实现这一目标，Theodo 公司采取了以下措施：

## 1. 选择 AWS

云技术的竞争很激烈：亚马逊云服务（AWS）、谷歌云服务（GCP）、微软云服务（Azure）、IBM 云服务、阿里云服务。这些平台都提供了自己的云计算产品，并且特性都在快速迭代中。 Ben Ellerby 在[这篇文章](https://medium.com/serverless-transformation/choosing-the-right-cloud-provider-a-serverless-cloud-atlas-eeae672076ce)\[1\]中比较了前三名的云服务提供商，而我们更青睐于 AWS 的解决方案。在 Serverless 架构方面，AWS 是最先进的。借助 AWS 的解决方案，我们可以尽可能接近 Serverless 架构。为了说明这一点，我们将在下文中详细介绍构成我们架构模块的每个 AWS 服务。

## 2. 使用 TypeScript 开发 Node.js 项目

JavaScript 是世界上最受欢迎的编程语言之一，社区也很活跃，根据 Datadog 的调查（可参考之前发的[一篇公众号文章](https://mp.weixin.qq.com/s/CnHAvL3hawi49_zoTNGCNg)），Serverless 架构中也是如此。尽管 Python 以 47％的占有率领先，但目前已部署的 Lambdas 中有 39% 是运行 JavaScript 的。 TypeScript 丰富了 JavaScript 的特性。最后说明下， 在绝大多数用例中， Lambdas 中的 JavaScript 都运行得很好。

## 3. Serverless Framework

Serverless Framework 完成了大部分基础架构即代码（Infrastructure as Code, IaC）的工作（基于 CloudFormation，CloudFormation 是 AWS 提供的一个基础设置编排工具）。定义一个对 HTTP 事件做出响应的 Lambda 函数， Serverless 架构框架将自动部署相关的 API Gateway资源、相应的路由以及新的 Lambda 函数。当需要更复杂的服务配置时，只需简单地添加一些 CloudFormation 即可 。

## 4. 细粒度的 Lambda 函数

Lambda 是一个函数，它有自己的工作任务，并且能做得很好。比如：

* 项目的前端需要获得一个项目列表，那这个功能可以新建一个 Lambda 函数。
* 当用户注册后，我们需要发送确认电子邮件， 为这个功能也可以新建一个 Lambda 函数。
* 当然，某些特定的代码（例如数据实体 Entity）可以分解成小的单元，并在专用的 utilities 文件夹中共享。但一定要非常小心这些代码，因为任何更改都会影响所有相关的 Lambda 函数。而且因为每个 Lambda 是可以独立测试和部署的，因此可能会遗漏一些内容（这时 TypeScript 就派上用场了）。

## 5. 分解成微服务

为了避免团队之间相互影响，同时避免 package.json 和 serverless.yaml 配置文件过大（CloudFormation 的资源限制数量为 200）以及过长的 CloudFormation 部署时间，同时也为了方便我们在代码库中定位，并在所有 Lambdas 函数之间明确清晰的团队职责：我们定义了微服务划分的边界。Ben Ellerby 在这篇文章中写了一个方法， [EventBridge Storming](https://aws.amazon.com/cn/eventbridge/)\[2\]，来帮助定义这些界限。

在我们的单体代码库中：一个微服务=一个 CloudFormation 堆栈=一个 serverless.yml + package.json。此外，微服务有只属于自己的数据实体，这些数据实体不会与其他微服务共享。

早期在项目中我们推荐只使用 JavaScript，但出于种种原因，可能想要使用另一种语言，或者可能希望逐步迁移到 JavaScript 中的 Serverless 架构。在 Serverless 架构中，微服务的优势是你可以在架构中混合多种技术栈，只要保证微服务之间的抽象接口一致即可。

## 6. 使用事件驱动

![](https://static001.infoq.cn/resource/image/11/74/118beff0bf65cffb0e39beb8673a8574.png)

同时微服务之间需要完全独立，如果其中一个微服务出现事故，或者正在对另一个微服务进行重大改动，这对于系统其他部分的影响应该越小越好。为实现这个目标，Lambdas 函数仅通过 EventBridge 这个 Serverless架构的事件总线来和其他 Lambdas 函数交互。在[这篇文章](https://medium.com/serverless-transformation/eventbridge-the-key-component-in-serverless-architectures-e7d4e60fca2d)\[3\]中， Ben Ellerby 详细叙述了为什么 EventBridge 用途这么大 。

# 二、详解 Serverless 架构模块

上面已经介绍了一些背景知识，接下来详细介绍下本文开篇的 Serverless 架构图中的每个模块。

## 1. 前端开发

![](https://static001.infoq.cn/resource/image/87/f0/87c6e57a8cec8ed42fa348a9982a3bf0.png)

我们优秀的无服务器后端需要以某种方式为前端提供数据。为简化与 AWS 耦合的前端开发，我们利用了 Amplify（AWS 提供的前端框架）。Amplify 囊括了几个不同的东西：一个命令行工具、一个基础架构即代码（IaC）工具、一个 SDK 和一套 UI 组件。我们利用前端的 JS 的 SDK 来加快与其他资源（比如用于认证的 Cognito）的集成，这些资源通常是通过其他 **基础架构即代码工具**（比如 Serverless Framework）来部署的。

## 2.  网站托管

![](https://static001.infoq.cn/resource/image/41/8b/416e9ca3e2ec398c60018a835eccec8b.png)

如今，大多数网站是单页应用（Single Page Application，SPA），它们是功能齐全的动态应用程序，被打包在一组静态文件中。这些文件是在用户的浏览器首次访问 URL 时下载的。在 AWS 环境中，我们在 S3（AWS 的 文件存储服务）中托管这些静态文件，并通过 CloudFront（AWS 的 CDN 服务）来公开。

虽说上面场景是多数，但前端的趋势仍然在朝着诸如 Next.js 之类的服务端渲染（Server Side Rendering，即 SSR）发展。要在 Serverless 架构中运行一个 SSR 网站，我们可以利用 CloudFront 中的 Lambda @ Edge。可以使用更接近用户端的 Lambdas 函数来进行服务端渲染。

## 3. 域名与证书

![](https://static001.infoq.cn/resource/image/76/65/76e884bbcb145788cb9eb818b5e31465.png)

在我们网站中，我们希望使用相比于原始自动生成的 S3 URL 更好的 URL，为了做到这一点，我们使用 Certificate Manager \(AWS 证书管理服务\)来生成证书，并将其绑定在 CloudFront（AWS 的 CDN 服务），并使用 Route 53 （AWS 的域名管理服务）来管理域名。

## 4. 业务逻辑接口

![](https://static001.infoq.cn/resource/image/80/24/806507cbc8d2eb3e9079ccb06cbdd124.png)

现在，我们的网站需要连接后端，以获得和推送数据。为此，我们使用 API Gateway来处理 HTTP 连接和路由，并为每个路由同步触发一个 Lambda 函数。我们的 Lambda 函数包含与 DynamoDB（AWS 的非关系数据库） 通信的业务逻辑，以便存储和使用数据。

架构如上文所示是事件驱动的，这意味着可以立即回复用户请求，同时继续在后台异步地处理请求。例如，DynamoDB （AWS 的非关系数据库）提供了流（Streams），它可以对任何数据改动作出反应，并且异步地触发 Lambda 函数。大多数 Serverless 架构的服务都有类似的功能。

## 5. 异步任务

![](https://static001.infoq.cn/resource/image/9y/31/9yye6507a45b9816a82b3ba8a2481d31.png)

我们项目的架构是事件驱动的，所以许多的 Lambda 函数都是异步的，通常是由 EventBridge 事件、S3 事件、DynamoDB 流等事件触发。例如，我们系统中有一个异步 Lambda 函数，负责在成功注册后发送欢迎电子邮件。

在分布式异步系统中，故障处理是非常重要的。所以对于异步的 Lambda 函数，我们使用它们的死信队列（Dead Letter Queue，DLQ），并且将最终的故障信息首先传递给 Simple Notification Service（SNS）（AWS 的 消息推送服务，如电子邮件，短信等），然后再传给 Simple Queue Service（SQS）（AWS 的消息队列服务）。我们现在必须这样做，因为 AWS 暂时还不支持将 SQS 直接连接到 Lambda DLQ 上。

## 6. 后端向前端的推送

![](https://static001.infoq.cn/resource/image/59/4e/59ce50yyb002a553cbfe5c67f0e2ef4e.png)

有了异步的操作，前端不能在等待一个 XHR （_xhr_：_XMLHttpRequest_）响应时仅仅显示一个加载页面。我们需要后端的预备状态和数据推送。为此，我们利用了 API Gateway 的 WebSocket，这个 API 可以使 WebSocket 保持连接状态，并且仅在在收到消息时触发 Lambdas 函数。 [这篇文章](https://medium.com/serverless-transformation/asynchronous-client-interaction-in-aws-serverless-polling-websocket-server-sent-events-or-acf10167cc67)\[4\]深入讨论了相比较于其他解决方案，为什么我们选择了 WebSocket，以及如何实现它。

## 7. 文件上传

![](https://static001.infoq.cn/resource/image/0e/0f/0ec6217d8895b03ab15c93682700650f.png)

处理 Lambda 的文件上传流（Stream）可能会造成较大的开销。相较于这个方案，S3 \(AWS 的文件存储服务\) 还提供了一个功能，使得前端能使用由 Lambda 生成的、签名（安全）的上传 URL 来直接上传文件到 S3 （AWS 的文件存储服务）。

## 8. 用户与认证

![](https://static001.infoq.cn/resource/image/83/d4/83878c3108041yy82431abf1d48878d4.png)

Cogito\( AWS 的认证服务\)有我们所需要的所有东西：认证、用户管理、访问控制以及外部身份提供商集成。尽管大家知道它使用起来有些复杂，但它确实可以为我们做不少事情。和其他服务一样，它由专用的 SDK 来与 Lambda 交互，并且可以通过分发事件来触发 Lambda。本例中，API Gateway 路由绑定了原生Cogito 授权服务。同时也暴露了一个用于刷新身份验证令牌的 Lambda 函数和一个用于获取用户列表的 Lambda 函数。

## 9. 状态机

![](https://static001.infoq.cn/resource/image/0a/4d/0a85712e398eb6c18398547fa24b3d4d.png)

在某些情形下，我们的逻辑和数据流可能会非常复杂。如果直接在 Lambda 函数内部手动维护和操作这些数据流，正在运行的系统可能会难以跟踪和掌控。因此，AWS 为我们提供了专门提供了一个服务：可视化工作流服务 （ [Step Functions](https://aws.amazon.com/cn/step-functions/)）。

我们通过 CloudFormation （AWS 的基础设施编排工具）来声明状态机：包括每个子步骤和状态、每个希望得到的结果和不希望得到的结果，并且将一些操作（例如等待、选择）或者一个 Lambda 函数挂载在这些步骤上。然后，就饿可以通过 AWS 界面实时看到这些服务的运行状态。在其中的每个步骤中，还可以定义重试和失败处理逻辑。Ben Ellerby 在 [这篇文章](https://medium.com/serverless-transformation/serverless-event-scheduling-using-aws-step-functions-b4f24997c8e2)\[5\]中进一步详细介绍了该服务。

这里举一个更加具体的例子，假设我们希望通过 SaaS 发送一个电子邮件广告，并且能够确保该广告已经发送完毕：

* 步骤 1，Lambda：要求 SaaS 发送电子邮件广告系列并获取广告的 ID。
* 步骤 2，任务令牌 Lambda：从 Step Function \(可视化工作流服务 \) 中获得回调令牌，将其链接到广告 ID，然后等待来自 SaaS 的回调。
* 步骤 3，（在任务流之外的）Lambda：在广告的状态发生改变时（待定、存档、失败、成功），从 SaaS 通过一个钩子函数调用，然后通过对应的回调令牌根据新的广告状态来继续任务流。
* 步骤 4，选择（Choice）：基于状态选择，如果这个广告还没有成功，回到第二步。
* 步骤 5，（结束）Lambda：在广告发送后，更新用户。

[这篇文章](https://medium.com/@zaccharles/9df97fe8973c)\[6\]深入讲述了任务令牌（Task Tokens ）是如何工作的。

## 10. 安全

![](https://static001.infoq.cn/resource/image/19/87/19de1ff7ee3af5e89ff32726bdcb6587.png)

Identity and Access Management \(IAM\) \(AWS 权限控制服务\)可以更加细粒度地管控任何 AWS 访问，不管是开发人员的访问、持续集成与持续交付流程 ， 还是 AWS 的服务调用另一个服务。IAM 的管控是非常精细的，需要平台开发者认真考虑某个特定的“消费者”被允许操作的所有行为。这意味着我们基础架构中的每一层都是受到保护的。

对于非常敏感的数据，比如 SaaS 的 API 密钥，我们将其安全地存储在 System Manager （AWS 的系统管理服务）中的 Parameter Store 中。无论是来自我们 Serverless Framework 还是 CloudFormation 文件，甚至是来自业务代码的访问，都必须通过对应的 SDK 来请求它们。值得一提的是， AWS Secrets Manager （AWS 的密钥管理软件\)也可以完成类似的工作。

如果对该话题感兴趣，推荐来自 Sat G 的 [这篇文章](https://medium.com/version-1/protecting-your-serverless-solution-cd5d4247e27c)\[6\]，关于 Serverless 中的安全问题在这篇文章中有更详细的讲解。

## 11. 监控

![](https://static001.infoq.cn/resource/image/45/5c/45484ce192570bffdc9c176ffe40b55c.png)

CloudWatch （AWS 的云监控服务）是监控服务的业界标准。所有 AWS 服务的基础指标和日志都可以发送到 CloudWatch。当然我们可以做更多的事情：将自定义的指标和日志发送到 CloudWatch，创建指标 DashBoard，在超过阈值后触发警报；数据分析与挖掘整理后展现在自定义的图表中。

可观测性方面还有其他选择，比如 X-Ray （AWS 分布式追踪服务），它的目标是在整个分布式系统中端到端地追踪请求，然后直观动态得展现出来。只不过现在这个追踪服务时不时会失败，因为它还不支持某些 AWS 服务，比如 EventBridge（而这在我们的架构中是重中之重）。

另一个服务，基于 X-Ray 和 CloudWatch 构建的 ServiceLens （AWS 的可视化监控服务），效果也很赞。

# 参考文章

1. `https://medium.com/serverless-transformation/choosing-the-right-cloud-provider-a-serverless-cloud-atlas-eeae672076ce`
2. `https://aws.amazon.com/cn/eventbridge/`
3. `https://medium.com/serverless-transformation/eventbridge-the-key-component-in-serverless-architectures-e7d4e60fca2d`
4. `https://medium.com/serverless-transformation/asynchronous-client-interaction-in-aws-serverless-polling-websocket-server-sent-events-or-acf10167cc67`
5. `https://medium.com/serverless-transformation/serverless-event-scheduling-using-aws-step-functions-b4f24997c8e2`
6. `https://medium.com/@zaccharles/9df97fe8973c`
