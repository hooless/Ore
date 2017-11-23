###  值得玩味的 bash 代码 -  规范 老道

```
#!/bin/sh
#auther: xiaoding
#version: v1.0
#date: 2014-08-13
#name: center_monitor.sh
#function: monitoring the operation of center YUM source
#1. monitoring the nginx daemon, i.e running status, listen status, php scripts performance
#2. monitoring throughputs of center YUM souce, i.e the RX and TX, the speed in network, error or dropped packets
#3. monitoring the http, by nginx log, i.e status, remote_addr, request etc.
#4. Alert the emergency by sms

LOG_MIRROR="/var/log/nginx/mirror.log"
LOG_MIRRORLIST="/var/log/nginx/mirrorlist.log"

num_error=0
gredient=2
num_right=0

convergentSend() {
        local msg=$1; shift
        ((num_error++))
        num_right=0
        if [[ $num_error -eq $gredient ]]; then
                report_by_sms "$msg"
                num_error=0
                gredient=$((gredient*2))
        fi
}

DownConvergence() {
        local msg=$1; shift
        log "$msg"

        ((num_right++))
        if [[ $num_right -eq 100 ]]; then
                if [[ $gredient -ne 2 ]]; then
                        gredient=$((gredient/2))
                        echo $gredient
                fi
                num_right=0
		num_error=1
        fi
}

#basic function
function log() {
	local msg=$@; shift
	local datetime=$(date +"%Y-%m-%d %H:%M:%S")
	echo "$datetime $msg"
}

function report_by_sms {
	local msg=$@; shift
	local my_dir=$(dirname $(readlink -m $0))
	local phones=$(cat conf/sms_list)

	if [[ -f ./disable_sms ]]; then
		log "告警短信被屏蔽. 内容:[$msg]"
	else
		$my_dir/sendsms.py $phones "$msg"
		log "已发送告警短信到$phones, 内容:[$msg]"
	fi
}

#nginx daemon
function CheckPort() {
	local nginx_status=$(cat $1 | awk '{print $7}' | awk -F'/' '{print $2}')
	local nginx_listenon=$(cat $1 | awk '{print $6}')

	if [[ $nginx_status = "nginx" ]] && [[ $nginx_listenon = "LISTEN" ]]; then
		echo 0
	fi
}

function CheckNginx() {
	local flag_daemon=$(service nginx status | grep -c 'running')
	local flag_mirrorlist_port=$(netstat -pln | grep 'nginx' | awk '{print $4}' | awk 'BEGIN{FS=":";}{print $2}' | grep '80')
	local flag_vault_port=$(netstat -pln | grep 'nginx' | awk '{print $4}' | awk 'BEGIN{FS=":";}{print $2}' | grep '8080')
	local flag_php=$(curl "http://172.17.1.165:80/?release=6.3&repo=os&arch=x86_64" | wc -l)

	if [[ $flag_daemon -ne 1 ]]; then
		log "ERROR: nginx is not running"
		convergentSend "中心YUM源nginx没有启动,请及时开启"
	else
		DownConvergence "nginx is running"
	fi

	if [[ ! -z $flag_mirrorlist_port ]]; then
		DownConvergence "nginx is listening port 80"
	else
		log "ERROR: nginx does not exist or port for mirrorlist is wrong"
		convergentSend "nginx mirrorlist服务器没有监听80端口,或者nginx没有启动"
	fi

	if [[ ! -z $flag_vault_port ]]; then
		DownConvergence "nginx is listening port 8080"
	else
		log "ERROR: nginx does not exist or port for vault is wrong"
		convergentSend "nginx vault服务器没有监听8080端口,或者nginx没有启动"
	fi

	if [[ $flag_php -eq 1 ]]; then
		DownConvergence "mirrorlist php is working, sends the ip list"
	else
		log "ERROR: mirrorlist php is wrong"
		convergentSend "中心源php存在问题,没有返回指定数目的cache源IP,请及时处理"
	fi
}

#throughputs
function GetNetValue() {
	local value=$(cat /proc/net/dev | grep 'eth0' | tr : " " | awk '{print $'$1'}')
	echo $value
}

function CheckThroughputs() {
	local RX_pre=$(cat /proc/net/dev | grep 'eth2' | tr : " " | awk '{print $2}')
	local TX_pre=$(cat /proc/net/dev | grep 'eth2' | tr : " " | awk '{print $10}')

	sleep 1

	local RX_next=$(cat /proc/net/dev | grep 'eth2' | tr : " " | awk '{print $2}')
	local TX_next=$(cat /proc/net/dev | grep 'eth2' | tr : " " | awk '{print $10}')

	RX=$(($RX_next-$RX_pre))
	TX=$(($TX_next-$TX_pre))

	if [[ $RX -lt 1024 ]];then
      		RX="$RX B/s"
    	elif [[ $RX -gt 1048576 ]];then
      		RX=$(echo $RX | awk '{print $1/1048576 "MB/s"}')
		log "ERROR: Receiving tunnel is busy"
		convergentSend "中心源接收流量太大,请保持观察"
    	else
      		RX=$(echo $RX | awk '{print $1/1024 "KB/s"}')
    	fi

	if [[ $TX -lt 1024 ]];then
      		TX="${TX}B/s"
      	elif [[ $TX -gt 1048576 ]];then
      		TX=$(echo $TX | awk '{print $1/1048576 "MB/s"}')
    	else
      		TX=$(echo $TX | awk '{print $1/1024 "KB/s"}')
    	fi

	log "RX is $RX, TX is $TX"
}

function CheckErrDrop() {
	local RX_packets_pre=$(GetNetValue 3)
	local RX_errs_pre=$(GetNetValue 4)
	local RX_drop_pre=$(GetNetValue 5)
	local TX_packets_pre=$(GetNetValue 11)
	local TX_errs_pre=$(GetNetValue 12)
	local TX_drop_pre=$(GetNetValue 13)

	sleep 3

	local RX_packets_next=$(GetNetValue 3)
	local RX_errs_next=$(GetNetValue 4)
	local RX_drop_next=$(GetNetValue 5)
	local TX_packets_next=$(GetNetValue 11)
	local TX_errs_next=$(GetNetValue 12)
	local TX_drop_next=$(GetNetValue 13)

	RX_packets=$(($RX_packets_next-$RX_packets_pre))
	RX_errs=$(($RX_errs_next-$RX_errs_pre))
	RX_drop=$(($RX_drop_next-$RX_drop_pre))
	TX_packets=$(($TX_packets_next-$TX_packets_pre))
	TX_errs=$(($TX_errs_next-$TX_errs_pre))
	TX_drop=$(($TX_drop_next-$TX_drop_pre))

	if [[ $RX_packets -ne 0 ]]; then
		RX_err_rate=$(($RX_errs/$RX_packets))
		RX_drop_rate=$(($RX_drop/$RX_packets))
	else
		RX_err_rate=0
		RX_drop_rate=0
	fi

	if [[ $TX_packets -ne 0 ]] ; then
		TX_err_rate=$(($TX_errs/$TX_packets))
		TX_drop_rate=$(($TX_drop/$TX_packets))
	else
		TX_err_rate=0
		TX_drop_rate=0
	fi

	if [[ `echo "$RX_err_rate > 0" | bc` -eq 1 ]] || [[ `echo "$RX_drop_rate > 0" | bc` -eq 1 ]]; then

		log "ERROR: RX tunnel has some errors or drops some packets"
		convergentSend "中心源接收包时出现错误或者丢包,请及时查看处理"
	else
		DownConvergence "No error or drop in RX channel"
	fi

	if [[ `echo "$TX_err_rate > 0" | bc` -eq 1 ]] || [[ `echo "$TX_drop_rate > 0" | bc` -eq 1 ]]; then
		log "ERROR: TX tunnel has some errors or drops some packets"
		convergentSend "中心源发送包出现错误或者丢包,请及时查看处理"
	else
		DownConvergence "No error or drop in TX channel"
	fi
}

GetLineField() {
	local file_name=$1;shift
	local line=$1;shift
	local field=$1;shift

	cat $file_name | sed -n ''$line'p' | awk '{print $'$field'}'
}

#http
function CheckHTTP() {
	local file_name=$1; shift
	local request_time=$(cat $file_name | awk '{print $7}' | sort -nr | sed -n '1p')
	local request_line=$(cat -b $file_name | grep $request_time | sed -n '1p' | awk '1{print $1}')
	local remote_addr=$(GetLineField $file_name $request_line 2)
	local host=$(GetLineField $file_name $request_line 3)
	local request=$(GetLineField $file_name $request_line 5)
	local status=$(cat $file_name | awk '{print $9}' | grep -v '200' | wc -l)

	if [[ `echo "$request_time > 120" | bc` -eq 1 ]]; then
		log "ERROR: $remote_addr Slowly complete the yum request"
		convergentSend "中心源处理客户端的需求缓慢,请保持观察"
	else
		DownConvergence "HTTP is running well"
	fi

	if [[ $status -gt 0 ]]; then
		log "ERROR: $host $remote_addr request status is wrong"
		convergentSend "中心源处理客户端request时,存在不成功情况,请及时处理"
	else
		DownConvergence "HTTP is running well"
	fi
}

#begin the scritps
while :
do
	CheckNginx
	CheckThroughputs
	CheckErrDrop
	if [[ `cat $LOG_MIRROR` != "" ]] && [[ `cat $LOG_MIRRORLIST` != "" ]]; then
		CheckHTTP $LOG_MIRROR
		CheckHTTP $LOG_MIRRORLIST
	fi

	sleep 15
done
```
