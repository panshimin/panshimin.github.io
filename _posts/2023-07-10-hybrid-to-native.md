---
layout:     post
title:      知乎详情页 Hybrid 转 Native
subtitle:  
date:       2023-07-10
author:     潘世民
header-img: img/2023-07-10/2023-07-10-bg.jpeg
catalog: true
tags:
    - 知乎
    - 短容器
    - Hybrid
    - Native
---

### 背景
在短容器项目上线之后，我们发现列表上卡片内容展示的行数和业务收益正相关；如果把详情页的内容完全在卡片上展示出来，那么收益岂不更高 😁；但是目前我们的卡片不支持展示出了纯文本之外的更多类型的内容；
##### 详情页文本结构现状
正文： HTML 富文本；   
普通摘要：纯文本；   
短容器列表卡片：最多 700 字纯文本；   
下面我简单介绍下短容器的卡片的在站内的现状：   
1、如果这个内容是纯文本且这个内容行数不超过下发的最大行数(max_line)，那么这个内容就可以在卡片上完全展示，可以看一下这个 case，在卡片上把一个比较短的回答在卡片上完整展示了。
![不折叠](/img/2023-07-10/mix_card_long_text.jpg)
2、如果这个内容是纯文本但是内容的长度超过了 max_line，那么只展示前 max_line 行，在文本的下面展示了一个 `查看详情`按钮，点击这个按钮，进入`Hybrid`详情页；
![折叠](/img/2023-07-10/mix_card_pic_fold.jpg)
3、如果这个内容包含图片、视频等内容，先展示一部分文字内容，然后再使用一个视图展示这个内容中的图片的缩略图，如下图所示
![折叠图片](/img/2023-07-10/2023-07-10-native-pic.jpg)

在知乎的监控系统上，知乎站内 ANR 每天差不多发生 3000+；经过分析绝大部分都是以渲染长文引起的，这个急需治理，而在当前 HTML 文本下基本无解；   
同时我们也分析了前端和客户端在`编辑`、`渲染`、`理解`、`互动`这几个维度的对比
![html_富文本_badcase](/img/2023-07-10/html_bad_case.png)
综合考虑了产品需求和前端渲染存在的问题，我们决定尝试把 hybrid 详情页使用 native 技术来渲染；

### 调研
##### PDD
听说 PDD 的 APP 之所以比其他购物 APP 流畅就是因为 PDD 的详情页是使用 Native 实现的；
我们可以使用 `Lookin` 来一探究竟；   
![pdd_lookin](/img/2023-07-10/pdd_lookin.jpg)
从上面 PDD 商品详情页的页面布局来看，详情页确实是使用 native 来实现的，并没有使用 webView；   从页面结构可以看出 PDD 是把详情页拆分成`商品大图`、`价格`、`促销信息`、`先用后付`、`留白`、`标题标签`等节点，每一种节点使用对应类型的 cell 来渲染；这样就可以实现详情页完全由 Native 渲染了；
##### 今日头条
使用 `Lookin` 分析发现头条的新闻详情页的正文部分还是使用 webView 渲染的；但是头条的 UGC 详情页的正文内容是使用 Native 来实现的
![头条_UGC](/img/2023-07-10/toutiao_ugc_page.jpg)
从上面的页面布局信息可以看出，头条的 UGC 正文内容是直接使用 `UITextView` 来实现的；

由于知乎的回答、文章的详情页是知乎核心的消费场景，知乎各个部门都有业务在详情页中流通；如商业部门的各种类型的广告卡片、好物卡片、芝士卡片等等；会员部门的促销卡片、会员购买卡片、盐选卡片等等，这些第三方卡片应该由他们自己解析和渲染，我们的详情页应该只提供槽位；同时也考虑到知乎有些文章、回答的内容比较长，直接使用一个 `UITextView` 渲染可能存在性能问题；我们决定采取 PDD 的这种方案来对 hybrid 详情页 native 化；

