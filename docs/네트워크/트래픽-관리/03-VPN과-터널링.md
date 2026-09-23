# VPN과 터널링

## VPN 프로토콜

### VPN 개요

VPN(Virtual Private Network)은 공용 네트워크를 통해 사설 네트워크에 연결하는 방식이다. 여기서는 암호화된 터널로 통신을 보호하는 구성을 살펴본다.

Remote Access 또는 Client-to-Site 구성에서는 개인 기기가 회사 VPN 서버를 통해 내부망에 접속한다. Site-to-Site 구성은 본사와 지사의 VPN Gateway 사이에 터널을 만들어 두 네트워크를 연결한다.

### IPsec

- IPsec(Internet Protocol Security)은 네트워크 계층에서 IP 패킷을 암호화하고 인증하는 프로토콜 스위트임.

- IPsec 구성 요소:
- IPsec
- 프로토콜:
- AH (Authentication, ESP (Encapsulating
- Header), Security Payload)
- 인증만,  인증 + 암호화
- 무결성 보장,  기밀성, 무결성 보장
- 키 관리:
- IKE (Internet Key Exchange)
- IKEv1 / IKEv2
- SA (Security Association) 협상
- 키 교환 (Diffie-Hellman)
- 모드:
- Transport Mode, Tunnel Mode
- 호스트 간 통신,  네트워크 간 통신
- 원본 IP 헤더 유지,  새 IP 헤더 추가

- IPsec Transport vs Tunnel Mode:
- Transport Mode:
- IP 헤더, ESP/AH 헤더, Payload
- (원본), (암호화)
- 인증 범위
- Tunnel Mode:
- 새 IP, ESP/AH 헤더, 원본 IP, Payload
- 헤더, 헤더
- 암호화 범위

### OpenVPN

- OpenVPN은 SSL/TLS 기반의 오픈소스 VPN 솔루션임.

- OpenVPN 특징:
- SSL/TLS 기반 암호화
- TCP/UDP 모두 지원
- 다양한 인증 방식 (인증서, ID/PW, 2FA)
- 크로스 플랫폼 지원
- NAT/방화벽 친화적
- OpenVPN 서버 설정 (/etc/openvpn/server.conf):

```conf
# /etc/openvpn/server.conf

# 네트워크 설정
port 1194
proto udp
dev tun

# 인증서 경로
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh2048.pem

# VPN 네트워크
server 10.8.0.0 255.255.255.0

# 클라이언트 라우팅
push "route 192.168.1.0 255.255.255.0"
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"

# 연결 설정
keepalive 10 120
cipher AES-256-GCM
auth SHA256
user nobody
group nogroup
persist-key
persist-tun

# 로깅
status /var/log/openvpn-status.log
log-append /var/log/openvpn.log
verb 3
```

```conf
# OpenVPN 클라이언트 설정 (client.ovpn)

client
dev tun
proto udp
remote vpn.example.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun

ca ca.crt
cert client.crt
key client.key

remote-cert-tls server
cipher AES-256-GCM
auth SHA256
verb 3
```

### WireGuard

- WireGuard는 최신 암호화를 사용하는 간단하고 빠른 VPN 프로토콜임.

- WireGuard 특징:
- WireGuard
- 코드베이스: ~4,000줄 (OpenVPN: ~100,000줄)
- 암호화: ChaCha20, Poly1305, Curve25519, BLAKE2s
- 성능: OpenVPN 대비 약 3-4배 빠름
- UDP만 사용 (더 빠르고 효율적)
- 커널 레벨 구현 (Linux 5.6+)
- Cryptokey Routing (공개키 기반 라우팅)

```bash
# WireGuard 설치 (Ubuntu)
sudo apt install wireguard

# 키 생성
wg genkey | tee privatekey | wg pubkey > publickey
```

```ini
# /etc/wireguard/wg0.conf (서버)

[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

# IP 포워딩 활성화
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# 클라이언트 1
PublicKey = CLIENT1_PUBLIC_KEY
AllowedIPs = 10.0.0.2/32

[Peer]
# 클라이언트 2
PublicKey = CLIENT2_PUBLIC_KEY
AllowedIPs = 10.0.0.3/32
```

```ini
# /etc/wireguard/wg0.conf (클라이언트)

[Interface]
Address = 10.0.0.2/24
PrivateKey = CLIENT_PRIVATE_KEY
DNS = 8.8.8.8

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0    # 모든 트래픽 VPN으로
PersistentKeepalive = 25        # NAT 유지
```

```bash
# WireGuard 인터페이스 시작
sudo wg-quick up wg0

# 상태 확인
sudo wg show

# 인터페이스 중지
sudo wg-quick down wg0
```

### VPN 프로토콜 비교

- 특성: 속도:
  - IPsec/IKEv2: 빠름
  - OpenVPN: 보통
  - WireGuard: 매우 빠름
- 특성: 보안:
  - IPsec/IKEv2: 높음
  - OpenVPN: 높음
  - WireGuard: 높음
- 특성: 코드 크기:
  - IPsec/IKEv2: 매우 큼
  - OpenVPN: 큼 (~100K)
  - WireGuard: 작음 (~4K)
- 특성: 설정 복잡도:
  - IPsec/IKEv2: 복잡
  - OpenVPN: 중간
  - WireGuard: 간단
- 특성: 프로토콜:
  - IPsec/IKEv2: UDP/TCP
  - OpenVPN: UDP/TCP
  - WireGuard: UDP만
