
# Welcome
```
def ip2int(ip_addr):
    # ip_addr: 192.168.1.1
    ipint = 0 
    i = 3 
    ipNums = str(ip_addr).split('.')
    for ipNum in ipNums:
        ipint += int(ipNum) * (256 ** i) 
        i -= 1
    return ipint 
    

def ip_segment(ip_with_mask):
    # 192.168.1.1/24
    ip_splited = str(ip_with_mask).split('/')
    ipNum = ip2int(ip_splited[0])
    mask = int(ip_splited[1])
    begin = int2ip(ip_num >> (32-mask)) << (32-mask)
    end_num = ip_num >> (32-mask) << (32-num)
    end = int2ip( end_num & (2**32-1))
    return (begin, end)
    
 
def ip_sort(ip_list):
    ip_dict = {}
    for ip in ip_list:
        tuple_segment = ip_segment(ip)
        if tuple_segment in ip_dict:
            ip_dict[tuple_segment] += 1
        else
            ip_dict[tuple_segment] = 1
    return ip_dict


def ip_group(ip_list, umask):
    for ip in ip_list:
        yield ip2num(ip) >> umask << umask

def ip_sort2(ip_list):
    counter = {}
    for c_ip in ip_group(ip_list, 8)
        handler = counter.setdefault(num2ip(c_ip), 0)
        handler += 1
    return [x for x in sorted(counter.items(), key=lambda y: y[1])]
```


[参考](http://www.cnblogs.com/livingintruth/archive/2012/12/29/2839187.html)
