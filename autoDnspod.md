### Dnspod API 切换一个域名

#### 1. [修改记录](http://www.dnspod.cn/docs/records.html#record-modify)
`login_token`获取: 登陆 ->  用户中心  -> 安全设置  -> API Token 。获取得到的 `ID:<id>` ，`Token:<token>` 格式的数据， login_token 在 API 中传入的格式为 `<id>,<token>`
```
官方范例: 
curl -X POST https://dnsapi.cn/Record.Modify -d 'login_token=<login_token>&format=json&domain_id=<domain_id>&record_id=<record_id>&sub_domain=<sub_domain>&value=<value>&record_type=<record_type>&record_line_id=<record_line_id>'
```
非自定义的参数(需获取): `login_token` , `domain_id` , `record_id` , `record_line_id`。    
```
# python 定义一个函数 demo
import request

record_modify_url ="https://dnsapi.cn/Record.Modify"
def modify_domain_value(login_token,domain_id,sub_domain,value,record_id,record_line_id,record_type):
    record_modify_data = {
            'login_token' : login_token , 
            'format' : 'json' ,
            'domain_id' : domain_id ,
            'sub_domain' : sub_domain,
            'value' : value
            'record_id' : record_id,
            'record_line_id' : record_line_id,
            'record_type' : record_type,
    }

    record_modify_r  = requests.post(record_modify_url, data = record_modify_data)
    records_modify  = record_modify_r.json()['record']
    print records_modify
modify_domain_value(login_token,domain_id,'www','2.2.2.2',record_id,record_line_id,'A')  #自定义参数与非自定义参数的传入
```
#### 2. [获取域名列表](http://www.dnspod.cn/docs/domains.html#domain-list)
```
官方范例: 
curl 'https://dnsapi.cn/Domain.List' -d 'login_token=<login_token>&format=json'  
```
获取修改域名需要的 `domain_id`
```
import requests

# python 定义一个函数 demo
domain_list_url ="https://dnsapi.cn/Domain.List"
def get_domain_id(login_token,domain_name):
    domain_list_data = {
            'login_token' : login_token,
            'format' : 'json'
    }
    domain_list_r  = requests.post(domain_list_url, data = domain_list_data)
    domains_list  = domain_list_r.json()['domains']
    domain_id = ''
    for domain in domains_list:    
        if domain['name'] == domain_name:    # 一个用户管理下有很多 domain ，获取要修改的 domain 的id
            domain_id =  domain['id']
    return domain_id

domain_id = get_domain_id(login_token_str,'hooless.com') #自定义参数与非自定义参数的传入
print domain_id
```

### 3. [记录列表](http://www.dnspod.cn/docs/records.html#record-info)
```
官方范例: 
curl 'https://dnsapi.cn/Record.List' -d 'login_token=<login_token>&format=json&domain_id=<domain_id>&sub_domain=<sub_domain>’
```
获取 `record_id` , `record_line_id `
```
# python 定义一个函数 demo
import requests

record_list_url ="https://dnsapi.cn/Record.List"
def get_record_info(login_token,domain_id,sub_domain):
    record_list_data = {
            'login_token' : login_token, 
            'format' : 'json' ,
            'domain_id' : domain_id ,
            'sub_domain' : sub_domain
    }
    record_list_r  = requests.post(record_list_url, data = record_list_data)
    records_list  = record_list_r.json()['records']
    record_id = ''
    for record in records_list:
        if record['name'] == sub_domain:
            record_id =  record['id']
            record_line_id = record['line_id']
    return record_id,record_line_id

record_id,record_line_id = get_record_info(login_token_str,domain_id,'www') #自定义参数与非自定义参数的传入
print record_id,record_line_id
```