### 结构化协议
#### 节点定义
##### 块节点
HTML 富文本、知乎富文本编辑器内的元素都为 `块 节点`，块节点在富文本及编辑器内占据整行，呈纵向排列；  
块节点有 2 种形式，文本块节点和原子节点：   
1、文本块节点内无法嵌套任意其他块节点，可以放富文本，通过 `标记` 控制文本样式的展现；   
2、原子节点内无法放置任意节点，但会提供该节点所需要的数据，原子节点的 UI 完全由业务方自己渲染；   
知乎对于块节点这样设计是基于下面的考虑：
1、尽可能保证扁平化的富文本结构层级，避免层级嵌套过深；
2、经过分析，目前知乎站内创作者对于富文本主要的使用场景比较简单；
所以对于`块节点`的添加了约束：   
1、不可嵌套 -> 块节点无法嵌套任意块节点；   
2、原子化 -> 节点的 UI 结构比较复杂时，支持业务自定义这个节点的渲染；同时有利于第三方业务自定义节点的接入和渲染；
###### 块节点的通用属性
![块属性](/img/2023-07-10/block_node_define.jpg)
##### 块节点的特定属性
###### 文本块节点
文本块节点，即 `type` 是 `paragraph` 的节点
详细定义
![文本节点](/img/2023-07-10/paragraph.jpg)
###### JSON 结构示例
``` 
{
    "type": "paragraph",
    "id": "7af38973-3787-41b3-bd75-0ed3a1edfac9",
    "paragraph": {
        "text": "港澳通行证+有效签注：过关就靠这个啦！",
        /// 对 "港澳通行证+有效签注：" 加粗
        "marks": [
            {
                "end_index": 10,
                "start_index": 0,
                "type": "bold"
            }
        ]
    }
}
```

###### 标题节点
heading 标题节点，等同与 paragraph 节点，只是样式上的扩展
详细定义
![heading_node_define](/img/2023-07-10/heading_node_define.jpg)
###### JSON 结构示例
```
{
    "type": "heading",
    "id": "7af38973-3787-41b3-bd75-0ed3a1edfafd",
    "heading": {
        "text": "Octopus 八达通",
        "level": 2,
        "marks": [
            {
                "end_index": 11,
                "start_index": 0,
                "type": "bold"
            },
            {
                "end_index": 11,
                "start_index": 0,
                "type": "italic"
            }
        ]
    }
}
```
其他类型的节点 `paragraph` 和 `heading` 节点的定义和结构类似就不再一一列举了；
#### 原子块节点
原子节点可以理解为一个插槽，它只提供一个视图的占位及内部需要的数据；节点的 UI 渲染可以由我们根据节点的类型自定义；原子节点内无法实现富文本样式；
image 节点就是一个原子节点
image 节点的详细定义 
![图片节点](/img/2023-07-10/image_node_define.jpg)
###### JSON 结构
``` 
{
    "type": "image",
    "id": "7",
    "image": {
        "description": "",
        "height": 1066,
        "is_gif": false,
        "layout": "normal",
        "original_token": "v2-211236bc5a4a44fd7ca44ac28959b370",
        "original_urls": [
            "https://picx.zhimg.com/v2-211236bc5a4a44fd7ca44ac28959b370_720w.jpg?source=7e7ef6e2"
        ],
        "status": "normal",
        "token": "v2-211236bc5a4a44fd7ca44ac28959b370",
        "urls": [
            "https://pic1.zhimg.com/v2-211236bc5a4a44fd7ca44ac28959b370_b.jpg"
        ],
        "width": 1600
    }
}
```
card 节点，第三方的节点绝大部分都是通过 card 节点承接的；该节点作为卡片实体的容器，包括基本的数据及透传给业务方的数据，这种样式的卡片包括：
- 链接卡片   
- 知+ 卡片
- 商业卡片
- 教育卡片
- ...   
   
