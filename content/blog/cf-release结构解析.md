+++
id= "319"

title = "cf-release结构解析"
description = "1.制作时的cf-release结构解析此处指的release统一为CloudFoundry官方给出的cf-release，不做修改。1.1. 通过载入cf-release文件夹下config/final.yml文件，获得需要下载release文件的远程服务器网址，默认使用的提供商是s3，地址是：blob.cfblob.com "
tags= [ "Bosh" , "cf-release" , "cloudfoundry" ]
date= "2014-12-17 16:03:01"
author = "孙建波"
banner= "img/blogs/319/cf1.jpg"
categories = [ "cloudfoundry" ]

+++



1\. 制作时的cf-release结构解析
----------------------

此处指的release统一为CloudFoundry官方给出的[cf-release](https://github.com/cloudfoundry/cf-release)，不做修改。 

1.1 通过载入cf-release文件夹下config/final.yml文件，获得需要下载release文件的远程服务器网址，默认使用的提供商是s3，地址是：blob.cfblob.com 

<!--more-->

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616363/sel/cf1_pmdlgm.jpg" alt="link" style="zoom:50%;" />
</center>



1.2 通过config/blobs.yml，可以得到所有blobs的object\_id，通过服务器地址+object\_id拼接的字符串即可下载到相对应的blob内容。 

1.3 默认存储的位置为cf-release/.blobs，存储的文件名为sha1值，下载完成后会在cf-release/blobs文件夹下创建以package真实名字命名的软链接到.blobs里面各个具体的包。  

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616363/sel/cf2_xcc1uv.jpg" alt="install" style="zoom:50%;" />
</center>



1.4 下载完所有的blobs后，开始对照cf-release/packages文件夹下各个包的spec文件逐个在blobs文件夹下找到，然后拷贝到.final\_builds或者.dev\_builds，根据是否加了--final参数决定。拷贝前会执行预安装脚本prepackaging，检查文件是否都存在，做一些单元测试等。执行完后把prepackaging脚本删除后压缩文件夹。 (**TIPS**：有时候某些不需要部署的组件，却因为过不了prepacking脚本的执行导致release做不出来，可以把prepackaging脚本删掉再制作，会自动跳过这个执行过程。) 

1.5 对所有cf-release/jobs进行的操作相对简单，除了拷贝到.final\_builds或者.dev\_builds以外，通过spec文件检查template等文件是否齐全。

1.6 最后生成releases/cf-#{version}.yml文件,在dev\_releases文件夹下生成cf-{version}.dev.yml release就算初步制作完成了。

2\. 部署时的cf-release结构解析
----------------------
<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616366/sel/cf3_mdkfp5.jpg" alt="解析过程" style="zoom:45%;" />
</center>



2.1. 获得cf-release的配置文件： 扫描./releases以及./dev\_releases文件夹，对其中的release配置文件进行排序，排序规则为数字大的优先，相同大小的数字以小数点后大的优先，两个数字都相同取没有dev标记的。 194 > 193 194.1 > 194 194.1 > 194.1-dev 这里得到的最新的文件，就是定义当前release包所有版本的配置文件，称之为@release。 

2.2. 获取部署配置文件manifest/cf.yml中，要部署的job构成的所有template。部署时定义的job在配置文件中包含多个template，每个template由多个package组成。

```yaml
---
deployment: cf
jobs:
  - name: nats
    template:
      - nats
      - nats_stream_forwarder
  - name: nfs_server
    template:
      - debian_nfs_server
```


2.3. 对于2.2中找出的每个template，找到其在@release文件中的version编号以及sha1值（jobs属性下），然后找到.final\_builds/jobs下对应的index.yml和.dev\_builds/jobs下对应的index.yml，比对两个文件中的sha1，找到对应的版本。此时我们就获得了template的全部具体信息，称之为@template。 

2.4. @template下有个压缩包，后缀为.tgz，解压缩后得到job.MF文件，可获得该template的所有配置文件，配置文件需要的属性以及依赖的packages。也就是这里，我们获得了构成这个template的所有packages名字。然后我们对照之前的@release文件，又可以得到具体每个package需要的版本。 

2.5. 值得注意的是，每个template由一个或多个packages构成，而每个package，由零个或多个其他packages构成，而每个package依赖哪些其它package，也在@release文件中的packages栏目下。 

2.6. 通过类似的方法，我们在.final\_builds和.dev\_builds中的packages对应的package中可以对比出具体的package版本信息，找到需要部署的包，我们命名为@package。 至此，部署所需要的cf-release结构就已经全部解析出来了。

3\. 部署
------

3.1. 默认的部署目录为/var/vcap，部署之前会在该目录下创建目录bosh,data,jobs,monit,packages,shared,store,sys这几个目录。 

3.2. 真实的部署数据都存放在/var/vcap/data目录下，该目录下的jobs、packages以及sys分别在/var/vcap建立软连接到jobs、packages和sys，部署前会检查packages文件下的具体某个package文件夹下的.version文件，如果内容相同则表示已经部署了该package。同理，部署完后会生成.version文件表示已经部署完该package。 

3.3. 当目录创建完毕后，就解压缩我们之前找到@package，里面必然包含了一个名为packaging的shell脚本，这个脚本里包含了所有部署需要运行的过程，我们需要做的就是执行这个脚本，就把最基本的环境安装起来了。 

<center>
<img src="https://res.cloudinary.com/rachel725/image/upload/v1605616365/sel/cf4_hbjpag.jpg" alt="部署" style="zoom:40%;" />
</center>