- 특성: 방화벽 우회:
  - IPsec/IKEv2: 어려움
  - OpenVPN: 쉬움 (443포트)
  - WireGuard: 중간
- 특성: 모바일 지원:
  - IPsec/IKEv2: 내장 (iOS/Android)
  - OpenVPN: 앱 필요
  - WireGuard: 앱 필요
- 특성: 기업 환경:
  - IPsec/IKEv2: 최적
  - OpenVPN: 좋음
  - WireGuard: 신흥

## 터널링

### 터널링 개요

터널링은 한 프로토콜의 패킷을 다른 프로토콜로 감싸 전송하는 기술이다. 원본 헤더와 페이로드 바깥에 터널 헤더를 붙이면 중간 네트워크는 이 외부 헤더를 보고 전달한다. 원본 패킷은 터널 방식에 따라 암호화할 수도 있다.

### 주요 터널링 프로토콜

#### GRE (Generic Routing Encapsulation)

아래 예제는 Router A의 `10.0.1.1`과 Router B의 `10.0.2.1` 사이에 GRE 터널을 만든다. 터널 인터페이스에는 `172.16.1.1/30`과 `172.16.1.2/30`을 지정하고, 양쪽의 `192.168.1.0/24`와 `192.168.2.0/24` 네트워크로 향하는 경로를 추가한다.

```bash
# Router A에서 GRE 터널 생성
sudo ip tunnel add gre1 mode gre remote 10.0.2.1 local 10.0.1.1 ttl 255
sudo ip link set gre1 up
sudo ip addr add 172.16.1.1/30 dev gre1
sudo ip route add 192.168.2.0/24 via 172.16.1.2

# Router B에서 GRE 터널 생성
sudo ip tunnel add gre1 mode gre remote 10.0.1.1 local 10.0.2.1 ttl 255
sudo ip link set gre1 up
sudo ip addr add 172.16.1.2/30 dev gre1
sudo ip route add 192.168.1.0/24 via 172.16.1.1
```

#### SSH 터널링

```bash
# 1. Local Port Forwarding
# 로컬 8080 → SSH 서버 → 원격 서버 80
ssh -L 8080:remote-server:80 user@ssh-server

# 로컬에서 접근: http://localhost:8080

# 2. Remote Port Forwarding
# 원격 8080 → SSH 서버 → 로컬 80
ssh -R 8080:localhost:80 user@ssh-server

# 외부에서 접근: http://ssh-server:8080

# 3. Dynamic Port Forwarding (SOCKS Proxy)
ssh -D 1080 user@ssh-server

# 브라우저에서 SOCKS5 프록시로 localhost:1080 설정

# 4. 백그라운드 실행
ssh -f -N -L 8080:remote-server:80 user@ssh-server
# -f: 백그라운드
# -N: 원격 명령 실행 안함
```

- SSH 터널링 시나리오:
- Local Port Forwarding:
- Local, SSH, SSH Server, DB Server
- :8080, →, →, :5432
- 방화벽 뒤의 DB에 안전하게 접근
- Remote Port Forwarding:
- Local, SSH, SSH Server, 외부 사용자
- :3000, →, :8080, ←
- 로컬 개발 서버를 외부에 노출
- SOCKS Proxy:
- Local, SSH, SSH Server, Any Server
- :1080, →, →, anywhere
- 모든 트래픽을 SSH 서버를 통해 전송

#### SSL/TLS 터널링

```python
# Python에서 SSL 소켓 터널링
import socket
import ssl

def ssl_tunnel():
    # SSL 컨텍스트 생성
    context = ssl.create_default_context()

    # 일반 소켓 생성
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # SSL로 래핑
    ssl_sock = context.wrap_socket(sock, server_hostname='example.com')

    # 연결
    ssl_sock.connect(('example.com', 443))

    # HTTPS 요청
    ssl_sock.send(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")

    # 응답 수신
    response = ssl_sock.recv(4096)
    print(response.decode())

    ssl_sock.close()
```

#### stunnel (SSL 터널링)

```ini
# /etc/stunnel/stunnel.conf

# 클라이언트 모드 - 평문 트래픽을 SSL로 암호화
[mysql-client]
client = yes
accept = 127.0.0.1:3306
connect = remote-db:13306

# 서버 모드 - SSL 트래픽을 평문으로 복호화
[mysql-server]
accept = 13306
connect = 127.0.0.1:3306
cert = /etc/stunnel/stunnel.pem
```

## 참고 자료

- [RFC 7230 - HTTP/1.1 Message Syntax and Routing](https://datatracker.ietf.org/doc/html/rfc7230)
- [RFC 4301 - Security Architecture for IP](https://datatracker.ietf.org/doc/html/rfc4301) (IPsec)
- [WireGuard White Paper](https://www.wireguard.com/papers/wireguard.pdf)
- [OpenVPN Documentation](https://openvpn.net/community-resources/)
- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [HAProxy Documentation](https://www.haproxy.org/download/2.6/doc/configuration.txt)
- [Squid Proxy Documentation](http://www.squid-cache.org/Doc/)
- [IVPN - VPN Protocol Comparison](https://www.ivpn.net/pptp-vs-ipsec-ikev2-vs-openvpn-vs-wireguard/)

## 관련 학습

- [프록시와 리버스 프록시](01-프록시와-리버스-프록시.md)
- [TLS와 HTTPS](../../보안/암호화와-전송-보안/02-TLS와-HTTPS.md)
- [로드 밸런싱](02-로드-밸런싱.md)
- [CDN](04-CDN.md)