详细定义
![card_node_define](/img/2023-07-10/card_node_define.jpg)
###### JSON 结构
``` 
{
    "id": "10e7d5c0843db50441f0a6ce1a6eb38f",
    "type": "card",
    "card": {
        "biz_url": "https://xg.zhihu.com/plugin/10e7d5c0843db50441f0a6ce1a6eb38f?BIZ=ECOMMERCE",
        "card_type": "ad-link-card",
        "content_type": "ARTICLE",
        "cover": "https://pic3.zhimg.com/v2-9b4d085c47608d3b5b84f6f179524c2d.png?source=d6434cab",
        "extra_info": "{\"setAdjson\":true,\"setProductName\":true,\"setAppid\":false,\"productName\":\"重庆\",\"price\":0,\"externalInformation\":\"13022323120\",\"setCommercialType\":false,\"id\":990900,\"setChannelSource\":true,\"is_canvas\":false,\"setIs_canvas\":true,\"setPluginName\":false,\"setWechat\":true,\"setConversion\":true,\"setQrCode\":false,\"cardId\":\"Plugin_10e7d5c0843db50441f0a6ce1a6eb38f\",\"conversionIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=conversion\u0026ci=__CREATIVEID__\u0026pli=990900\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"setPackageName\":true,\"setAppInfo\":false,\"setDescription\":true,\"setEcomSource\":false,\"androidScore\":0.0,\"si\":\"08b72c11-b399-447b-8d6a-e2378c742e46\",\"web_experimentSize\":0,\"setPluginUrl\":true,\"setExInfo\":false,\"setLogo\":true,\"pluginToken\":\"10e7d5c0843db50441f0a6ce1a6eb38f\",\"setImplantType\":false,\"setPluginStyle\":true,\"url\":\"https://www.zhihu.com/people/lang-ye-shu-bang?WECHAT_ID=13022323120\",\"setExternalInformation\":true,\"setInteractionLaunch\":true,\"setIsPlugin\":false,\"setDeeplinkUrl\":true,\"setAdTag\":false,\"setCardId\":true,\"setVerifyType\":false,\"setTemplate_id\":false,\"isPlugin\":false,\"commercialType\":0,\"setAndroidScore\":true,\"setPrice\":true,\"picUrl\":\"https://pic3.zhimg.com/v2-9b4d085c47608d3b5b84f6f179524c2d.png?source=d6434cab\",\"validStatus\":0,\"logo\":\"https://pic2.zhimg.com/v2-9b4d085c47608d3b5b84f6f179524c2d.png?source=d6434cab\",\"setId\":true,\"setHashId\":false,\"setValidStatus\":false,\"trackUrl\":{\"viewSize\":1,\"setImgView\":false,\"conversionIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=conversion\u0026ci=__CREATIVEID__\u0026pli=990900\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"conversionSize\":1,\"impressionIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=impression\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"click\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=click\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"view\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=view\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"clickIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=click\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"imgViewSize\":0,\"clickSize\":1,\"setConversion\":true,\"setView\":true,\"impressionSize\":1,\"setClick\":true,\"impression\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=impression\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"imgClickSize\":0,\"setImgClick\":false,\"viewIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=view\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"setImpression\":true,\"conversion\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=conversion\u0026ci=__CREATIVEID__\u0026pli=990900\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"]},\"channelSource\":\"\",\"setShareDesc\":false,\"iosScore\":0.0,\"setShareUrl\":false,\"setUrl\":true,\"setAndroidStoreUrl\":false,\"adjson\":\"{\\\"ads\\\":[{\\\"view_x_tracks\\\":[],\\\"conversion_track_js\\\":[],\\\"debug_tracks\\\":[],\\\"click_tracks\\\":[],\\\"creatives\\\":[{\\\"asset\\\":{\\\"landing_url\\\":\\\"https://www.zhihu.com/people/lang-ye-shu-bang?WECHAT_ID=13022323120\\\"}}],\\\"impression_tracks\\\":[],\\\"view_tracks\\\":[],\\\"close_tracks\\\":[],\\\"conversion_tracks\\\":[]}]}\",\"setPhoneNumber\":false,\"setIosScore\":true,\"setIosAppStoreUrl\":false,\"deeplinkUrl\":\"\",\"description\":\"一键复制加玥玥微信号【13022323120】免费领取重庆旅游攻略\u003e\u003e\u003e\",\"conversionSize\":1,\"setLabelName\":false,\"setPluginToken\":true,\"pluginUrl\":\"https://xg.zhihu.com/plugin/10e7d5c0843db50441f0a6ce1a6eb38f?BIZ=ECOMMERCE\",\"packageName\":\"com.tencent.mm\",\"setBigImg\":false,\"setSi\":true,\"conversion\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=conversion\u0026ci=__CREATIVEID__\u0026pli=990900\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"implantType\":0,\"interactionLaunch\":\"去复制\",\"setScene\":false,\"setPicUrl\":true,\"cardType\":\"wechat\",\"wechat\":\"13022323120\",\"setTrackUrl\":true,\"pluginStyle\":\"card\",\"setWeb_experiment\":false,\"adTag\":\"广告\",\"setOriginUrl\":false,\"setCardType\":true,\"setAndroidUrl\":false}",
        "url": "https://xg.zhihu.com/plugin/10e7d5c0843db50441f0a6ce1a6eb38f?BIZ=ECOMMERCE",
        "url_token": "642866731"
    }
}
```
其他的原子节点如`video`、`code_block`等等节点和`image`、`card` 节点的定义差不多，就不在列举le。

