# 如何用 Github 轻松拉取谷歌容器镜像

在 k8s 的深度实践中，我们有时需要拉取谷歌容器镜像，域名诸如 **gcr.io**，**k8s.gcr.io**。但是由于网络的一些限制和成本的一些考量，做起来比较棘手。国内的一些镜像加速，往往并不能提供持续的免费服务或者同步镜像的版本比较旧。

本文以拉取 [k8s nginx ingress controller](https://github.com/kubernetes/ingress-nginx) 容器镜像为例，来谈谈如何借助 **Github Actions + Github** 容器镜像服务来拉取谷歌镜像。

## 在 Github 中生成访问 Github Packages Registry 的 token 

在 Github 个人主页里点击 **Settings --- Developer settings --- Personal access tokens --- New personal access token** 并勾选 **write:packages，read:packages，delete:packages** 来让这个 token 具有 Pacakgs 管理权限，当然也包括 **ghcr.io** 的容器镜像管理权限。记录下 token，以便在 repository 的 secret 和 docker login 的命令行使用。

Github 推出的容器镜像服务，域名为 **ghcr.io(Github Container Registry)**。仓库的命名空间就是 github 的用户名，在 Github 个人主页里的 Packages 即可看到所有上传的 docker 镜像列表。也就是说通过命令行 `docker push ghcr.io/lanzhiwang/镜像名` 可以将镜像推送到 Package 中。

但这个 Packages 不仅仅局限于 docker 制品，可以是 npm，Maven 等，具体可以参考官方文档。

命令行操作 ghcr.io 如下：

```bash
$ echo <personal access token> | docker login ghcr.io --username lanzhiwang --password-stdin

$ docker tag app ghcr.io/lanzhiwang/app:1.0.0

$ docker push ghcr.io/lanzhiwang/app:1.0.0

```

默认上传到 Packages 后的镜像是私有的，仅个人能看到和下载。可以通过 **Package Settings --- Make public** 来设置为公共，就可以免密码拉取了。

还可以把上传的 image 关联到代码项目的 README 文件，这样逻辑就更清楚了，这个镜像是某个项目的制品或者某个项目要用到这个镜像。

## Github Action 拉取镜像并上传

新建一个 repository，在项目里通过 **Settings --- Secrets --- New secret** 把之前生成的personal access token 存放到环境变量 GHCRIOTOKEN，以便在 Actions 的 pipeline 文件里引用。

在这个项目的 Actions 里配置 **.github/workflow/main.yaml** 文件 ，Actions workflow 会在Github 中动态生成的 Ubuntu 构建环境中，拉取谷歌镜像，并上传至 Github 镜像仓库。

main.yaml 文件如下：

```yaml
name: GitGoogleContainer
on:
  push:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # - uses: actions/checkout@v2
      - name: Login in Docker Registry
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GHCRIOTOKEN }}
      - name: Docker pull and push
        run: |
          docker pull k8s.gcr.io/ingress-nginx/controller:v0.35.0  
          docker tag  k8s.gcr.io/ingress-nginx/controller:v0.35.0  ghcr.io/lanzhiwang/ingress-nginx/controller:v0.35.0       
          docker push ghcr.io/lanzhiwang/ingress-nginx/controller:v0.35.0
```

当 Actions 流程跑完以后，我们就可以在 Github Packages 中看到刚才从谷歌拉取的镜像。这样我们再在项目里拉取镜像的时候就可以直接从Github拉取了。

