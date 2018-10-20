---
layout:       post
title:        "SpringBoot之支付宝支付 -- 手机网站支付"
subtitle:     "手机网站支付"
date:         2018-10-08 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - SpringBoot
---
# SpringBoot之支付宝支付 -- 手机网站支付
## 内容目录
[TOC]
## 手机网站支付介绍
&emsp;&emsp;适用于商家在移动端网页应用中集成支付宝支付功能.  
&emsp;&emsp;商家在网页中调用支付宝提供的网页支付接口调起支付宝客户端内的支付模块,商家网页会跳转到支付宝中完成支付,支付完后跳回到商家网页内,最后展示支付结果.若无法唤起支付宝客户端,则在一定的时间后会自动进入网页支付流程
## 产品流程
### 用户已安装支付宝
步骤1:用户在浏览器中访问商家网页应用,选择商品下单、确认购买,进入支付环节,选择支付宝付款,用户点击去支付  
步骤2:进入到支付宝支付路由页面,支付宝处理支付请求,并尝试唤起支付宝客户端  
步骤3:进入到支付宝页面,调起支付宝支付,出现确认支付界面  
步骤4:用户确认收款方和金额,点击立即支付后出现输入密码界面  
步骤5:输入正确密码后,支付宝端显示支付结果  
步骤6:自动回跳到浏览器中,商家根据付款结果个性化展示订单处理结果  
### 用户未安装支付宝
步骤1:若用户未安装支付宝客户端,用户进入到支付宝网页收银台,用户登录支付宝账户  
步骤2:登录成功后,进入付款确认页面  
步骤3:用户点击确认付款,进入支付密码页面  
步骤4:用户输入密码,完成支付,展示支付结果  
## 手机网站支付流程
手机网站支付包括两类API:
- 页面跳转类:需要从前端页面以Form表单的形式发起请求,浏览器会自动跳转至支付宝的相关页面,用户在该页面完成相关业务操作后,再回跳到商户指定页面.例如手机网站支付接口
- 系统调用类:直接从服务端发起HTTP请求,支付宝会同步返回请求结果.例如交易查询接口
### 调用流程
![手机网站调用流程](/img/in-post/SpringBoot/Alipay/手机网站支付调用流程.png)
## 手机网站支付示例
### 利用SDK快速接入
手机网站支付alipay.trade.wap.pay:  
&emsp;&emsp;对于页面跳转类API,SDK不会也无法像系统调用类API一样自动请求支付宝并获得结果,而是在接受request请求对象后,为开发者生成前台页面请求需要的完整form表单的html(包含自动提交脚本),商户直接将这个表单的String输出到http response中即可  
```
public void doPost(HttpServletRequest httpRequest,
HttpServletResponse httpResponse) throws ServletException, IOException {
AlipayClient alipayClient = new DefaultAlipayClient("https://openapi.alipay.com/gateway.do", APP_ID, APP_PRIVATE_KEY, "json", CHARSET, ALIPAY_PUBLIC_KEY, "RSA2"); //获得初始化的AlipayClient
AlipayTradeWapPayRequest alipayRequest = new AlipayTradeWapPayRequest();//创建API对应的request
alipayRequest.setReturnUrl("http://domain.com/CallBack/return_url.jsp");
alipayRequest.setNotifyUrl("http://domain.com/CallBack/notify_url.jsp");//在公共参数中设置回跳和通知地址
alipayRequest.setBizContent("{" +
" \"out_trade_no\":\"20150320010101002\"," +
" \"total_amount\":\"88.88\"," +
" \"subject\":\"Iphone6 16G\"," +
" \"product_code\":\"QUICK_WAP_PAY\"" +
" }");//填充业务参数
String form="";
try {
form = alipayClient.pageExecute(alipayRequest).getBody(); //调用SDK生成表单
} catch (AlipayApiException e) {
e.printStackTrace();
}
httpResponse.setContentType("text/html;charset=" + CHARSET);
httpResponse.getWriter().write(form);//直接将完整的表单html输出到页面
httpResponse.getWriter().flush();
httpResponse.getWriter().close();
}
```
SDK快速接入方式完成上述代码即可条码支付
### 利用官方Demo使用SpringBoot方式接入
此处代码参考了https://blog.csdn.net/vbirdbest/article/details/80684460博客,在此声明  
#### pom文件引入Alipay依赖  
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<dependency>
    <groupId>com.alipay</groupId>
    <artifactId>alipay-sdk-java</artifactId>
    <version>20170725114550</version>
