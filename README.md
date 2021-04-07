#!/bin/bash

LOG_FILE=ddns.log

# Will get the Wan Port IP address from your router, since the IP address is incorrect when getting from the website if you are using a SSR in your router. 
# Here my router's IP is 192.168.2.1, replace this to your router's IP
STR=`ssh admin@192.168.2.1 "ifconfig | grep -A 1 ppp | grep inet"`

AliDDNS_LocalIP=`echo $STR | awk -F ":" '{print $2}' | awk -F " " '{print $1}'`


AliDDNS_DomainName="domain.com" # Replace this to your domain
AliDDNS_SubDomainName="ddns"    # Replace this to your sub domain name
AliDDNS_TTL="600"
AliDDNS_AK="Your Ali Access Key" #Replace this to your Ali Access Key
AliDDNS_SK="Your Ali Secret Key" # Replace this to your Ali secret Key

AliDDNS_DomainIP=`nslookup $AliDDNS_SubDomainName.$AliDDNS_DomainName $AliDDNS_DomainServerIP 2>&1`
if [ "$?" -eq "0" ]
then
	AliDDNS_DomainIP=`echo "$AliDDNS_DomainIP" | grep 'Address:' | tail -n1 | awk '{print $NF}'`
	if [ "$AliDDNS_LocalIP" = "$AliDDNS_DomainIP" ]
	then
		echo "[$(date "+%G/%m/%d %H:%M:%S")] Local IP ($AliDDNS_LocalIP) is the same with Domain IP ($AliDDNS_DomainIP)" >> $LOG_FILE
		exit 0
	fi
fi

# Start to Change IP
timestamp=`date -u "+%Y-%m-%dT%H%%3A%M%%3A%SZ"` 
urlencode() {
	# urlencode <string>
	out="" 
	while read -n1 c 
	do 
		case $c in 
			[a-zA-Z0-9._-]) out="$out$c" ;; 
			*) out="$out`printf '%%%02X' "'$c"`" ;; 
		esac 
	done 
	echo -n $out
}

# Encrypt URL
enc() {
	echo -n "$1" | urlencode
}

send_request() {
	local args="AccessKeyId=$AliDDNS_AK&Action=$1&Format=json&$2&Version=2015-01-09"
	local hash=$(echo -n "GET&%2F&$(enc "$args")" | openssl dgst -sha1 -hmac "$AliDDNS_SK&" -binary | openssl base64)
	curl -s "http://alidns.aliyuncs.com/?$args&Signature=$(enc "$hash")"
}

get_recordid() {
	grep -Eo '"RecordId":"[0-9]+"' | cut -d':' -f2 | tr -d '"'
}

query_recordid() {
	send_request "DescribeSubDomainRecords" "SignatureMethod=HMAC-SHA1&SignatureNonce=$timestamp&SignatureVersion=1.0&SubDomain=$AliDDNS_SubDomainName.$AliDDNS_DomainName&Timestamp=$timestamp"
}

update_record() {
	send_request "UpdateDomainRecord" "RR=$AliDDNS_SubDomainName&RecordId=$1&SignatureMethod=HMAC-SHA1&SignatureNonce=$timestamp&SignatureVersion=1.0&TTL=$AliDDNS_TTL&Timestamp=$timestamp&Type=A&Value=$AliDDNS_LocalIP"
}

add_record() {
    send_request "AddDomainRecord&DomainName=$AliDDNS_DomainName" "RR=$AliDDNS_SubDomainName&SignatureMethod=HMAC-SHA1&SignatureNonce=$timestamp&SignatureVersion=1.0&TTL=$AliDDNS_TTL&Timestamp=$timestamp&Type=A&Value=$AliDDNS_LocalIP"
}

if [ "$AliDDNS_RecordID" = "" ]
then
    AliDDNS_RecordID=`query_recordid | get_recordid`
fi

if [ "$AliDDNS_RecordID" = "" ]
then
    AliDDNS_RecordID=`add_record | get_recordid`
    echo "[$(date "+%G/%m/%d %H:%M:%S")] Added RecordID : $AliDDNS_RecordID"
else
    update_record $AliDDNS_RecordID
    echo "[$(date "+%G/%m/%d %H:%M:%S")] Updated RecordID : $AliDDNS_RecordID"
fi

if [ "$AliDDNS_RecordID" = "" ]; then
    echo "[$(date "+%G/%m/%d %H:%M:%S")] DDNS Update Failed !"  >> $LOG_FILE
else
    echo "[$(date "+%G/%m/%d %H:%M:%S")] DDNS Update Success, New IP is : $AliDDNS_LocalIP"  >> $LOG_FILE
fi
