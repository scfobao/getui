# 个推V2版本

## 前言
由于个推没有Go的V2版本的SDK，V1版本快要不维护了，因此写个轮子，getui_test.go文件有各个方法的示例，方便各位快速上手

## 文件结构
```
getui
    |--auth         鉴权API：获取token，删除token
    |--publics      公共结构体，公共方法
    |--push         推送API
        |--single   单推送API：cid单推，别名单推，cid批量单推，别名批量单推
        |--list     批量推API：创建消息，cid批量推，别名批量推
        |--all      群推API：群推，根据条件筛选用户推送，使用标签快速推送
        |--mission  任务API：停止任务，查询定时任务，删除定时任务
    |--report       统计API（未开始）
    |--user         用户API（未开始）
    |--测试文件及其他

```

## 快速开始
```
#厂商通道普通模板+个推通道普通模板
#策略为4：先用厂商通道，失败后再用个推通道

package main

import (
	"context"
	"fmt"
	"log"
	"strconv"
	"time"

	"github.com/scfobao/getui/auth"
	"github.com/scfobao/getui/publics"
	"github.com/scfobao/getui/push/single"
)

func main() {
	cid := "XXXXXXXXXXXXXXXXXXXXXXXXXXXXX" // clientId
	ctxLocal := context.Background()
	tokenLocal := ""
	confLocal := publics.GeTuiConfig{
		AppId:        "", // 个推提供的id，密码等
		AppSecret:    "",
		AppKey:       "",
		MasterSecret: "",
	}
	// 1.获取token，注意这个token过期时间为一天+1秒，每分钟调用量为100次，每天最大调用量为10万次
	result, err := auth.GetToken(ctxLocal, confLocal)
	if err != nil {
		log.Fatalln("error:", err)
	}
	tokenLocal = result.Data.Token

	// 2.1ios厂商通道的参数
	iosChannel := publics.IosChannel{
		Type: "",
		Aps: &publics.Aps{
			Alert: &publics.Alert{
				Title: "卡是谁啊？",
				Body:  "为什么我们每天都要打 TA ？",
			},
			ContentAvailable: 0,
		},
		AutoBadge:      "+1",
		PayLoad:        "",
		Multimedia:     nil,
		ApnsCollapseId: "",
	}

	// 2.2安卓通道和个推通道的普通推送参数
	notification := publics.Notification{
		Title:       "卡是谁啊？",
		Body:        "为什么我们每天都要打 TA ？",
		ClickType:   "startapp", // 打开应用首页
		BadgeAddNum: 1,
	}

	// 2.3单推需要的参数
	singleParam := single.PushSingleParam{
		RequestId: strconv.FormatInt(time.Now().UnixNano(), 10), // 请求唯一标识号
		Audience: &publics.Audience{ // 目标用户
			Cid:           []string{cid}, // cid推送数组
			Alias:         nil,           // 别名送数组
			Tag:           nil,           // 推送条件
			FastCustomTag: "",            // 使用用户标签筛选目标用户
		},
		Settings: &publics.Settings{ // 推送条件设置
			TTL: 3600000, // 默认一小时，消息离线时间设置，单位毫秒
			Strategy: &publics.Strategy{ // 厂商通道策略，具体看public_struct.go
				Default: 1,
				Ios:     4,
				St:      4,
				Hw:      4,
				Xm:      4,
				Vv:      4,
				Mz:      4,
				Op:      4,
			},
			Speed:        100, // 推送速度，设置100表示：100条/秒左右，0表示不限速
			ScheduleTime: 0,   // 定时推送时间，必须是7天内的时间，格式：毫秒时间戳
		},
		PushMessage: &publics.PushMessage{
			Duration:     "", // 手机端通知展示时间段
			Notification: &notification,
			Transmission: "",
			Revoke:       nil,
		},
		PushChannel: &publics.PushChannel{
			Ios: &iosChannel,
			Android: &publics.AndroidChannel{Ups: &publics.Ups{
				Notification: &notification,
				TransMission: "", // 透传消息内容，与notification 二选一
			}},
		},
	}

	// 3.执行单推
	singleResult, err := single.PushSingleByCid(ctxLocal, confLocal, tokenLocal, &singleParam)
	if err != nil {
		log.Fatalln("error:", err)
	}
	log.Println("result:", singleResult)
}


```

## 开发前读一读这里
推送都包括：推送参数，推送方法，推送返回结果,主要功能都在推送参数上面。
下面以单推为例子详细介绍（其他推送每个参数和方法里面都有详细注释）：

```
    singleParam := single.PushSingleParam{
        RequestId:   strconv.FormatInt(time.Now().UnixNano(), 10), // 请求唯一标识号
        Audience:    &Audience{},       // 推送的目标用户
        Settings:    &Settings{},       // 推送条件：例如速度，定时推，厂商下发策略等
        PushMessage: &PushMessage{},    // 个推通道消息内容
        PushChannel: &PushChannel{      // 厂商通道消息内容
            Ios:     &IosChannel{},
            Android: &AndroidChannel{},
        },
    }
```
推送消息内容分为两种：个推通道消息内容（PushMessage）和厂商通道消息内容（PushChannel）

