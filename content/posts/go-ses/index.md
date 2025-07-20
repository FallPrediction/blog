+++
title = 'Go Ses Example'
date = 2025-07-20T12:42:22+08:00
draft = false
description = '一個簡單的 GO AES SES 寄信範例'
toc = true
tags = ['AWS']
categories = ['AWS']
+++

因為 AWS 官方文件沒有 GO 的範例，記錄一下測過的版本

main.go
```go
package main

import "fmt"

func main() {
	subject := "這是測試信件"
	bodyText := "這是測試信件的純文字內文"
	bodyHtml := "<p>這是測試信件的內文</p>"
	sender := "noreply@fall-prediction.net"
	recipients := []string{"lizne6z0@gmail.com"}

	ses := NewSes("ap-south-1")
	messageId, err := ses.SendEmail(subject, bodyText, bodyHtml, sender, recipients)

	if err != nil {
		fmt.Println("The email was not sent. Error message: ", err)
	} else {
		fmt.Println("Email sent! Message ID: " + messageId)
	}
}
```

ses.go
```go
package main

import (
	"context"
	"sync"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/sesv2"
	"github.com/aws/aws-sdk-go-v2/service/sesv2/types"
)

var ses *Ses
var sesOnce sync.Once

type Ses struct {
	client *sesv2.Client
}

func (s *Ses) SendEmail(subject, bodyText, bodyHtml string, sender string, recipients []string) (string, error) {
	emailInput := s.getEmailInput(subject, bodyText, bodyHtml, sender, recipients)
	output, err := s.client.SendEmail(context.Background(), &emailInput)
	return *output.MessageId, err
}

func (s *Ses) getEmailInput(subject, bodyText, bodyHtml string, sender string, recipients []string) sesv2.SendEmailInput {
	charset := "UTF-8"
	return sesv2.SendEmailInput{
		FromEmailAddress: &sender,
		Content: &types.EmailContent{
			Simple: &types.Message{
				Body: &types.Body{
					Html: &types.Content{
						Data:    &bodyHtml,
						Charset: &charset,
					},
					Text: &types.Content{
						Data:    &bodyText,
						Charset: &charset,
					},
				},
				Subject: &types.Content{
					Data:    &subject,
					Charset: &charset,
				},
			},
		},
		Destination: &types.Destination{
			ToAddresses: recipients,
		},
	}
}

func NewSes(region string) *Ses {
	if ses == nil {
		sesOnce.Do(func() {
			cfg, err := config.LoadDefaultConfig(context.TODO(), config.WithRegion(region))
			if err != nil {
				panic("unable to load SDK config" + err.Error())
			}
			sesClient := sesv2.NewFromConfig(cfg)
			ses = &Ses{sesClient}
		})
	}
	return ses
}
```

另：[官方文件](https://docs.aws.amazon.com/zh_tw/ses/latest/dg/manage-sending-quotas.html)建議為每個收件人個別呼叫 SendEmail 一次
