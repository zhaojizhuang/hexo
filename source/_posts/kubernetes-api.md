---
title: kubernetes 常用 API
date: 2018-09-02 13:13:00
type: "kubernetes"

---

kubectl  的所有操作都是调用 kube-apisever 的 API 实现的，所以其子命令都有相应的 API，每次在调用 kubectl 时使用参数  -v=9  可以看调用的相关 API，例：
 `$ kubectl get node -v=9` 

以下为 kubernetes 开发中常用的 API：
![deployment 常用 API](https://upload-images.jianshu.io/upload_images/1262158-6fdb3babc9522929.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![statefulset 常用 API](https://upload-images.jianshu.io/upload_images/1262158-2562e12aaa019909.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![pod 常用 API](https://upload-images.jianshu.io/upload_images/1262158-d74728f38ba4361e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![service 常用 API](https://upload-images.jianshu.io/upload_images/1262158-225076f08495b6d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![endpoints 常用 API](https://upload-images.jianshu.io/upload_images/1262158-71c815ad4fc45a65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![namespace 常用 API](https://upload-images.jianshu.io/upload_images/1262158-f79e0c4bb215bb40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![node 常用 API](https://upload-images.jianshu.io/upload_images/1262158-e67546fabc697d13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![pv 常用 API](https://upload-images.jianshu.io/upload_images/1262158-a0eca87df2960565.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 Markdown 表格显示过大，此仅以图片格式展示。