</dependency>
<dependency>
    <groupId>com.alipay</groupId>
    <artifactId>alipay-trade-sdk</artifactId>
    <version>20161215</version>
</dependency>
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
</dependency>
<dependency>
    <groupId>commons-configuration</groupId>
    <artifactId>commons-configuration</artifactId>
    <version>1.10</version>
</dependency>
<dependency>
    <groupId>com.google.zxing</groupId>
    <artifactId>core</artifactId>
    <version>3.2.1</version>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```
#### yml文件配置  
```
# 沙箱账号
pay:
  alipay:
    gatewayUrl: https://openapi.alipaydev.com/gateway.do
    appid: 2016091400508498
    appPrivateKey: MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCtXKWs+trRSuCxEUvlsEeSAuLWs3B/Dh74R8223BnfzoA29aGeoycAqfKlBMcbzU2G1KayESvZKGpwBLeejSbecRYjgZsQDyEaDimQQJtGFvZVV6u4XNnvIJ72eQzEaEIQfuiorjBTLm6DQuds4R0GxftqON6QFoIZkWB9ZrZKd02cuy16dW+UqtLVGGAHcCIAkB63zUiKSNfweMddneZ7MVs3lvu3xhMnD+5us/+n2Vp4qhfmpYLcdqIW6InU4GypeoOpyUTrfUGpgdR0l924vHy/GQJZEKFaRcK3cYK+ECyKpSIoqaJJFLHbkqsliuPpMUG+rM3jiqeIAH4psAznAgMBAAECggEBAJ5jyEbbxrJzrAh7GhHX1fwUQPYSadTbrPYAfHX2cHlnrQMJtsk+nTLhEv0r+VJwZ8WpYkfMong8kcqYtL7ajcmsHqMAFhE9EWxBxj2ymWsXLabZe93sj30IG9Rq0nxcGQgDO0RqKWLGSFgK93Al2KRInKT3InkY53K+vR61ihVLmGf7+GwPtIZteBy+tuAyvcj2RlkYvjiFi3ySyZ1wA3sJIlgrGiixd6fj20XFGNE3CnAwu0BJgXXbP/S9J4C0RRa3ZXj8fX7oONhVxz2xKxn7AT4e8OWjt7J41H2LRct8Fgl9pqgz2FJYv/WfbkG8x9fGiKYYvPXoxjjrk/tkewkCgYEA8f9Lcu5JPrE9rpw9zlwhm5cOO81xLxdwL5R5/1bRP48BZGIYuqlCbVvjJVqtO8eTnLhUwH7fG8B7cmoeO9bGr9GQrtfyCqz6FtVymTBieJlfgZDVhtzyv2qKOBMIFE8jsbSBK/NHHMvykJ+XdQ1riwCeQDdXICRuYTTFwGk2OsUCgYEAt2SoN95tVmVrvKG6ATLNEtge/ozeVywA4GjltrSw/G9vqp+DkkT2pY19uROuzMazoTzKWpPho2q/qzNlv/ANbOFM2GEmKamQ7CO88JgRxMsPTvc/HxCLU/ClMJU8LcOf9LfP2KYZpPwuheKJoF4vDGj8NsbFmccJyYSdpkNEk7sCgYBJlL2FMaz1sgC2Ue19DIhvfaunRV1P20mSPgwmNmijccETm7w3LXX0OIdFeV/JGHLqqSWj7i+6iXk/ncKZoUGCfi8G6sQ+uL/GJ5qTt6GJV+ExTS+PtSjeSO/EAw1m13Vb+C16hpstx1l23f+4aJ81gbecgPct38XsKpaiXZtOnQKBgQCMsN7QRYYxwoq9YoDUzIlAzKYyeBVWYL6najHYUZR5hG/xQIBqZRem9/4cTvpJxKInrwA6LrrqaEl0aHDFp75U6h7O3PCvA5PXZK9dD/yJsZIj7U/yX/nTQokn1UUegrYiwiTkusBvrrtuINWePsLvTVc4GpObHnPmsiNTWsWwYwKBgENaeTNOCHV2km/ysXQSEIhKbtlAMQPsgWHCt/bzHlF9m18izb1LrJyjzcSsd+Zy78R+pv4G50Q27c3e/DFPz/wYxN/yHWRbyLBA8ipJbCtMtPEdS9krpmN6cChIdLGbz4CVUqOPSRzNb9lhhgPCcCNRq6DG3HBceb1Se9VnO3zk
    alipayPublicKey: MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEApFQKccMq+wPoWh93DcX3XYrnT7FJ3gntJA/jEwgk6Jd3iEVS2CyUCCgFVcQn8xjXT81YbZHYvoC50IBuu+A+Ey+J8VIgsBm5g9uwbOLRf66GrZjuKOlalHm5gHXjcL2gZRMBJEStOxstcO2YQriqhQzdL3EKp+HQc9u14IOVFpZdR8qq1l7CzKHn1vQo/1fUPfUrTLQqSuQTU9Ospv/QHFqOJA7DPetUQ+jnaZ082f3clr4ITw4EE8A6IWqTXcETTx5j/udCGP84g2Y4j+8i9DqYGyD5ePVgt4G0ICBX1bi1qNlylfxRg8Y3c1DFrRGyr0NpKQxSVXkYaVNvrCoudQIDAQAB
    returnUrl: http://yxep7y.natappfree.cc/alipay/return # 手机网站支付成功后跳转页面
    notifyUrl: http://yxep7y.natappfree.cc/alipay/notify

spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
    mode: HTML5
    encoding: UTF-8
```
#### 配置类  
```
@Data
@Slf4j
@ConfigurationProperties(prefix = "pay.alipay")
public class AlipayProperties {

    /** 支付宝gatewayUrl */
    private String gatewayUrl;
    /** 商户应用id */
    private String appid;
    /** RSA私钥，用于对商户请求报文加签 */
    private String appPrivateKey;
    /** 支付宝RSA公钥，用于验签支付宝应答 */
    private String alipayPublicKey;
    /** 签名类型 */
    private String signType = "RSA2";

    /** 格式 */
    private String formate = "json";
    /** 编码 */
    private String charset = "UTF-8";

    /** 同步地址 */
    private String returnUrl;

    /** 异步地址 */
    private String notifyUrl;

    /** 最大查询次数 */
    private static int maxQueryRetry = 5;
    /** 查询间隔（毫秒） */
    private static long queryDuration = 5000;
    /** 最大撤销次数 */
    private static int maxCancelRetry = 3;
    /** 撤销间隔（毫秒） */
    private static long cancelDuration = 3000;

    private AlipayProperties() {}

    /**
     * PostContruct是spring框架的注解，在方法上加该注解会在项目启动的时候执行该方法，也可以理解为在spring容器初始化的时候执行该方法。
     */
    @PostConstruct
    public void init() {
        log.info(description());
    }

    public String description() {
        StringBuilder sb = new StringBuilder("\nConfigs{");
        sb.append("支付宝网关: ").append(gatewayUrl).append("\n");
        sb.append(", appid: ").append(appid).append("\n");
        sb.append(", 商户RSA私钥: ").append(getKeyDescription(appPrivateKey)).append("\n");
        sb.append(", 支付宝RSA公钥: ").append(getKeyDescription(alipayPublicKey)).append("\n");
        sb.append(", 签名类型: ").append(signType).append("\n");

        sb.append(", 查询重试次数: ").append(maxQueryRetry).append("\n");
        sb.append(", 查询间隔(毫秒): ").append(queryDuration).append("\n");
        sb.append(", 撤销尝试次数: ").append(maxCancelRetry).append("\n");
        sb.append(", 撤销重试间隔(毫秒): ").append(cancelDuration).append("\n");
        sb.append("}");
        return sb.toString();
    }

