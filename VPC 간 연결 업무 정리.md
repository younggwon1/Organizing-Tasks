# VPC 간 연결 업무 정리

## 1. VPC Peering

**구성도**

![Redis Cloud VPC Peering](https://github.com/younggwon1/toy-project/assets/108652621/6008606d-3965-43ac-97e0-190bde514890)

**상황**

> AWS A Accounts 내 EKS 에서 운영되는 서비스 <-> Redis Cloud 연결이 필요

**작업 순서**

1. Redis Cloud VPC Peering 연결 요청
   1. To set up VPC peering:
      1. From the [Redis Cloud console](https://app.redislabs.com/), select the **Subscriptions** menu and then select your subscription from the list.
      2. Select **Connectivity > VPC Peering**.
      3. Select **Add peering**.
      4. Enter **VPC peering** details.
   2. [참고 문서](https://redis.io/docs/latest/operate/rc/security/vpc-peering/)
2. AWS A Accounts 에서 VPC Peering 연결 승락
3. AWS A Accounts VPC Main route table(default rtb) 에 Redis Cloud VPC CIDR 추가
   1. Destination : Redis Cloud VPC CIDR (10.200.100.0/20)
   2. Target : Peering connection ID
4. AWS A Accounts EKS Subnet route table 에 Redis Cloud VPC CIDR 추가

**테스트**

EKS 내 (EKS Subnet)에서 운영되는 서비스에서 Redis Cloud Endpoint 호출

```
sh-4.2$ curl -v {redis cloud endpoint}
*   Trying {redis cloud ip}:{redis cloud port}...
* Connected to {redis cloud endpoint}({redis cloud ip}) port {redis cloud port} (#0)
> GET / HTTP/1.1
> Host: {redis cloud endpoint}
> User-Agent: curl/8.0.1
> Accept: */*
>
* Received HTTP/0.9 when not allowed
* Closing connection 0
curl: (1) Received HTTP/0.9 when not allowed
```

---

## 2. Transit Gateway

**전반적인 상황**

> AWS A Accounts VPC 와 AWS B Accounts VPC 간의 연결을 위해 Transit Gateway 를 사용
>
> Transit Gateway 로 각 Accounts 의 VPC 가 연결된 상황
>
> - AWS A Accounts VPC Main route table(default rtb) 에 AWS B Accounts VPC CIDR 추가
>   - Destination : AWS B Accounts VPC CIDR (10.168.0.0/16)
>   - Target : Transit gateway ID
> - AWS B Accounts VPC Main route table(default rtb) 에 AWS A Accounts VPC CIDR 추가
>   - Destination : AWS A Accounts VPC CIDR (192.168.0.0/16)
>   - Target : Transit gateway ID

**첫번째 상황**

> AWS A Accounts 내 EKS 에서 운영되는 서비스 -> AWS B Accounts 내의 PostgreSQL 에 연결이 필요

**구성도**

![Transit Gateway first situation](https://github.com/younggwon1/toy-project/assets/108652621/6a6850af-167a-4211-a8af-8361d6af75fb)

**작업 순서**

> TCP 통신이기 때문에 **3-way handshake** 를 고려.
>
> (EKS -> PostgreSQL 만을 고려하는 것이 아닌, 호출에 대한 응답도 필요하므로 PostgreSQL -> EKS 도 고려가 필요)

1. AWS B Accounts 내 PostgreSQL Security Group 에 AWS A Accounts VPC CIDR 추가
   1. Source : 192.168.0.0/16
   2. Port : 5432
2. AWS B Accounts 의 각 PostgreSQL Subnet route table 에 AWS A Accounts VPC CIDR 추가
   1. Destination : AWS A Accounts VPC CIDR (192.168.0.0/16)
   2. Target : Transit gateway ID
3. AWS A Accounts 의 각 EKS Subnet route table 에 AWS B Accounts VPC CIDR 추가
   1. Destination : AWS B Accounts VPC CIDR (10.168.0.0/16)
   2. Target : Transit gateway ID

**두번째 상황**

> AWS B Accounts 내 EKS 에서 운영되는 서비스 -> AWS A Accounts 내 EKS 에서 운영되는 서비스를 호출하기 위해 Internal LB 와 연결된 도메인을 호출하는 상황 (호출 시 Time Out 발생)

**구성도**

![Transit Gateway second situation](https://github.com/younggwon1/toy-project/assets/108652621/15e38758-d7e4-4433-8d20-f6aa5e097f73)

**작업 순서**

1. AWS A Accounts 의 각 EKS Subnet route table 에 AWS B Accounts VPC CIDR 추가
   1. Destination : AWS B Accounts VPC CIDR (10.168.0.0/16)
   2. Target : Transit gateway ID
2. AWS A Accounts 의 Internal LB 의 Security Group 에 AWS B Accounts 내 각 EKS Subnet CIDR 추가
   1. Source : AWS B Accounts 내 각 EKS Subnet CIDR
   2. Port : 80, 443
3. AWS B Accounts 의 각 EKS Subnet route table 에 AWS A Accounts VPC CIDR 추가
   1. Destination : AWS A Accounts VPC CIDR (192.168.0.0/16)
   2. Target : Transit gateway ID

-> 여기까지 했을 때 통신이 될 줄 알았지만 여전히 Time Out 발생, 추가 작업으로 다음 스텝을 진행

4. AWS A Accounts 의 Internal LB의 Network mapping 탭을 보게 되면 Mapping 된 Subnet이 있는데, 해당 Subnet 의 route table에 AWS B Accounts VPC CIDR 추가
   1. Destination : AWS B Accounts VPC CIDR (10.168.0.0/16)
   2. Target : Transit gateway ID

-> Time Out 해결
