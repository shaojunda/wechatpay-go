# 微信支付 API v3 Go SDK
[微信支付 API v3](https://wechatpay-api.gitbook.io/wechatpay-api-v3/) 官方Go语言客户端代码库。

## 版本信息

版本号：`v0.2.1`

版本功能：
1. 支持微信支付 API v3 请求签名与应答验签能力的 HTTP Client: `core.Client`，该 HTTP Client 在执行请求时将自动携带身份认证信息，并检查应答的微信支付签名。
2. 微信支付平台证书下载库 `core/downloader`，提供手动下载器`CertificateDownloader`，以及自动下载管理器`CertificateDownloaderMgr`
3. 微信支付回调通知处理库 `core/notify`，提供通知处理器`Handler`，可以对微信支付回调通知进行验签，然后对回调通知内容进行解密，并解析为特定的结构（如支付回调通知的`Transaction`），也可以选择解析为字典`map[string]interface{}`
4. 微信支付各服务API对应的SDK，目前仅包含： 
    - 微信核心支付4种常用支付接口（JSAPI支付, APP支付，H5支付，Native支付）的SDK。特别的，为【JSAPI支付】与【APP支付】提供了自动构建拉起支付所需签名的接口。
	- 微信支付4种文件上传接口的SDK
	- 微信支付证书下载接口的SDK
	- 更多API跟进中

兼容性：
本版本为测试版本，因为接口重命名/代码结构调整等原因，与之前版本存在不兼容的情况。

## 快速开始
### 安装
#### 1、使用 Go Modules 管理你的项目（如果项目目录下存在`go.mod`文件的话说明你的项目已经使用 Go Modules 进行管理）
如果你的项目还不是使用 Go Modules 进行管理，在项目根目录下执行：
```shell
go mod init
```

#### 2、添加依赖
在 `go.mod` 文件中加入对本SDK的依赖：
```
require (
    github.com/wechatpay-apiv3/wechatpay-go v0.2.1
)
```
并使用`go mod tidy`进行下载。

也可以直接在项目目录中执行： 
```shell
go get -u github.com/wechatpay-apiv3/wechatpay-go@v0.2.1
```
来自动完成`go.mod`的修改与SDK的下载。

### 初始化 Client
为向微信支付 API 发送请求，需要初始化一个 `core.Client` 实例。
```go
package main

import (
	"context"
	"crypto/rsa"
	"log"
	"net/http"

	"github.com/wechatpay-apiv3/wechatpay-go/core"
	"github.com/wechatpay-apiv3/wechatpay-go/core/option"
)

func main() {
	// 示例参数，实际使用时请自行初始化
	var (
		mchID                      string              // 商户号
		mchCertificateSerialNumber string              // 商户证书序列号
		mchPrivateKey              *rsa.PrivateKey     // 商户私钥
		mchAPIv3Key                string              // 商户APIv3密钥
		customHTTPClient           *http.Client        // 自定义客户端实例
	)

	ctx := context.Background()
	opts := []core.ClientOption{
		// 一次性设置 签名/验签/敏感字段加解密，并注册 平台证书下载器，自动定时获取最新的平台证书
		option.WithWechatPayAutoAuthCipher(mchID, mchCertificateSerialNumber, mchPrivateKey, mchAPIv3Key),
		// 设置自定义 HTTPClient 实例，不设置时默认使用 http.Client{}，并设置超时时间为 30s
		option.WithHTTPClient(customHTTPClient),
	}
	client, err := core.NewClient(ctx, opts...)
	if err != nil {
		log.Printf("new wechat pay client err:%s", err)
		return
	}
	// 接下来使用 client 进行请求发送
	_ = client
}
```
使用以上方法，可以快速构建一个集成了签名/验签的`Client`，除此以外，还会自动初始化全局默认的平台证书下载管理器`downloader.MgrInstance()`提供平台证书。
每24小时更新一次平台证书，保证验签所使用的证书更新。

如果你希望直接使用本地的平台证书，也可以使用如下初始化方法
```go
func main() {
	// 示例参数，实际使用时请自行初始化
	var (
		mchID                      string              // 商户号
		mchCertificateSerialNumber string              // 商户证书序列号
		mchPrivateKey              *rsa.PrivateKey     // 商户私钥
		wechatPayCertList          []*x509.Certificate // 平台证书列表
		customHTTPClient           *http.Client        // 自定义客户端实例
	)

	ctx := context.Background()
	opts := []core.ClientOption{
		// 一次性设置 签名/验签/敏感字段加解密，使用本地提供的平台证书列表
		option.WithWechatPayAuthCipher(mchID, mchCertificateSerialNumber, mchPrivateKey, wechatPayCertList),
		// 设置自定义 HTTPClient 实例，不设置时默认使用 http.Client{}，并设置超时时间为 30s
		option.WithHTTPClient(customHTTPClient),
	}
	client, err := core.NewClient(ctx, opts...)
	if err != nil {
		log.Printf("new wechat pay client err:%s", err)
		return
	}
	// 接下来使用 client 进行请求发送
	_ = client
}
```
`core.Client`初始化完成后，可以在多个goroutine中并发使用。

#### 名词解释

+ 商户API证书，是用来证实商户身份的。证书中包含商户号、证书序列号、证书有效期等信息，由证书授权机构(Certificate Authority ，简称CA)签发，以防证书被伪造或篡改。如何获取请见 [商户API证书](https://wechatpay-api.gitbook.io/wechatpay-api-v3/ren-zheng/zheng-shu#shang-hu-api-zheng-shu) 。
+ 商户API私钥。商户申请商户API证书时，会生成商户私钥，并保存在本地证书文件夹的文件apiclient_key.pem中。注：不要把私钥文件暴露在公共场合，如上传到Github，写在客户端代码等。
+ 微信支付平台证书。平台证书是指由微信支付负责申请的，包含微信支付平台标识、公钥信息的证书。商户可以使用平台证书中的公钥进行应答签名的验证。获取平台证书需通过 [获取平台证书列表](https://wechatpay-api.gitbook.io/wechatpay-api-v3/ren-zheng/zheng-shu#ping-tai-zheng-shu) 接口下载。
+ 证书序列号。每个证书都有一个由CA颁发的唯一编号，即证书序列号。如何查看证书序列号请看 [这里](https://wechatpay-api.gitbook.io/wechatpay-api-v3/chang-jian-wen-ti/zheng-shu-xiang-guan#ru-he-cha-kan-zheng-shu-xu-lie-hao) 。

### 发送 HTTP 请求
以 [下载微信支付平台证书](https://wechatpay-api.gitbook.io/wechatpay-api-v3/jie-kou-wen-dang/ping-tai-zheng-shu) 为例：
```go
var (
	result *core.APIResult
	err error
)
result, err = client.Get(ctx, "https://api.mch.weixin.qq.com/v3/certificates")
// 处理 result.Response
```
使用 `core.Client` 发送 HTTP 请求后会得到一个 `core.APIResult` 实例，其中包含了当次请求的请求报文 `*http.Request` 以及应答报文 `*http.Response`。

### 使用服务接口 SDK

#### 以 [JSAPI下单](https://pay.weixin.qq.com/wiki/doc/apiv3_partner/apis/chapter4_1_1.shtml) 为例

```go
import (
	"log"
	"github.com/wechatpay-apiv3/wechatpay-go/services/payments/jsapi"
)

func JsapiPrepay() error {
	// 请自行初始化 client
	var client *core.Client
	
	svc := jsapi.JsapiApiService{Client: client}
	// 得到prepay_id，以及调起支付所需的参数和签名
	resp, result, err := svc.PrepayWithRequestPayment(ctx,
		jsapi.PrepayRequest{
			Appid:       core.String("wxd678efh567hg6787"),
			Mchid:       core.String("1900009191"),
			Description: core.String("Image形象店-深圳腾大-QQ公仔"),
			OutTradeNo:  core.String("1217752501201407033233368018"),
			Attach:      core.String("自定义数据说明"),
			NotifyUrl:   core.String("https://www.weixin.qq.com/wxpay/pay.php"),
			Amount: &jsapi.Amount{
				Total: core.Int32(100),
			},
			Payer: &jsapi.Payer{
				Openid: core.String("oUpF8uMuAJO_M2pxb1Q9zNjWeS6o"),
			},
		},
	)
	
	if err == nil {
		log.Println(resp)
	} else {
		log.Println(err)
		return err
	}
}
```

#### 以 [查询订单](https://pay.weixin.qq.com/wiki/doc/apiv3_partner/apis/chapter4_1_2.shtml) 为例

```go
import (
	"log"
	"github.com/wechatpay-apiv3/wechatpay-go/services/payments/jsapi"
)

func QueryOrder() error {
	// 请自行初始化 client
	var client *core.Client
	svc := jsapi.JsapiApiService{Client: client}
	
	resp, result, err := svc.QueryOrderById(ctx,
		jsapi.QueryOrderByIdRequest{
			TransactionId: core.String("4200000985202103031441826014"),
			Mchid:         core.String("1900009191"),
		},
	)
	
	if err == nil {
		log.Println(resp)
	} else {
		log.Println(err)
		return err
	}
}
```

#### 以 [图片上传API](https://pay.weixin.qq.com/wiki/doc/apiv3/apis/chapter2_1_1.shtml) 为例：
```go
import (
	"os"
	"github.com/wechatpay-apiv3/wechatpay-go/core"
	"github.com/wechatpay-apiv3/wechatpay-go/core/consts"
	"github.com/wechatpay-apiv3/wechatpay-go/services/fileuploader"
)

func UploadImage() error {
	// 请自行初始化 client
	var client *core.Client
	
	file, err := os.Open("resource/demo.jpg")
	defer file.Close()
	if err != nil {
		return err
	}
	
	svc := fileuploader.ImageUploader{Client: client}
	resp, result, err := svc.Upload(ctx, file, "demo.jpg", consts.ImageJPG)
	// 处理 resp 
}
```

## 目录介绍
```
github.com/wechatpay-apiv3/wechatpay-go
├── core       // 本代码库核心功能
├── services   // 微信支付各服务API对应的SDK，目前只包含部分API，更多服务跟进中
├── docs       // 以上微信支付各服务API对应的SDK的说明文档
├── utils      // 各种常用工具函数
└── README.md  // 本文档
```

## 敏感信息加解密
为了保证通信过程中敏感信息字段（如用户的住址、银行卡号、手机号码等）的机密性，微信支付 API v3 要求商户对上送的敏感信息字段进行加密。
与之相对应，微信支付会对下行的敏感信息字段进行加密，商户需解密后方能得到原文。
详见[这里](https://wechatpay-api.gitbook.io/wechatpay-api-v3/qian-ming-zhi-nan-1/min-gan-xin-xi-jia-mi) 

### 加解密算法工具
工具包 `github.com/wechatpay-apiv3/wechatpay-go/utils` 中提供了加解密所需要的函数，可以使用这些方法手动对敏感信息字段进行加解密
```go
package utils

// EncryptOAEPWithPublicKey 使用公钥进行加密
func EncryptOAEPWithPublicKey(message string, publicKey *rsa.PublicKey) (ciphertext string, err error)
// EncryptOAEPWithCertificate 先解析出证书中的公钥，然后使用公钥进行加密
func EncryptOAEPWithCertificate(message string, certificate *x509.Certificate) (ciphertext string, err error)

// DecryptOAEP 使用私钥进行解密
func DecryptOAEP(ciphertext string, privateKey *rsa.PrivateKey) (message string, err error)
```

### 设置 `Wechatpay-Serial` 请求头
对请求参数进行加密后，需要在请求头中添加 `Wechatpay-Serial` 参数，用于传递加密所使用的微信支付平台证书序列号。

目前 `core.Client` 的 `Get/Post/Put/Patch/Delete/Upload` 方法只能传递最简单的内容，并不支持设置 HTTPHeader。
你可以使用 `Request` 方法来传输自定义 HTTPHeader。

### [建设中] 服务 SDK 自动加解密
目前我们已经完成了敏感字段自动加解密工具的开发，你可以使用`WithWechatPayAuthCipher`/`WithWechatPayAutoAuthCipher`/`WithCipher`等选项为
`core.Client` 设置加解密器。
但是比较遗憾的是，加解密器的正确运作依赖于我们在接口请求/应答结构所进行的Tag标记，也就是说只有我们提供了SDK的服务才能享受到自动加解密。
而目前我们所提供了SDK的服务接口均不存在需要加解密的字段，故而暂时这个能力还无法为大家提供服务。如果开发者需要访问存在**敏感字段**的接口，目前还是需要手动进行加解密。
我们会尽快上线更多服务接口的SDK，让自动加解密发挥它的价值。

## 回调通知验签与解密
`core/notify` 中提供了回调通知处理器 `Handler`，提供回调通知验签与解密功能。`Handler` 需要使用验签器 `Verifier` 和 商户APIv3密钥进行初始化。
```go
mchAPIv3Key := "<your api v3 key>"
verifier := verifiers.NewSHA256WithRSAVerifier(core.NewCertificateMapWithList([]*x509.Certificate{wechatPayCert})) 

handler := NewNotifyHandler(mchAPIv3Key, verifier)
```
初始化完成后即可调用`handler.ParseNotifyRequest`方法进行回调通知的验签及解密。
### 使用平台证书下载管理器初始化 `notify.Handler`
如果你已经知道如何在创建 `client` 的时候使用平台证书下载管理器自动更新平台证书，你可以在回调处理中也享受管理器提供了平台证书自动更新能力。
为了你能正确使用平台证书管理器，请确保你已经阅读并理解[在更多地方使用平台证书下载管理器](README.md#在更多地方使用平台证书下载管理器)的说明。

以下是使用平台证书下载管理器初始化 `notify.Handler` 的示例代码
```go
var (
	mchID                      string              // 商户号
	mchCertificateSerialNumber string              // 商户证书序列号
	mchPrivateKey              *rsa.PrivateKey     // 商户私钥
	mchAPIv3Key                string              // 商户APIv3密钥
)

func Example_a() {
	ctx := context.Background()
	client, err := core.NewClient(ctx, option.WithWechatPayAutoAuthCipher(mchID, mchCertificateSerialNumber, mchPrivateKey, mchAPIv3Key))
	if err != nil {
		return
	}
	
	// 由于已经使用 WithWechatPayAutoAuthCipher 在 downloader.MgrInstance() 中注册了商户的下载器，则下面可以直接使用该下载管理器获取证书
	certVisitor := downloader.MgrInstance().GetCertificateVisitor(mchID)
	handler := notify.NewNotifyHandler(mchAPIv3Key, verifiers.NewSHA256WithRSAVerifier(certVisitor))
}

func Example_b() {
	// 这是一个单纯的回调处理进程，没有使用 WithWechatPayAutoAuthCipher 创建商户的 client，此时则需要手动注册下载器
	ctx := context.Background()
	err := downloader.MgrInstance().RegisterDownloaderWithPrivateKey(ctx, mchPrivateKey, mchCertificateSerialNumber, mchID, mchAPIV3Key)
	if err != nil {
		return	
	}
	
	// 注册完成，使用下载管理器获取证书
	certVisitor := downloader.MgrInstance().GetCertificateVisitor(mchID)
	handler := notify.NewNotifyHandler(mchAPIv3Key, verifiers.NewSHA256WithRSAVerifier(certVisitor))
}

```

## 自定义签名生成器与验证器
当默认的本地签名和验签方式不适合你的系统时，你可以通过实现`Signer`或者`Verifier`来定制签名和验签。
比如，你可以把商户私钥集中存储，业务系统通过远程调用进行签名，你可以这样做。
```golang
// 签名器
type CustomSigner struct {
}

func (s *CustomSigner) Sign(ctx context.Context, message string) (*auth.SignatureResult, error) {
    // TODO: 远程调用获取签名信息
    return &auth.SignatureResult{MchID: "xxx", MchCertificateSerialNo: "xxx", Signature: "xxx"}, nil
}

// 校验器
type CustomVerifier struct {
}

func (v *CustomVerifier) Verify(ctx context.Context, serial, message, signature string) error {
    // TODO: 远程调用验签
    return nil
}
```

当你需要使用自定的签名器和校验器时，你可以这样来创建客户端
```golang
package core_test

import (
	"context"

	"path/to/your/custom_signer"
	"path/to/your/custom_verifier"

	"github.com/wechatpay-apiv3/wechatpay-go/core"
	"github.com/wechatpay-apiv3/wechatpay-go/core/auth/credentials"
	"github.com/wechatpay-apiv3/wechatpay-go/core/auth/validators"
	"github.com/wechatpay-apiv3/wechatpay-go/core/option"
)

func NewCustomClient(ctx context.Context, mchID string) (*core.Client, error) {
	signer := &custom_signer.CustomSigner{
		// ... 
	}
	verifier := &custom_verifier.CustomVerifier{
		// ...
	}

	opts := []core.ClientOption{
		option.WithSigner(signer),
		option.WithVerifier(verifier),
	}

	return core.NewClient(ctx, opts...)
}
```

## 常见问题
### 如何下载平台证书
现在本 SDK 已经提供平台证书下载器库以及命令行工具供开发者使用。
#### 在代码中下载平台证书
使用 `cert.CertificateDownloader` 下载平台证书
```go
var (
	mchID           string
	mchCertSerialNo string
	mchPrivateKey   *rsa.PrivateKey
	mchAPIv3Key     string
)
// 假设以上参数已初始化完成

d, err := cert.NewCertificateDownloader(ctx, mchID, mchPrivateKey, mchCertSerialNo, mchAPIv3Key)
if err != nil {
	return err
}
// Downloader 创建时会立即进行一次下载操作，创建成功后即可直接访问

// 访问平台证书内容
for serialNo, certContent := range d.ExportAll(ctx) {
	// ...
}

// 获得解析好的平台证书结构 *x509.Certificate
for serialNo, cert := range d.GetAll(ctx) {
	// ...
}

// 重新下载证书(如果你认为有必要的话）
err := d.DownloadCertificates(ctx)
if err != nil {
	return err
}
```
微信支付推荐每24小时更新一次平台证书列表，来保证平台证书的平滑替换。为此本 SDK 还提供了自动平台证书下载管理器 `CertificateDownloaderMgr`
对一般用户而言，你可以在初始化 Client 时使用前文所建议的`option.WithWechatPayAutoAuthCipher`，便可以享受到我们为你建立的默认证书下载管理器。
你可以使用`downloader.MgrInstance()`访问到这个默认的管理器，请不要调用它的`Stop()`方法，除非你知道自己在做什么。

使用`option.WithWechatPayAutoAuthCipher`初始化`core.Client`后，会自动在默认的证书下载管理器中注册对应商户的证书自动更新任务。
它除了会为`core.Client`中的验签器提供平台证书外，你也可以直接使用`mgr.GetCertificate`或`mgr.GetCertificateVisitor`等方法直接获取到证书，这些证书可以用于敏感字段的加密。

#### 在更多地方使用平台证书下载管理器
> 注意：如果你想使用 `downloader.MgrInstance().GetCertificate` 或 `downloader.MgrInstance().GetCertificateVisitor` 来获取某个商户`mchID`的平台证书（比如用于回调通知验签），
> 请确保你在该进程中使用过 `option.WithWechatPayAutoAuthCipher` 创建该商户的 `client`。
> 这是因为平台证书下载管理器必须使用商户信息注册下载器之后才能提供对应商户的平台证书。
> 如果你并不需要创建该商户的 client，请使用 `downloader.MgrInstance().RegisterDownloaderWithPrivateKey` 手动注册下载器。
>
> `option.WithWechatPayAutoAuthCipher` 和 `downloader.MgrInstance().RegisterDownloaderWithPrivateKey` 只需要在进程中调用过一次即可，多次调用无实际意义。
> 如果二者都没有调用过，则查询该商户的平台证书时结果均为空。

如果你希望了解更多，并自行对平台证书下载管理器的生命周期进行管理，可以自行阅读 [`core/downloader`](core/downloader) 的代码。

#### 使用命令行工具下载证书到本地目录
首先使用 `go` 指令下载命令行工具
```shell
go get -u github.com/wechatpay-apiv3/wechatpay-go/cmd/wechatpay_download_certs@v0.2.1
```
然后执行 `wechatpay_download_certs` 即可下载平台证书到当前目录
```shell
wechatpay_download_certs -m <mchID> -p <mchPrivateKeyPath> -s <mchSerialNo> -k <mchAPIv3Key>
```
完整参数列表可运行 `wechatpay_download_certs -h` 进行查看。

### 如何下载账单
[账单下载API](https://pay.weixin.qq.com/wiki/doc/apiv3/wxpay/pay/bill/chapter3_3.shtml) 分成了两个步骤：
1. `/v3/bill/tradebill` 获取账单下载链接和账单摘要
2. `/v3/billdownload/file` 账单文件下载，请求需签名但应答不签名

其中第二步的应答中不包含应答数字签名，无法进行验签，应使用`WithoutValidator()`**跳过**应答签名的校验。
```go
opts := []core.ClientOption{
	option.WithMerchantCredential(mchID, mchCertificateSerialNumber, privateKey),
	option.WithoutValidator(),
}

client, err := core.NewClient(ctx, opts...)
```

> **注意**：第一步中应正常对应答签名进行验证
> 
> **注意**：开发者在下载文件之后，应使用第一步获取的账单摘要校验文件的完整性

### 如何查看 HTTP 请求的 Request 信息
不论是使用 `Client` 的 HTTP 方法（`Get/Post/Put/Delete`等）直接发送 HTTP 请求，还是使用服务API对应的SDK发起请求，均会返回 `*core.APIResult` 结构。
该结构中包含了本次发起 HTTP 请求的 `http.Request` 对象和微信支付应答的 `http.Response` 对象。
```golang
var (
	request *http.Request
	response *http.Response
)
result, err = client.Get(ctx, "")

request = result.Request
response = result.Response 
```

## 联系我们
如果你发现了BUG或者有任何疑问、建议，欢迎通过 [issue](https://github.com/wechatpay-apiv3/wechatpay-go/issues) 进行反馈

也欢迎访问我们的[开发者社区](https://developers.weixin.qq.com/community/pay)