    private String getKeyDescription(String key) {
        int showLength = 6;
        if (StringUtils.isNotEmpty(key) && key.length() > showLength) {
            return new StringBuilder(key.substring(0, showLength)).append("******")
                    .append(key.substring(key.length() - showLength)).toString();
        }
        return null;
    }
}

@Configuration
@EnableConfigurationProperties(AlipayProperties.class)
public class AlipayConfiguration {

    @Autowired
    private AlipayProperties properties;

    @Bean
    public AlipayTradeService alipayTradeService() {
        return new AlipayTradeServiceImpl.ClientBuilder()
                .setGatewayUrl(properties.getGatewayUrl())
                .setAppid(properties.getAppid())
                .setPrivateKey(properties.getAppPrivateKey())
                .setAlipayPublicKey(properties.getAlipayPublicKey())
                .setSignType(properties.getSignType())
                .build();
    }

    @Bean
    public AlipayClient alipayClient(){
        return new DefaultAlipayClient(properties.getGatewayUrl(),
                properties.getAppid(),
                properties.getAppPrivateKey(),
                properties.getFormate(),
                properties.getCharset(),
                properties.getAlipayPublicKey(),
                properties.getSignType());
    }
}
```
#### Controller代码  
```
@Slf4j
@Controller
@RequestMapping("/alipay/wap")
public class AlipayWAPPayController {

    @Autowired
    private AlipayProperties alipayProperties;

    @Autowired
    private AlipayClient alipayClient;

    /**
     * 去支付
     *
     * 支付宝返回一个form表单，并自动提交，跳转到支付宝页面
     *
     * @param response
     * @throws Exception
     */
    @PostMapping("/alipage")
    public void gotoPayPage(HttpServletResponse response) throws AlipayApiException, IOException {
        // 订单模型
        String productCode="QUICK_WAP_WAY";
        AlipayTradeWapPayModel model = new AlipayTradeWapPayModel();
        model.setOutTradeNo(UUID.randomUUID().toString());
        model.setSubject("支付测试");
        model.setTotalAmount("0.01");
        model.setBody("支付测试，共0.01元");
        model.setTimeoutExpress("2m");
        model.setProductCode(productCode);

        AlipayTradeWapPayRequest wapPayRequest =new AlipayTradeWapPayRequest();
        wapPayRequest.setReturnUrl("http://yxep7y.natappfree.cc/alipay/wap/returnUrl");
        wapPayRequest.setNotifyUrl(alipayProperties.getNotifyUrl());
        wapPayRequest.setBizModel(model);

        // 调用SDK生成表单, 并直接将完整的表单html输出到页面
        String form = alipayClient.pageExecute(wapPayRequest).getBody();
        System.out.println(form);
        response.setContentType("text/html;charset=" + alipayProperties.getCharset());
        response.getWriter().write(form);
        response.getWriter().flush();
        response.getWriter().close();
    }


    /**
     * 支付宝页面跳转同步通知页面
     * @param request
     * @return
     * @throws UnsupportedEncodingException
     * @throws AlipayApiException
     */
    @RequestMapping("/returnUrl")
    public String returnUrl(HttpServletRequest request, HttpServletResponse response) throws UnsupportedEncodingException, AlipayApiException {
        response.setContentType("text/html;charset=" + alipayProperties.getCharset());

        //获取支付宝GET过来反馈信息
        Map<String,String> params = new HashMap<>();
        Map requestParams = request.getParameterMap();
        for (Iterator iter = requestParams.keySet().iterator(); iter.hasNext();) {
            String name = (String) iter.next();
            String[] values = (String[]) requestParams.get(name);
            String valueStr = "";
            for (int i = 0; i < values.length; i++) {
                valueStr = (i == values.length - 1) ? valueStr + values[i]
                        : valueStr + values[i] + ",";
            }
            //乱码解决，这段代码在出现乱码时使用。如果mysign和sign不相等也可以使用这段代码转化
            valueStr = new String(valueStr.getBytes("ISO-8859-1"), "utf-8");
            params.put(name, valueStr);
        }

        boolean verifyResult = AlipaySignature.rsaCheckV1(params, alipayProperties.getAlipayPublicKey(), alipayProperties.getCharset(), "RSA2");
        if(verifyResult){
            //验证成功
            //请在这里加上商户的业务逻辑程序代码，如保存支付宝交易号
            //商户订单号
            String out_trade_no = new String(request.getParameter("out_trade_no").getBytes("ISO-8859-1"),"UTF-8");
            //支付宝交易号
            String trade_no = new String(request.getParameter("trade_no").getBytes("ISO-8859-1"),"UTF-8");

            return "wapPaySuccess";

        }else{
            return "wapPayFail";

        }
    }

