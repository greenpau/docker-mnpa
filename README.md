# mnpa in a docker container

Build and quickly test `mnpd` container:

```
cd mnpa/docker/ubuntu && docker build -t mnpd .
docker run -d -i -t --name=mnpd0001 --privileged --cap-add all --net=host -v /tmp:/tmp mnpd /bin/bash
docker exec -it mnpd0001 /bin/bash
```

Perform additional preliminary tests:

```
docker run --rm -i -t --name=mnpd1234 --privileged --cap-add all --net=host -v /tmp:/tmp mnpd mnpd --sender ens1f0/100/64-1400/1/0/239.1.1.1/5001/TEST1 --verbose
docker run --rm -i -t --name=mnpd1234 --privileged --cap-add all --net=host -v /tmp:/tmp mnpd ping 192.168.1.1
```


Stress-test containers with the below script (e.g. `mnpd.build`):

```
cd "$(dirname "$0")"
MCS=$(docker ps -a | egrep "mnpd[0-9]{4}" | cut -d" " -f1 | wc -l | xargs);
if (( $MCS > 0 )); then
  echo "error: currently, there are ${MCS} active containers on this host";
  echo "  please stop them using the below command:";
  echo '    docker rm -f $(docker ps -a | egrep "mnpd[0-9]{4}" | cut -d" " -f1 | xargs)';
  exit 1;
fi

cat > mnpd.list << EOF
239.1.1.1:5001
239.1.2.2:5001
239.2.1.1:5001
239.2.2.2:5001
EOF

truncate -s 0 mnpd.sh

gawk -F":" 'BEGIN {
 MCNT=1;
 CT="mnpd";
 CN="mnpd";
 INTV=1;
 TTL=255;
 "hostname -s" | getline HOST;
}{
    for ( x = 1; x <= 20; x++ ) {
        MSG=HOST"."x"."$1"."$2;
        gsub(/[^0-9a-z\.]/, ".", MSG);
                printf("echo -n \"starting %s%04d \"\n", CN, MCNT);
        printf("docker run -d -i -t --name=%s%04d --privileged --cap-add all --net=host -v /tmp:/tmp %s %s --sender  ens1f0/0/64-1400/1/0/%s/%s/%s\n",
               CN, MCNT, CT, CN, $1, $2, MSG);
        MCNT++;
    }
}' mnpd.list > mnpd.sh
chmod +x mnpd.sh
./mnpd.sh
rm mnpd.{sh,list}
```

Validate the commands:

```
docker ps --no-trunc=true | grep mnpd
```

Validate network traffic bandwidth on the port:

```
ethtool -S ens1f0 | grep tx_packets
ethtool -S ens1f0 | grep tx.*_to_
```

Also, please consider enabling [Kernel Bypass](https://blog.cloudflare.com/kernel-bypass/).
