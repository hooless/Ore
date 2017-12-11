### prometheus

####  简述
#### 优势 
#### 选型
1. 监控组选型
2. 讨论 metric 使用方式，查询语句也很灵活
- metric 的使用方式（大型运维的需求，五个点的数据的存储和查询，metric 这写标准）
- 灵活
    聚合: sum by, topk ,increase , rate , min, max ,quantile , avg

- 不关注 qps，io吞吐型的服务; 关注相应时间、错误code、流量(qps)
- 七牛为什么快，http-rpc-长连接，内部sdk有本地缓存(domain->bucket->account->conf->rs->mongo...->bd; localcache->rsmemcache->bd)

#### 时间序列数据库
按时间查就比较快，已经排查好的时间，指定时间就是快
#### SQL [query](https://prometheus.io/docs/prometheus/latest/querying/basics/)
####  logexport
原理：实现了 tailf , 设计雏形
模式：
- dir ：追最新的log ，eof 等待检查有没有新文件，没有就等log ，可配置 500ms, 服务log , 滚动生成新文件名
- file : 只追一个文件，活跃日志的名字不变的，eof 检查 inode 是不是有变化，重新 open log ，冲头开读文件

log 解析：
- 审计log ：以 tab 分割，每一段的含义都是固定的，code 、api、resptime、reqContentLength、respContentLength、特殊段[没有为i空]
```
service_response_code
service_response_length
service_request_length
service_response_duration_ms
service_xwarn_count
```
- nginx log
```
service_response_code
service_response_length
service_request_length
service_response_duration_ms
nginx_response_duration_loc
```
- prometheus sdk 
[sdk](https://github.com/prometheus/client_golang)
parser -> sdk -> http -> pushgateway(prometheus pull) -> prometheus 时许数据库
- 
```
func getResponseCounterVec(logtype string, constLabels map[string]string) *prometheus.CounterVec {
	return prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name:        logtype + "_response_code",
			Help:        logtype + " response code",
			ConstLabels: constLabels,
		},
		[]string{"tag", "api", "method", "code"},
	)
}

func getResponseDurationVec(logtype string, constLabels map[string]string) *prometheus.HistogramVec {
	buckets := Buckets

	if strings.ToLower(logtype) == "nginx" {
		buckets = NginxBuckets
	}
	return prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:        logtype + "_response_duration_ms",
			Help:        logtype + " response duration ms",
			Buckets:     buckets,
			ConstLabels: constLabels,
		},
		[]string{"tag", "api", "method", "code", "reqlength", "resplength"},
	)
}

var constLabels = map[string]string{"host": Hostname, "idc": r.Idc, "service": r.Service, "team": r.Team}
```
- 记录
```
service_response_code{api="io.fetch",code="478",exported_job="logexporter",host="nb1983",idc="nb",instance="192.168.54.57:39092",job="pushgateway_4",service="IO",sname="io3",tag="mirror",team="kodo"}	3577

service_response_duration_ms_bucket{api="io.upload",code="200",exported_job="logexporter",host="nb2166",idc="nb",instance="192.168.54.57:19092",job="pushgateway_1",le="500",reqlength="4194304_",resplength="0_65536",service="IO",sname="io",tag="callback",team="kodo"}	0
```

配置文件的动态加载：
fsnotify : watch  dir 
```
		"name": "{{.dir}}",
		"mode": "dir",
		"path": "/home/qboxserver/{{.dir}}/_package/run/auditlog/{{.service}}",
		"type": "{{ .service | ToUpper }}",
		"log_type": "REQ",
		"reqlength_cnt_enabled": true,
		"resplength_cnt_enabled": true,
		"x-warn_cnt_enabled": true,
		"x-warns": ["BDERROR"],
		"resp_duration_enabled": true,
		"size_buckets": [0, 65536,131072, 262144, 524288, 1048576, 4194304]
                            64k, 128k, 256k, 512k, 1m, 4m
	}
```

#### metric 
#### counter
#### histogram

#### 评估需求，数据关心，运维友好，使用方便，响应时间、BI 没人了, 就是快30s

#### 业务侧
- 监控迁移
```
idc api code trigger level tags
all cfgg 597 >0 warning [KODO]
all cfgg 597 >1000 high [KODO]
all cfgg 5xx >0 warning [KODO]
all cfgg 5xx >1000 high [KODO]
bc cfgg 2xx >100 warning [KODO]
nb cfgg 595 >100 warning [TS]

idc api code trigger(ms) level tags interval(5m/15m/30m/1h/6h/1d)
all up.upload(reqlength) 2xx >260 warning [KODO] 15m
all up.upload(reqlength) 2xx >500 high [KODO] 15m
all up.mkfile(reqlength) 2xx >1000 warning [KODO] 15m
all up.mkfile(reqlength) 2xx >1500 high [KODO] 15m
all up.bput(reqlength) 2xx >50 warning [KODO] 15m
all up.bput(reqlength) 2xx >80 high [KODO] 15m
all up.mkblk(reqlength) 2xx >50 warning [KODO] 15m
all up.mkblk(reqlength) 2xx >80 high [KODO] 15m
```
- 自动生成
```
            alert="""ALERT %(team)s_interval_%(time)s_%%(idc)s_%(service)s_%(api_alertx)s_%(codex_alert)s
     IF  %(cmdline)s  %(triggerx)s
     FOR 5s
     LABELS {
       severity = "%(level)s"
     }
     ANNOTATIONS {
       summary = ' %(api_type)s ',
       description = '%(levelx)s %(tags)s %%(idc)s %(api)s %(codex)s %(flag)s: {{$value}} %(trigger)s %(unit)s ',
       link = 'http://gateway.%%(idc)s.prometheus.qiniu.io/graph#%(queryline)s',
       color = ' %(color)s'
    } """ % {'team':team, 'time':time,'service':service, 'api_alertx':api_alertx , 'codex_alert':codex_alert , 'cmdline':cmdline, 'triggerx':triggerx ,'trigger':trigger , 'level':le
vel , 'api_type':api_type, 'levelx':levelx, 'tags':tags,'api':api , 'codex':codex ,'flag':flag, 'unit':unit ,'queryline':queryline, 'color':color}
```
- 自己写规则
```
pfdstg.code.percent;pfdstg_5xx-57x;all;y;5m;critical;[KODO];(sum(rate(service_response_code{service="PFDSTG",code=~"5[0-6|8-9]."}[5m]))/sum(rate(service_response_code{service="PFDSTG"}[5m])) )*100; >0.5
pfdstg.code.number;pfdstg_597;all;y;5m;critical;[KODO];sum(rate(service_response_code{service="PFDSTG",code="597"}[5m])); >0
dc.code.percent;dc_5xx-57x;all;y;5m;warning;[KODO];(sum(rate(service_response_code{service="DC",code=~"5[0-6|8-9]."}[5m]))/sum(rate(service_response_code{service="DC"}[5m])) )*100; >0.01

def get_alert(row):
    urlprefix = '[{"range_input":"1h","end_input":"","step_input":"","stacked":"","expr":"'
    urlsuffix = '","tab":1}]'
    urlline = urlprefix + row['query'].replace('"','\\"') + urlsuffix
    queryline  = urllib.quote(urlline,'(|)|~').replace('%','%%')
    alertnamex = row['alertname'].replace('-','_')
    severityx = '* * *' + row['severity'].upper()
    if row['severity'] == 'critical':
        color = 'error'
    else:
        color = row['severity']


    alert="""ALERT kodo_%(interval)s_%%(idc)s_%(alertnamex)s
     IF  %(query)s  %(trigger)s
     FOR 10s
     LABELS {
       severity = "%(severity)s"
     }
     ANNOTATIONS {
       summary = ' %(summary)s',
       description = '%(severityx)s %(summary)s %(tags)s %%(idc)s %(alertname)s : value={{$value}} %(trigger)s',
       link = 'http://%(ifgateway)s%%(idc)s.prometheus.qiniu.io/graph#%(queryline)s',
       color = '%(color)s'
    }""" % { 'interval':row['interval'], 'alertname':row['alertname'], 'alertnamex':alertnamex, 'query':row['query'],'trigger':row['trigger'],'severity':row['severity'],'severityx':
severityx,'summary':row['summary'],'tags':row['tags'],'ifgateway':row['ifgateway'],'queryline':queryline , 'color':color }
    return alert
```

#### grafana 
- code 
- 响应时间
- 自定义metric ，打数据
- 好看、灵活、可定制、同时支持多种数据来源

#### prometheus 数据
- 机器扩展 HA， 两份数据
- 磁盘扩展 磁盘做 raid1/raid10, storge的高可用
- 请求数据，多次少量
- grafana 支持 promethues 的数据源，统筹各个平台的数据。
- 索引持久化
- 1.8之后 storage 支持 remote  read & write , 高可用可以根据rds 自己的情况来做

#### PromQL
- metric_name{labal="xx"}
- 常见函数 :sum by, topk ,increase , rate , min, max ,quantile , avg

#####  优势
-  go 写的，效率高，部署方便、快捷
-  官方写了一个高校的 time series database 
-  官方支持力度很大，开发者很活跃
-  社区发展的很良好，有很多的 plugin 和 export 
-  Grafana 原生支持
-  各种 server 的discovery 
-  不仅仅是个 monitor tool 
-  k8s 结合的很好
-  生态也变得很好了

#### rule 规则
- prometheus 配置rule 的文件路径 (2.0 变yaml)
- prometheus 根据 rule 模块触发告警，将告警发给 alltermange
- labels :  alter 会 join 这里定义的label ，alter 里面的labels 一样就是同一个 alter  , (lebebs,anotation,star,end,url)
- annotation: 描述、额外信息、自定义动态信息,可以引用 alter 里面的数据做为变量,但是在这个 alter 里面 join 的 label 无法使用
- rule 发送给 altermanage 的是 http 以 json 的形式发给  altermange

#### altermange
- 告警通知平台: alter 分组、 去噪(剔除掉,两条很相似，告警级别的控制)、沉默(隔多久不显示)
- prometheus 的自带的 rules 模块，能够通过定义的规则产生相应的告警信息，并将告警推送到 altermnage 
- 接收 alter （json) ， 一次接收很多 json 的数组 ，发送消息
    - 接收数据，通知到一个渠道
    - 不通渠道，配置成不同的模版
    - 根据模版渲染成不同的告警信息,发送出去
- 配置分组
    - route  来发送路由
    - route 下面套路由 ->  reciver
    - group_by 根据 altername 来分组，进行组管理（频率、等待时间,波动 30s 又有消息来就不发了）
    - matche_re -> repeat_interval -> recvier
    - group_wait : 等待发送，可能恢复
    - group_interval: 5m ，如果老得没有恢复，继续命中，等待这么久再发
- inhibiy_rules:


#### lingxing