    /**
     * 退款
     * @param orderNo 商户订单号
     * @return
     */
    @PostMapping("/refund")
    @ResponseBody
    public String refund(String orderNo) throws AlipayApiException {
        AlipayTradeRefundRequest alipayRequest = new AlipayTradeRefundRequest();

        AlipayTradeRefundModel model=new AlipayTradeRefundModel();
        // 商户订单号
        model.setOutTradeNo(orderNo);
        // 退款金额
        model.setRefundAmount("0.01");
        // 退款原因
        model.setRefundReason("无理由退货");
        // 退款订单号(同一个订单可以分多次部分退款，当分多次时必传)
//        model.setOutRequestNo(UUID.randomUUID().toString());
        alipayRequest.setBizModel(model);

        AlipayTradeRefundResponse alipayResponse = alipayClient.execute(alipayRequest);
        System.out.println(alipayResponse.getBody());

        return alipayResponse.getBody();
    }

    /**
     * 退款查询
     * @param orderNo 商户订单号
     * @param refundOrderNo 请求退款接口时，传入的退款请求号，如果在退款请求时未传入，则该值为创建交易时的外部订单号
     * @return
     * @throws AlipayApiException
     */
    @GetMapping("/refundQuery")
    @ResponseBody
    public String refundQuery(String orderNo, String refundOrderNo) throws AlipayApiException {
        AlipayTradeFastpayRefundQueryRequest alipayRequest = new AlipayTradeFastpayRefundQueryRequest();

        AlipayTradeFastpayRefundQueryModel model=new AlipayTradeFastpayRefundQueryModel();
        model.setOutTradeNo(orderNo);
        model.setOutRequestNo(refundOrderNo);
        alipayRequest.setBizModel(model);

        AlipayTradeFastpayRefundQueryResponse alipayResponse = alipayClient.execute(alipayRequest);
        System.out.println(alipayResponse.getBody());

        return alipayResponse.getBody();
    }

    /**
     * 关闭交易
     * @param orderNo
     * @return
     * @throws AlipayApiException
     */
    @PostMapping("/close")
    @ResponseBody
    public String close(String orderNo) throws AlipayApiException {
        AlipayTradeCloseRequest alipayRequest = new AlipayTradeCloseRequest();
        AlipayTradeCloseModel model =new AlipayTradeCloseModel();
        model.setOutTradeNo(orderNo);
        alipayRequest.setBizModel(model);

        AlipayTradeCloseResponse alipayResponse= alipayClient.execute(alipayRequest);
        System.out.println(alipayResponse.getBody());

        return alipayResponse.getBody();
    }
}
```
#### WebMvcConfiguration  
通过访问http://localhost:8080/toPay来跳转到toPay.html页面  
```
@Configuration
public class WebMvcConfiguration extends WebMvcConfigurationSupport {
    @Override
    protected void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/toPay").setViewName("toPay");
        super.addViewControllers(registry);
    }
}
```
#### templates  
toPay.html  
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body style="font-size: 30px">

<form method="post" action="/alipay/wap/alipage">
    <h3>购买商品：越南新娘</h3>
    <h3>价格：20000</h3>
    <h3>数量：2个</h3>

    <button style="width: 100%; height: 60px; alignment: center; background: blue" type="submit">支付</button>
</form>

</body>
</html>
```
wapPaySuccess.html  
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>

<h1>WAP 支付成功，请及时享用！欢迎下次再来</h1>

</body>
</html>
```
wapPayFail.html  
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>

<h2>WAP 支付失败，请重新支付</h2>

</body>
</html>
```