完整的一个 json 结构
``` 
{
    "type": "article",
    "id": "19201404",
    "segments": [
        {
            "id": "0",
            "paragraph": {
                "marks": [],
                "text": "毕业之行的愿望拖了几年一直都没有实现，这次趁着四个人都有时间赶紧约起！“总要去一趟重庆吧，去感受一次山城的热情”这样一个充满了人情味和烟火气的城市深深吸引着我们。三千年巴渝，八百载重庆，这里，同时存在着过去和未来。"
            },
            "type": "paragraph"
        },
        {
            "id": "1",
            "paragraph": {
                "marks": [
                    {
                        "end_index": 106,
                        "start_index": 0,
                        "type": "bold"
                    }
                ],
                "text": "这次的重庆之旅我们没有选择跟团和自驾游，而是在朋友的推荐下选择了一位当地的旅游定制师，不用自己规划路线，预定酒店、门票，还没有强制消费，不像跟团一样限制太多，有想去的地方也可以提前和定制师小姐姐说，真的超级轻松！"
            },
            "type": "paragraph"
        },
        {
            "id": "2",
            "paragraph": {
                "marks": [
                    {
                        "end_index": 35,
                        "start_index": 0,
                        "type": "bold"
                    }
                ],
                "text": "这个导游小姐姐的联系方式我也放到下面了，有需要的姐妹也可以直接找她哦!"
            },
            "type": "paragraph"
        },
        {
            "card": {
                "biz_url": "https://xg.zhihu.com/plugin/10e7d5c0843db50441f0a6ce1a6eb38f?BIZ=ECOMMERCE",
                "card_type": "ad-link-card",
                "content_type": "ARTICLE",
                "cover": "https://pic3.zhimg.com/v2-9b4d085c47608d3b5b84f6f179524c2d.png?source=d6434cab",
                "extra_info": "{\"setAdjson\":true,\"setProductName\":true,\"setAppid\":false,\"productName\":\"重庆\",\"price\":0,\"externalInformation\":\"13022323120\",\"setCommercialType\":false,\"id\":990900,\"setChannelSource\":true,\"is_canvas\":false,\"setIs_canvas\":true,\"setPluginName\":false,\"setWechat\":true,\"setConversion\":true,\"setQrCode\":false,\"cardId\":\"Plugin_10e7d5c0843db50441f0a6ce1a6eb38f\",\"conversionIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=conversion\u0026ci=__CREATIVEID__\u0026pli=990900\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"setPackageName\":true,\"setAppInfo\":false,\"setDescription\":true,\"setEcomSource\":false,\"androidScore\":0.0,\"si\":\"08b72c11-b399-447b-8d6a-e2378c742e46\",\"web_experimentSize\":0,\"setPluginUrl\":true,\"setExInfo\":false,\"setLogo\":true,\"pluginToken\":\"10e7d5c0843db50441f0a6ce1a6eb38f\",\"setImplantType\":false,\"setPluginStyle\":true,\"url\":\"https://www.zhihu.com/people/lang-ye-shu-bang?WECHAT_ID=13022323120\",\"setExternalInformation\":true,\"setInteractionLaunch\":true,\"setIsPlugin\":false,\"setDeeplinkUrl\":true,\"setAdTag\":false,\"setCardId\":true,\"setVerifyType\":false,\"setTemplate_id\":false,\"isPlugin\":false,\"commercialType\":0,\"setAndroidScore\":true,\"setPrice\":true,\"picUrl\":\"https://pic3.zhimg.com/v2-9b4d085c47608d3b5b84f6f179524c2d.png?source=d6434cab\",\"validStatus\":0,\"logo\":\"https://pic2.zhimg.com/v2-9b4d085c47608d3b5b84f6f179524c2d.png?source=d6434cab\",\"setId\":true,\"setHashId\":false,\"setValidStatus\":false,\"trackUrl\":{\"viewSize\":1,\"setImgView\":false,\"conversionIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=conversion\u0026ci=__CREATIVEID__\u0026pli=990900\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"conversionSize\":1,\"impressionIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=impression\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"click\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=click\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"view\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=view\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"clickIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=click\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"imgViewSize\":0,\"clickSize\":1,\"setConversion\":true,\"setView\":true,\"impressionSize\":1,\"setClick\":true,\"impression\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=impression\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"imgClickSize\":0,\"setImgClick\":false,\"viewIterator\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=view\u0026ci=__CREATIVEID__\u0026pli=990900\u0026et=__ET__\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"setImpression\":true,\"conversion\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=conversion\u0026ci=__CREATIVEID__\u0026pli=990900\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"]},\"channelSource\":\"\",\"setShareDesc\":false,\"iosScore\":0.0,\"setShareUrl\":false,\"setUrl\":true,\"setAndroidStoreUrl\":false,\"adjson\":\"{\\\"ads\\\":[{\\\"view_x_tracks\\\":[],\\\"conversion_track_js\\\":[],\\\"debug_tracks\\\":[],\\\"click_tracks\\\":[],\\\"creatives\\\":[{\\\"asset\\\":{\\\"landing_url\\\":\\\"https://www.zhihu.com/people/lang-ye-shu-bang?WECHAT_ID=13022323120\\\"}}],\\\"impression_tracks\\\":[],\\\"view_tracks\\\":[],\\\"close_tracks\\\":[],\\\"conversion_tracks\\\":[]}]}\",\"setPhoneNumber\":false,\"setIosScore\":true,\"setIosAppStoreUrl\":false,\"deeplinkUrl\":\"\",\"description\":\"一键复制加玥玥微信号【13022323120】免费领取重庆旅游攻略\u003e\u003e\u003e\",\"conversionSize\":1,\"setLabelName\":false,\"setPluginToken\":true,\"pluginUrl\":\"https://xg.zhihu.com/plugin/10e7d5c0843db50441f0a6ce1a6eb38f?BIZ=ECOMMERCE\",\"packageName\":\"com.tencent.mm\",\"setBigImg\":false,\"setSi\":true,\"conversion\":[\"https://sugar.zhihu.com/plutus_adreaper/page_monitor_log?si=08b72c11-b399-447b-8d6a-e2378c742e46\u0026ti=__ATOKEN__\u0026pf=__OS__\u0026idfa=__IDFA__\u0026imei=__IMEI__\u0026androidid=__ANDROIDID__\u0026pt=article\u0026ed=BiBUKF0xBSkqGGNWB2F6BV1wAHrjCFe9I-NO5g==\u0026oaid=__OAID__\u0026at=conversion\u0026ci=__CREATIVEID__\u0026pli=990900\u0026zid=__ZONEID__\u0026ps=card\u0026pi=__PAGEID__\u0026ad_si=__SESSIONID__\u0026sqnc=__SEQUENCE__\u0026bi=__BUSINESSID__\u0026tst=__TASKTYPE__\u0026tsi=__TASKID__\u0026psi=__POSID__\u0026mi=__MIXINFO__\u0026dt=__DYNAMICTITLE__\u0026cbed=__CBED__\u0026plp=__PLUGINPOSITION__\u0026zpt=__ZPT__\"],\"implantType\":0,\"interactionLaunch\":\"去复制\",\"setScene\":false,\"setPicUrl\":true,\"cardType\":\"wechat\",\"wechat\":\"13022323120\",\"setTrackUrl\":true,\"pluginStyle\":\"card\",\"setWeb_experiment\":false,\"adTag\":\"广告\",\"setOriginUrl\":false,\"setCardType\":true,\"setAndroidUrl\":false}",
                "id": "10e7d5c0843db50441f0a6ce1a6eb38f",
                "url": "https://xg.zhihu.com/plugin/10e7d5c0843db50441f0a6ce1a6eb38f?BIZ=ECOMMERCE",
                "url_token": "642866731"
            },
            "id": "3",
            "type": "card"
        },
        {
            "blockquote": {
                "marks": [],
                "text": "D1：重庆→两江夜游\nD2：解放碑→仙女山大草原→天生三桥\nD3：武隆→天坑寨子→万盛最美黑山谷\nD4：万盛→磁器口古镇→李子坝轻轨穿楼→洪崖洞→解放碑结束行程"
            },
            "id": "4",
            "type": "blockquote"
        },
        {
            "heading": {
                "level": 1,
                "marks": [
                    {
                        "end_index": 10,
                        "start_index": 0,
                        "type": "bold"
                    }
                ],
                "text": "D1：重庆→两江夜游 "
            },
            "id": "5",
            "type": "heading"
        },
        {
            "id": "6",
            "paragraph": {
                "marks": [],
                "text": "高铁到重庆真的巨快，三四个小时的高铁就能搞定！工作人员把我们送到酒店的时候还挺早，我们就和定制师小姐姐商量能不能临时加一个两江夜游，看看重庆不一样角度的夜景。多亏旺季还没到，票不是特别难定！不然我们也感受不到重庆不一样的美！"
            },
            "type": "paragraph"
        },
        {
            "id": "7",
            "image": {
                "description": "",
                "height": 1066,
                "is_gif": false,
                "layout": "normal",
                "original_token": "v2-211236bc5a4a44fd7ca44ac28959b370",
                "original_urls": [
                    "https://picx.zhimg.com/v2-211236bc5a4a44fd7ca44ac28959b370_720w.jpg?source=7e7ef6e2"
                ],
                "status": "normal",
                "token": "v2-211236bc5a4a44fd7ca44ac28959b370",
                "urls": [
                    "https://pic1.zhimg.com/v2-211236bc5a4a44fd7ca44ac28959b370_b.jpg"
                ],
                "width": 1600
            },
            "type": "image"
        },
        {
            "heading": {
                "level": 1,
                "marks": [
                    {
                        "end_index": 18,
                        "start_index": 0,
                        "type": "bold"
                    }
                ],
                "text": "D2：解放碑→仙女山大草原→天生三桥"
            },
            "id": "8",
            "type": "heading"
        },
        {
            "id": "9",
            "paragraph": {
                "marks": [],
                "text": "解放碑是重庆的象征，它是抗战胜利的精神象征，是中国唯一一座纪念中华民族抗日战争胜利的纪念碑。三十年前重庆人常常说的“进城”指的就是去解放碑。"
            },
            "type": "paragraph"
        },
        {
            "id": "10",
            "image": {
                "description": "",
                "height": 1920,
                "is_gif": false,
                "layout": "normal",
                "original_token": "v2-7a2120f0afc7c3d8726c2d8ea4004da4",
                "original_urls": [
                    "https://picx.zhimg.com/v2-7a2120f0afc7c3d8726c2d8ea4004da4_720w.jpg?source=7e7ef6e2"
                ],
                "status": "normal",
                "token": "v2-7a2120f0afc7c3d8726c2d8ea4004da4",
                "urls": [
                    "https://pica.zhimg.com/v2-7a2120f0afc7c3d8726c2d8ea4004da4_b.jpg"
                ],
                "width": 1440
            },
            "type": "image"
        },
        {
            "id": "11",
            "paragraph": {
                "marks": [],
                "text": "雨后的仙女山真的好美，去的那天刚好是阴天，山上的温度很低。本来期望的是阳光灿烂，没成想到雾气弥漫，整个山间笼罩在雾中，期望没有满足但却是另一种惊喜，甚至拍出来效果比有阳光还好看些~拍照真的很出片~"
            },
            "type": "paragraph"
        },
        {
            "id": "12",
            "paragraph": {
                "marks": [],
                "text": "小Tips! :这里7、8月份有露营音乐会哦！"
            },
            "type": "paragraph"
        },
        {
            "id": "13",
            "image": {
                "description": "",
                "height": 1920,
                "is_gif": false,
                "layout": "normal",
                "original_token": "v2-f8bf698d48bf12f5335b356271a6d7e8",
                "original_urls": [
                    "https://pic1.zhimg.com/v2-f8bf698d48bf12f5335b356271a6d7e8_720w.jpg?source=7e7ef6e2"
                ],
                "status": "normal",
                "token": "v2-f8bf698d48bf12f5335b356271a6d7e8",
                "urls": [
                    "https://pic1.zhimg.com/v2-f8bf698d48bf12f5335b356271a6d7e8_b.jpg"
                ],
                "width": 1440
            },
            "type": "image"
        },
        {
            "id": "14",
            "paragraph": {
                "marks": [
                    {
                        "end_index": 37,
                        "entity_word": {
                            "attach_info_bytes": "sgJeChXmu6Hln47lsL3luKbpu4Tph5HnlLISBU1vdmllGIkSIJASKAE1AAAAADoHYXJ0aWNsZUAASABSJDI2N2M0YTFlLTZmZGMtNDViNC1hYzkyLWM5OGY5Yjg2MzI1Mw==",
                            "id": "0",
                            "type": "search",
                            "url": "https://www.zhihu.com/search?hybrid_search_extra=%7B%22sourceId%22%3A0%2C%22sourceType%22%3A%22search%22%7D\u0026hybrid_search_source=Entity\u0026q=%E6%BB%A1%E5%9F%8E%E5%B0%BD%E5%B8%A6%E9%BB%84%E9%87%91%E7%94%B2\u0026search_source=Entity",
                            "word": "满城尽带黄金甲"
                        },
                        "start_index": 30,
                        "type": "entity_word"
                    },
                    {
                        "end_index": 44,
                        "entity_word": {
                            "attach_info_bytes": "sgJXCgzlj5jlvaLph5HliJoSB1Vua25vd24YkxIglxIoATUAAAAAOgdhcnRpY2xlQABIAFIkMjY3YzRhMWUtNmZkYy00NWI0LWFjOTItYzk4ZjliODYzMjUz",
                            "id": "0",
                            "type": "search",
                            "url": "https://www.zhihu.com/search?hybrid_search_extra=%7B%22sourceId%22%3A0%2C%22sourceType%22%3A%22search%22%7D\u0026hybrid_search_source=Entity\u0026q=%E5%8F%98%E5%BD%A2%E9%87%91%E5%88%9A\u0026search_source=Entity",
                            "word": "变形金刚"
                        },
                        "start_index": 40,
                        "type": "entity_word"
                    }
                ],
                "text": "去重庆看自然景观一定要去天生三桥。地缝里是喀斯特地貌，也是《满城尽带黄金甲》和《变形金刚4》的取景地，往里走的时候可以看到一个很大的变形金刚。走到地缝中间还有大瀑布，被重庆人称为自己的百丈祭。"
            },
            "type": "paragraph"
        },
        {
            "id": "15",
            "image": {
                "description": "",
                "height": 1920,
                "is_gif": false,
                "layout": "normal",
                "original_token": "v2-ebf2bc2515b5503d45ea788be35de1c4",
                "original_urls": [
                    "https://pic1.zhimg.com/v2-ebf2bc2515b5503d45ea788be35de1c4_720w.jpg?source=7e7ef6e2"
                ],
                "status": "normal",
                "token": "v2-ebf2bc2515b5503d45ea788be35de1c4",
                "urls": [
                    "https://pic3.zhimg.com/v2-ebf2bc2515b5503d45ea788be35de1c4_b.jpg"
                ],
                "width": 1440
            },
            "type": "image"
        },
        {
            "heading": {
                "level": 1,
                "marks": [
                    {
                        "end_index": 14,
                        "start_index": 0,
                        "type": "bold"
                    }
                ],
                "text": "D3：武隆→天坑寨子→黑山谷"
            },
            "id": "16",
            "type": "heading"
        },
        {
            "id": "17",
            "paragraph": {
                "marks": [
                    {
                        "end_index": 19,
                        "entity_word": {
                            "attach_info_bytes": "sgJYCgzph43luobmrabpmoYSCExvY2F0aW9uGJEWIJUWKAE1AAAAADoHYXJ0aWNsZUAASABSJDI2N2M0YTFlLTZmZGMtNDViNC1hYzkyLWM5OGY5Yjg2MzI1Mw==",
                            "id": "0",
                            "type": "search",
                            "url": "https://www.zhihu.com/search?hybrid_search_extra=%7B%22sourceId%22%3A0%2C%22sourceType%22%3A%22search%22%7D\u0026hybrid_search_source=Entity\u0026q=%E9%87%8D%E5%BA%86%E6%AD%A6%E9%9A%86\u0026search_source=Entity",
                            "word": "重庆武隆"
                        },
                        "start_index": 15,
                        "type": "entity_word"
                    },
                    {
                        "end_index": 33,
                        "entity_word": {
                            "attach_info_bytes": "sgJVCgzpo57lpKnkuYvlkLsSBU1vdmllGJ8WIKMWKAE1AAAAADoHYXJ0aWNsZUAASABSJDI2N2M0YTFlLTZmZGMtNDViNC1hYzkyLWM5OGY5Yjg2MzI1Mw==",
                            "id": "0",
                            "type": "search",
                            "url": "https://www.zhihu.com/search?hybrid_search_extra=%7B%22sourceId%22%3A0%2C%22sourceType%22%3A%22search%22%7D\u0026hybrid_search_source=Entity\u0026q=%E9%A3%9E%E5%A4%A9%E4%B9%8B%E5%90%BB\u0026search_source=Entity",
                            "word": "飞天之吻"
                        },
                        "start_index": 29,
                        "type": "entity_word"
                    }
                ],
                "text": "去年在抖音上因为土和丑而爆红的重庆武隆当然要去啊！景区的【飞天之吻】评分0分外貌，10分体验，虽丑但升上去景色美爆了！能不能坐上去全靠运气，如果风很大的话，这个项目会暂停！"
            },
            "type": "paragraph"
        },
        {
            "id": "18",
            "image": {
                "description": "",
                "height": 1706,
                "is_gif": false,
                "layout": "normal",
                "original_token": "v2-e02fa4cdd60b3ea8d843d0bab4dbea7e",
                "original_urls": [
                    "https://picx.zhimg.com/v2-e02fa4cdd60b3ea8d843d0bab4dbea7e_720w.jpg?source=7e7ef6e2"
                ],
                "status": "normal",
                "token": "v2-e02fa4cdd60b3ea8d843d0bab4dbea7e",
                "urls": [
                    "https://pic3.zhimg.com/v2-e02fa4cdd60b3ea8d843d0bab4dbea7e_b.jpg"
                ],
                "width": 1280
            },
            "type": "image"
        },
        {
            "id": "19",
            "paragraph": {
                "marks": [
                    {
                        "end_index": 40,
                        "entity_word": {
                            "attach_info_bytes": "sgJXCgzlpKflnLDkuYvlv4MSB1Vua25vd24YlRogmRooATUAAAAAOgdhcnRpY2xlQABIAFIkMjY3YzRhMWUtNmZkYy00NWI0LWFjOTItYzk4ZjliODYzMjUz",
                            "id": "0",
                            "type": "search",
                            "url": "https://www.zhihu.com/search?hybrid_search_extra=%7B%22sourceId%22%3A0%2C%22sourceType%22%3A%22search%22%7D\u0026hybrid_search_source=Entity\u0026q=%E5%A4%A7%E5%9C%B0%E4%B9%8B%E5%BF%83\u0026search_source=Entity",
                            "word": "大地之心"
                        },
                        "start_index": 36,
                        "type": "entity_word"
                    }
                ],
                "text": "《爸爸去哪儿》让更多人对这世代居住于此的50多户土家族和苗族原住村民的“大地之心”——天坑寨子有所了解，“轻云薄雾绕山谷，返璞归真入田园”是我们对寨子的第一映像。如果有时间的话，强烈推荐大家在这里多住几天，体验一下世外桃源生活。"
            },
            "type": "paragraph"
        }
    ]
}

```
##### 标记
「标记」用于提供文本块节点内的文本样式及其他内容信息，如加粗、斜体、下划线、链接等等
###### 标记的通用属性
![mark_define](/img/2023-07-10/mark_define.jpg)

