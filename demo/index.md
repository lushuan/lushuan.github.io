# Demo 图像和 shortcode

## 图片测试

测试1 直接引用

![dog](/images/dog2.jpg)

xxx

测试2 使用html 标签图片会根据比率缩放

<img src="/images/dog.jpg" width="70%" align="center" />

测试5 图片会默认左对齐

<img src="/images/dog.jpg" width="50%" align="center" />

测试6 图片会居中

<center><img src="/images/dog.jpg" width="50%" /></center>

测试6 显示图注，当文章的图片比较多的时候我们经常需要使用图注，当然我们可以直接这样

<center><img src="/images/dog.jpg" width="50%" /></center>
<center>可爱的金毛</center>

测试7 更优雅写法的图注

<center>{{< figure src="/images/dog.jpg" width="50%" title="可爱的金毛-优雅" >}}</center>


## 短代码片 shortcode
下面两种成对的短代码片

### markdown 引入
```shell
echo "kuberntes"
sudo kubectl get sa -A
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

### 通过highlight 引入，并指定语言
不指定语言会报错,随便指定语言会不显示
{{< highlight demo >}}
cat << EOF >> /etc/hosts
199.232.68.133 raw.githubusercontent.com
199.232.68.133 user-images.githubusercontent.com
199.232.68.133 avatars2.githubusercontent.com
199.232.68.133 avatars1.githubusercontent.com
EOF
{{< /highlight >}}

指定 bash 语言
{{< highlight bash >}}
cat << EOF >> /etc/hosts
199.232.68.133 raw.githubusercontent.com
199.232.68.133 user-images.githubusercontent.com
199.232.68.133 avatars2.githubusercontent.com
199.232.68.133 avatars1.githubusercontent.com
EOF
{{< /highlight >}}

指定 go-html-template 语言
{{< highlight go-html-template "lineNos=inline, lineNoStart=42" >}}
{{ range .Pages }}
  <h2><a href="{{ .RelPermalink }}">{{ .LinkTitle }}</a></h2>
{{ end }}
{{< /highlight >}}

指定 python 语言
{{< highlight python >}}
print("hello world")
{{< /highlight >}}


