---
title: 新浪图床图片替换为腾讯COS
date: 2023-02-04 21:39:04
tags:
- golang
categories:
- golang
---

### 概述

近期，有同学反馈，所有调用的微博图床图片都无法加载并提示“403 Forbidden”了。百度上说新浪开启了防盗链，查遍了网上一堆复制/粘贴出来的文章，不是开启反向代理就是更改请求头，但是这些方案都不行。最终方案只能自己自建图床（其实自建图床的成本并不高，只是自己懒得搞，注册了个七牛云的对象存储，但是需要绑定域名，所以上次只能作罢，这次就使用腾讯COS作为图床吧~），为了批量把文档中的新浪图片替换成腾讯cos，也是花了丢时间写了个脚本用于批量替换。

```golang
package main

import (
	"context"
	"fmt"
	"io"
	"io/fs"
	"net/http"
	"net/url"
	"os"
	"path"
	"path/filepath"
	"regexp"
	"strings"

	"github.com/tencentyun/cos-go-sdk-v5"
)

const (
	sinaImageCacheDir = "/tmp"
	mdFileDir         = "xxx"
	CosSecretId       = "xxx"
	CosSecretKey      = "xxx"
	CosBucketUrl      = "xxx"
)

type mdFile struct {
	fileName       string
	filePath       string
	fileContent    string
	sinaImages     map[string]string
	downloadImages map[string]string
	cosImages      map[string]string
}

func main() {
	if err := createSinaImageCacheDir(); err != nil {
		panic(err)
	}

	mdFiles, err := listFileContent(mdFileDir)
	if err != nil {
		panic(err)
	}

	for _, file := range mdFiles {
		downloadSinaImage(file)
	}

	for _, file := range mdFiles {
		uploadTencentCos(file)
	}

	for _, file := range mdFiles {
		replaceSinaImageUrl(file)
	}
}

func createSinaImageCacheDir() error {
	_, err := os.Stat(sinaImageCacheDir)
	if err != nil {
		if os.IsNotExist(err) {
			return os.Mkdir(sinaImageCacheDir, fs.ModePerm)
		}

		return err
	}

	return nil
}

func listFileContent(dir string) ([]*mdFile, error) {
	fileName, err := filepath.Glob(fmt.Sprintf("%s/*.md", dir))
	if err != nil {
		return nil, err
	}

	mdFiles := make([]*mdFile, 0, len(fileName))
	for _, name := range fileName {
		// 获取文件列表
		_, file := path.Split(name)
		fileContent, err := os.ReadFile(name)
		if err != nil {
			continue
		}

		// 获取新浪图片
		simaImages := map[string]string{}
		sinaImages := regexp.MustCompile(`https://tva1.sinaimg.cn/large/(\w+).jpg|.jpeg|.png`).FindAllString(string(fileContent), -1)
		fmt.Printf("sinaimages")
		for _, imageUrl := range sinaImages {
			_, imageName := path.Split(imageUrl)
			simaImages[imageUrl] = imageName
		}

		mdFiles = append(mdFiles, &mdFile{
			fileName:    file,
			filePath:    name,
			fileContent: string(fileContent),
			sinaImages:  simaImages,
		})
	}

	return mdFiles, nil
}

func downloadSinaImage(file *mdFile) {
	downloadImagePaths := map[string]string{}
	for sinaImageUrl, sinaFileName := range file.sinaImages {
		resp, err := http.Get(sinaImageUrl)
		if err != nil || resp.StatusCode != http.StatusOK {
			fmt.Printf("failed get sina image, error=%v, statusCode=%v", err, resp.StatusCode)
			continue
		}

		fileName := fmt.Sprintf("%s/%s", sinaImageCacheDir, sinaFileName)
		out, err := os.Create(fileName)
		if err != nil {
			fmt.Printf("failed create file, err=%v, fileName=%v", err, fileName)
			_ = resp.Body.Close()
			continue
		}

		// 然后将响应流和文件流对接起来
		_, err = io.Copy(out, resp.Body)
		if err != nil {
			fmt.Printf("failed io.Copy, err=%v", err)
			_ = resp.Body.Close()
			continue
		}

		downloadImagePaths[sinaImageUrl] = fileName
		_ = resp.Body.Close()
	}

	file.downloadImages = downloadImagePaths
}

// 上传到腾讯cos
func uploadTencentCos(file *mdFile) {
	u, _ := url.Parse(CosBucketUrl)
	b := &cos.BaseURL{BucketURL: u}
	c := cos.NewClient(b, &http.Client{
		Transport: &cos.AuthorizationTransport{
			SecretID:  CosSecretId,
			SecretKey: CosSecretKey,
		},
	})

	cosImageUrls := map[string]string{}
	for sinaImageUrl, localFileName := range file.downloadImages {
		fd, err := os.Open(localFileName)
		if err != nil {
			continue
		}

		fileName, ok := file.sinaImages[sinaImageUrl]
		if !ok {
			continue
		}
		_, err = c.Object.Put(context.Background(), fileName, fd, nil)
		if err != nil {
			fmt.Printf("put image to cos failed, err=%v", err)
			continue
		}
		cosImageUrls[sinaImageUrl] = fmt.Sprintf("%s/%s", CosBucketUrl, fileName)
	}

	file.cosImages = cosImageUrls
}

func replaceSinaImageUrl(file *mdFile) {
	for sinaImageUrl, cosImageUrl := range file.cosImages {
		file.fileContent = strings.ReplaceAll(file.fileContent, sinaImageUrl, cosImageUrl)
	}

	if err := os.WriteFile(file.filePath, []byte(file.fileContent), 0644); err != nil {
		fmt.Printf("failed write file, err=%v", err)
	}
}
```