###### 标记的特定属性
**bold 加粗**
详细定义
![mark_bold_define](/img/2023-07-10/mark_bold_define.jpg)
###### JSON 结构
```
{
    "id": "6",
    "type": "paragraph",
    "paragraph": {
        "text": "适量港币：部分小店只收现金，提前兑换一点港币很有必要！",
        "marks": [
            {
                "end_index": 4,
                "start_index": 0,
                "type": "bold"
            }
        ]
    }
}
```
**entity_word 实体词**
详细定义
![实体词定义](/img/2023-07-10/entity_word_define.jpg)
**特别说明**
由于`实体词`的后面有 icon，所以需要再实体词的文本后面插入一个空格帮忙占位；同时，end_index 的位置需要再空格之后；   
Example:   
一个段落内的文本：`这是一段包含实体词的文本`，其中`实体词`这三个字被标记为实体词，那么这个段落对应的 `text` 字段应该是这样的 `这是一段包含实体词 的文本`；start_index 为 6，
end_index 为 10；   
实体词对应的 icon 图标存在了客户端，当识别到 mark 是 entity_word 时，将实体词对应的文字替换到 start_index ~ end_index - 1 的位置；同时将实体词对用的图标替换到 end_index - 1 ~ end_index ;
###### JSON 结构
```
{
    "id": "14",
    "type": "paragraph",
    "paragraph": {
        "text": "在香港，八达通相当于我们手机的便捷支付方式，不仅可以搭港铁全线、大巴、叮叮车和天星小轮，一些餐厅和便利店都能直接刷卡消费！除了在机场、地铁站和码头可以租用实体八达通卡，八达通亦推出旅客版App和Apple Pay，免去持卡不便。",
        "marks": [
            {
                "end_index": 21,
                "start_index": 4,
                "type": "bold"
            },
            {
                "end_index": 106,
                "start_index": 97,
                "type": "bold"
            },
            {
                "end_index": 7,
                "entity_word": {
                    "attach_info_bytes": "sgJUCgnlhavovr7pgJoSB0NvbXBhbnkY2AUg2wUoBDUAAAAAOgdhcnRpY2xlQABIAFIkMmEzNDZkYTUtMWM0NS00Nzc0LWE5MDAtNzc2MWMwOTM2Mzc4",
                    "id": "0",
                    "type": "search",
                    "url": "https://www.zhihu.com/search?hybrid_search_extra=%7B%22sourceId%22%3A0%2C%22sourceType%22%3A%22search%22%7D\u0026hybrid_search_source=Entity\u0026q=%E5%85%AB%E8%BE%BE%E9%80%9A\u0026search_source=Entity",
                    "word": "八达通"
                },
                "start_index": 4,
                "type": "entity_word"
            }
        ]
    }
}
```
上面这个 paragraph 节点文本中的实体词的渲染效果如下：
![entity_word_demo](/img/2023-07-10/entity_word_demo.jpg)

`下划线`、`斜体`、`引用`等类型的标记我就不再一一列举了，其实都大同小异；

根据上面的分析和描叙，我们定义了一套富文本结构化协议；`内容平台`根据我们定义的结构化协议，把 webView 页面里面的标签按照顺序转换成一个个对应的块节点；

#### 客户端解析
客户端在请求回答或文章的数据时，这个时候返回的是经过内容平台解析之后的数据，而不是以前的 `html`数据；
客户端在拿到回答或文章的页面数据时，根据`结构化协议`中各种节点的定义，去解析和渲染对应的节点；
下面是`节点`、`mode`、`cell` 它们之间的映射关系:
![node_mode_cell](/img/2023-07-10/node_mode_cell.jpg)

详情页完成 `native` 化之后，知乎详情页的页面布局和`PDD`的商品详情页的页面布局类似了

![zhihu_detail_native](/img/2023-07-10/zhihu_detail_native.jpg)

至此，知乎回答、文章、想法等详情页完成了 `native` 化；