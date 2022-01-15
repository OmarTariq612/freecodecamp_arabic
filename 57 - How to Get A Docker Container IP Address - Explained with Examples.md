# شرح الوصول لعنوان الـ IP الخاص بـ Docker Container مع بعض الأمثلة

![Marcelo Costa](https://www.freecodecamp.org/news/content/images/size/w100/2020/06/mmiranda.jpg)

[Marcelo Costa](https://www.freecodecamp.org/news/author/mesmacosta/)

![How to Get A Docker Container IP Address - Explained with Examples](https://cdn-media-2.freecodecamp.org/w1280/5f9c9a19740569d1a4ca2384.jpg)

يمنحنا Docker القدرة على تشغيل التطبيقات في بيئات شبه معزولة تُدعى Conainers.

أعلم ما تفكر فيه (مقال آخر يتحدث عن Docker) إنه في كل مكان هذه الأيام! 

![](https://www.freecodecamp.org/news/content/images/2020/06/docker-i-see.jpg)

ولكن لا داعي للقلق، سنتطرق الى ما هو أبعد من تلك المقدمة الابتدائية المنتشرة عن Docker. حيث أن الفئة المستهدفة من هذا المقال هم مَن يعرفون بالفعل المفاهيم الأساسية وراء Docker و الـ Containers. 

هل تساءلت يومًا عن كيفية الوصول لعنوان الـ IP الخاص بـ Container ؟

## شرح Docker Network

  
بدءا، لنفهم طريقة عمل الـ Docker Network. لذلك سنركز على الـ `bridge`  network الافتراضية. هذا النوع من الشبكات الذي يستخدم في حال عدم تحديدك لنوع driver أخر. 

![](https://www.freecodecamp.org/news/content/images/2020/06/docker-network.png)

Docker network from:  [understanding-docker-networking-drivers-use-cases](https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/)

تعمل الـ `bridge` network كشبكة خاصة (private network) داخل الجهاز المضيف (host) حيث يمكن للـ containers التواصل مع بعضهم البعض. ولكن للسماح بالوصول الخارجي للـ containers (من خارج الـ private network) يجب فتح ports تشير إلى الـ containers المراد الوصول لها خارجيًا. 

تستخدم الشبكات من النوع `bridge` في حالة تشغيل التطبيقات في containers منفصلة قائمة بذاتها في حاجة للتواصل مع بعضها البعض .

في الصورة أعلاه يمكن لـ `dp` و `web` التواصل معًا من خلال شبكة من النوع `bridge` معرفة من قِبَل المستخدِم `mybridge`.

إذا لم يسبق لك إضافة شبكة (network) في Docker سترى شيء شبيه بذلك عند كتابة الأمر:

```bash
$ docker network ls

NETWORK ID          NAME                  DRIVER              SCOPE
c3cd46f397ce        bridge                bridge              local
ad4e4c24568e        host                  host                local
1c69593fc6ac        none                  null                local
```

الشبكة الافتراضية من النوع `bridge` ومعها نوعين آخرين `host` و `none`. سنتجاهل هذين النوعين و سنركز على الشبكة من النوع `bridge`  عندما نتطرق إلى الأمثلة.

## عنوان الـ IP الخاص بـ Docker Container

  
عندما يتصل container بـ Docker Network فإن هذه الشبكة تقوم بتعيين IP خاص بهذا الـ container. فعند إنشاء أي شبكة يتم تعيين  `subnet mask` خاص بها يمكّنها من إعطاء عناوين IP.

عادةً يستخدم Docker بشكل افتراضي **172.17.0.0/16** كـ subnet لـ container networking. 

لفهم الأمر جيدًا سنطرح مثال عملي.

![drawing](https://www.freecodecamp.org/news/content/images/2020/06/flamenco-done.png)

### مثال

  
لتوضيح ما سبق سنستخدم بيئتي Hive و Hadoop  

ألقِ نظرة علي ملف `docker-compose.yml` والذي سنقوم بتشغيله لاحقًا:

```
version: "3"

services:
  namenode:
    image: bde2020/hadoop-namenode:2.0.0-hadoop2.7.4-java8
    volumes:
      - namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=test
    env_file:
      - ./hadoop-hive.env
    ports:
      - "50070:50070"
  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop2.7.4-java8
    volumes:
      - datanode:/hadoop/dfs/data
    env_file:
      - ./hadoop-hive.env
    environment:
      SERVICE_PRECONDITION: "namenode:50070"
    ports:
      - "50075:50075"
  hive-server:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    environment:
      HIVE_CORE_CONF_javax_jdo_option_ConnectionURL: "jdbc:postgresql://hive-metastore/metastore"
      SERVICE_PRECONDITION: "hive-metastore:9083"
    ports:
      - "10000:10000"
  hive-metastore:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    command: /opt/hive/bin/hive --service metastore
    environment:
      SERVICE_PRECONDITION: "namenode:50070 datanode:50075 hive-metastore-postgresql:5432"
    ports:
      - "9083:9083"
  hive-metastore-postgresql:
    image: bde2020/hive-metastore-postgresql:2.3.0

volumes:
  namenode:
  datanode:

```

[From  **docker-hive**  GitHub](https://github.com/mesmacosta/docker-hive)

لا أحد يريد قراءة ملف **ضخم** لذلك هذه الصورة للتوضيح:

![](https://www.freecodecamp.org/news/content/images/2020/06/Screen-Shot-2020-06-21-at-2.48.18-PM.png)

لنبدأ تشغيل الـ containers :

```bash
docker-compose up -d

```

نستطيع رؤية الـ containers وعددهم 5

```bash
$ docker ps --format \
"table {{.ID}}\t{{.Status}}\t{{.Names}}"

CONTAINER ID        STATUS                   NAMES
158741ba0339        Up 1 minutes             dockerhive_hive-metastore-postgresql
607b00c25f29        Up 1 minutes             dockerhive_namenode
2a2247e49046        Up 1 minutes             dockerhive_hive-metastore
7f653d83f5d0        Up 1 minutes (healthy)   dockerhive_hive-server
75000c343eb7        Up 1 minutes (healthy)   dockerhive_datanode
```

لنلقِ نظرة على شبكات (Docker Networks) الخاصة بنا:

```bash
$ docker network ls

NETWORK ID          NAME                  DRIVER              SCOPE
c3cd46f397ce        bridge                bridge              local
9f6bc3c15568        docker-hive_default   bridge              local
ad4e4c24568e        host                  host                local
1c69593fc6ac        none                  null                local
```

انتظر لحظة ... هناك شبكة جديدة تٌدعى `docker-hive_default` !

يقوم docker compose بإنشاء شبكة للتطبيق الخاص بك ويعطيها اسمًا بناءً على اسم المشروع المأخوذ من اسم المجلد (directory) الموجود فيه. 

وحيث أن `docker-hive` هو اسم المجلد الخاص بالمشروع ،هذا يفسر اسم الشبكة الجديدة.

سنتطرق الآن إلى كيفية الوصول لعنوان ال IP فى Docker.

## أمثلة - كيفية الوصول لعنوان الـ IP الخاص بـ Docker Container  

والآن وبعد جذب انتباهكم سيتم كشف الستار عن إجابة هذا السؤال الغامض.

![drawing](https://www.freecodecamp.org/news/content/images/2020/06/bermuda-logged-out-1.png)

### 1. باستخدام Docker Inspect

  
يمثل Docker inspect أحد أهم الطرق للوصول لمعلومات دقيقة عن الـ Docker objects. فيمكنك انتقاء أي حقل (field) من صيغة JSON.

للوصول لعنوان الـ IP الخاص بـ `dockerhive_datanode` :

```bash
$ docker inspect -f \
'{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
75000c343eb7

172.18.0.5
```

ألم تقل أن Docker يستخدم **172.17.0.0/16** كـ subnet بشكل افتراضي لـ container networking ؟ لماذا عنوان الـ IP المٌعاد : **172.18.0.5** يقع خارجها ؟

![](https://www.freecodecamp.org/news/content/images/2020/06/Screen-Shot-2020-06-22-at-3.25.07-PM.png)

Image created on  [ip-address-in-cidr-range](https://tehnoblog.org/ip-tools/ip-address-in-cidr-range/)

للإجابة على هذا السؤال يلزم إلقاء نظرة على الإعدادات الخاصة بالشبكة (network settings) :

```bash
$ docker network inspect -f \
'{{range .IPAM.Config}}{{.Subnet}}{{end}}'  9f6bc3c15568

172.18.0.0/16
```

تم تطبيق هذا المثال على توزيعة لينكس (linux distribution) تٌدعى Compute Engine VM ، وفي هذا الاختبار تم تعيين subnet مختلفة : **172.18.0.0/16** لـ شبكة (Docker Network) وهذا يفسر كل شيء.

وبدلاً من الحاجة للوصول لعنوان الـ IP الخاص بكل container بصورة فردية

يمكننا إلقاء نظرة علي جميع عناوين الـ IP الخاصة بشبكة `docker-hive_default` :

```bash
$ docker network inspect -f \
'{{json .Containers}}' 9f6bc3c15568 | \
jq '.[] | .Name + ":" + .IPv4Address'

"dockerhive_hive-metastore-postgresql:172.18.0.6/16"
"dockerhive_hive-metastore:172.18.0.2/16"
"dockerhive_namenode:172.18.0.3/16"
"dockerhive_datanode:172.18.0.5/16"
"dockerhive_hive-server:172.18.0.4/16"
```

![drawing](https://www.freecodecamp.org/news/content/images/2020/06/cherry-success.png)

فى حال لم تلاحظ فاننا استخدمنا صيغة [**jq**](https://github.com/stedolan/jq) فى تحليل الـ map object الخاصة بالـ `Containers`.

### 2. باستخدام Docker exec

فى هذا المثال سنوضح الوصول لعنوان الـ IP الخاص بـ `dockerhive_namenode` 

```bash
$ docker exec dockerhive_namenode cat /etc/hosts

127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.18.0.3      607b00c25f29
```

### 3. باستخدام bash داخل الـ container

```bash
$ docker exec -it dockerhive_namenode /bin/bash

# running inside the dockerhive_namenode container
ip -4 -o address

7: eth0    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
```

يمكننا الوصول لعناوين الـ IP الخاصة بـ containers أخرى تقع مع `dockerhive_namenode` في شبكة مشتركة `docker-hive_default`

**Data node**

```bash
# running inside the dockerhive_namenode container
ping dockerhive_datanode

PING dockerhive_datanode (172.18.0.5): 56 data bytes
64 bytes from 172.18.0.5: icmp_seq=0 ttl=64 time=0.092 ms
```

**Hive mestastore**

```bash
# running inside the dockerhive_namenode container
ping dockerhive_hive-metastore

PING dockerhive_hive-metastore_1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: icmp_seq=0 ttl=64 time=0.087 ms
```

**Hive server**

```bash
# running inside the container
ping dockerhive_hive-server

PING dockerhive_hive-server (172.18.0.4): 56 data bytes
64 bytes from 172.18.0.4: icmp_seq=0 ttl=64 time=0.172 ms
```

## **الخاتمة**

تم تطبيق جميع الأمثلة على توزيعة لينكس (linux distribution) تٌدعى Compute Engine VM. قد تختلف بعض الأوامر قليلا إذا أردت تطبيق الأمثلة على نظام تشغيل Windows أو macOS.

يجب الأخذ في عين الاعتبار أن عناوين الـ IP فى الأمثلة السابقة هى داخلية (internal) بشبكة `docker-hive_default` ،وأنه عند الحاجة للوصول لهذه الـ containers خارجيا (من خارج شبكة `docker-hive_default`) فإنه يلزم استخدام عنوان الـ IP الخارجي الخاص بالجهاز المضيف (host) (على افتراض أنك قمت بالفعل بفتح الـ ports الخاصة بالـ containers بشكل صحيح). 
  
اجعل kubernetes يتعامل مع الأمور الخاصة بعناوين الـ IP فى حال كنت تستخدم kubernetes لإدارة الـ Docker containers الخاصة بك [kubernetes-expose-external-ip-address](https://kubernetes.io/docs/tutorials/stateless-application/expose-external-ip-address/).

*** Illustrations from  [icons8.com](https://icons8.com/)  by  [Murat Kalkavan](https://dribbble.com/muratkalkavan).**