## 个推通道消息内容（PushMessage）

PushMessage中的notification（个推消息），transmission（纯透消息），revoke（回退个推消息）三选一

notification：个推消息，仅支持安卓系统，iOS系统不展示个推通道下发的通知消息

transmission：纯透传消息内容，安卓和iOS均支持。

纯透传消息是啥？
透传消息是指消息传递到客户端只有消息内容，展现的形式由客户端自行定义。客户端可自定义通知的展现形式，可以自定义通知到达后的动作，或者不做任何展现

revoke：撤回消息时使用(仅支持撤回个推通道消息，消息撤回请勿填写策略参数)

**总结一下：个推消息通道，普通消息推送仅支持安卓系统，ios不行。纯透传消息不分系统，都可以。**

## 厂商通道消息内容（PushChannel）

先说ios，ios的推送用官方的[APNS](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/PayloadKeyReference.html)

ios想要用纯透模板，就不需要PushChannel的IosChannel参数，只需要填写PushMessage的Transmission

```
// ios厂商通道消息
type IosChannel struct {
    Type           string        `json:"type,omitempty"`             // 非必须，默认值：notify，voip：voip语音推送，notify：apns通知消息
    Aps            *Aps          `json:"aps,omitempty"`              // 推送通知消息内容
    AutoBadge      string        `json:"auto_badge,omitempty"`       // 非必须，用于计算icon上显示的数字，还可以实现显示数字的自动增减，如“+1”、 “-1”、 “1” 等，计算结果将覆盖badge
    PayLoad        string        `json:"payload,omitempty"`          // 非必须，增加自定义的数据
    Multimedia     *[]Multimedia `json:"multimedia,omitempty"`       // 非必须，该字段为Array类型,设置多媒体
    ApnsCollapseId string        `json:"apns-collapse-id,omitempty"` // 非必须，使用相同的apns-collapse-id可以覆盖之前的消息
}
// 推送通知消息内容
type Aps struct {
    Alert            *Alert `json:"alert"` // 非必须，通知消息
    ContentAvailable int    `json:"content-available"`
    /*
     * content-available非必须，默认值：0
     * 0表示普通通知消息(默认为0)
     * 1表示静默推送(无通知栏消息)，静默推送时不需要填写其他参数
     * 苹果建议1小时最多推送3条静默消息
     */
    Sound    string `json:"sound,omitempty"`     // 非必须，通知铃声文件名，如果铃声文件未找到，响铃为系统默认铃声。 无声设置为“com.gexin.ios.silence”或不填
    Category string `json:"category,omitempty"`  // 非必须，在客户端通知栏触发特定的action和button显示
    ThreadId string `json:"thread-id,omitempty"` // 非必须，ios的远程通知通过该属性对通知进行分组，仅支持iOS 12.0以上版本
}
```
按照结构填写就行了，但是ios有坑，在厂商下发策略上，也就是Settings结构体中的Strategy
```
// 厂商通道策略
type Strategy struct {
    Default int `json:"default,omitempty"`
    /*
     * default字段，非必须，默认值为 1
     * 默认所有通道的策略选择1-4
     * 1: 表示该消息在用户在线时推送个推通道，用户离线时推送厂商通道;
     * 2: 表示该消息只通过厂商通道策略下发，不考虑用户是否在线;
     * 3: 表示该消息只通过个推通道下发，不考虑用户是否在线；
     * 4: 表示该消息优先从厂商通道下发，若消息内容在厂商通道代发失败后会从个推通道下发。
     * 其中名称可填写: ios、st、hw、xm、vv、mz、op，
     */
    Ios int `json:"ios,omitempty"` // 非必须，ios通道策略1-4，表示含义同上，要推送ios通道，需要在个推开发者中心上传ios证书，建议填写2或4，否则可能会有消息不展示的问题
    St  int `json:"st,omitempty"`  // 非必须，通道策略1-4，表示含义同上，需要开通st厂商使用该通道推送消息
    Hw  int `json:"hw,omitempty"`  // 非必须，通道策略1-4，表示含义同上
    Xm  int `json:"xm,omitempty"`  // 非必须，通道策略1-4，表示含义同上
    Vv  int `json:"vv,omitempty"`  // 非必须，通道策略1-4，表示含义同上
    Mz  int `json:"mz,omitempty"`  // 非必须，通道策略1-4，表示含义同上
    Op  int `json:"op,omitempty"`  // 非必须，通道策略1-4，表示含义同上
}
```
Ios建议填写2或者4

总结一下：ios推送，同时配置了厂商通道和个推纯透模板，选择通道策略为4，则APNS没有成功，就走个推的纯透消息


安卓厂商通道比较简单，通知和纯透都有，二选一；并且厂商通道和个推通道可以同时设置，然后再设置厂商通道策略，把各个厂商的通道值都设为4：表示该消息优先从厂商通道下发，若消息内容在厂商通道代发失败后会从个推通道下发。
也可以不要PushChannel的AndroidChannel厂商通道，只设置个推的安卓推送，也就是PushMessage的Notification或者Transmission

## 问题反馈
如果有疑问可以写issues，我会快速反馈



