# ELK 스택 배포 실습 가이드

이 가이드는 ELK 스택(Elasticsearch, Logstash, Kibana)을 다양한 환경에서 배포하고 검증하는 실습을 다룹니다. 초보자는 Stage 1부터 시작하여 점진적으로 고급 환경(Stage 5)으로 넘어갈 수 있습니다. 각 단계는 "무엇을? 왜? 어떻게?" 구조로 구성되며, 오류 처리와 최적화 팁을 포함합니다.

## 대상 독자
- **Stage 1**: Docker 기본 지식
- **Stage 2**: TLS 및 보안 개념 이해
- **Stage 3**: Kubernetes 기본 지식 (kubectl, 네임스페이스)
- **Stage 4**: Linux 시스템 관리 경험
- **Stage 5**: 클라우드 서비스(AWS, Elastic Cloud) 사용 경험

---

## Stage 1. 로컬 개발·테스트 환경 (Docker Compose)

### 무엇을?
ELK 스택을 단일 호스트에서 Docker Compose로 배포하여 데이터 파이프라인을 검증합니다.

### 왜?
- 빠른 프로토타입 작성과 디버깅
- 팀원 간 동일한 개발 환경 보장

### 어떻게?
Docker Compose로 컨테이너를 오케스트레이션하며, 최소 설정으로 즉시 기동/제거합니다.

#### 1.1 프로젝트 디렉토리 구조 준비
```bash
mkdir -p elk-docker/{elasticsearch/config,logstash/pipeline,kibana/config}
cd elk-docker
```

**디렉토리 구조**:
```
elk-docker/
├── docker-compose.yml
├── elasticsearch/
│   └── config/elasticsearch.yml
├── logstash/
│   └── pipeline/logstash.conf
└── kibana/
    └── config/kibana.yml
```

#### 1.2 `docker-compose.yml` 작성
```yaml
version: '3.8'  # Docker Compose 파일 포맷 버전 지정 (3.8 이상을 권장)

services:
  es-master:  # Elasticsearch 마스터 노드 서비스 정의
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.2  # Elasticsearch 8.10.2 공식 이미지
    container_name: es-master  # 컨테이너 이름 설정
    environment:  # Elasticsearch 환경 변수 설정
      - node.name=es-master                    # 노드 이름
      - cluster.name=es-cluster                # 클러스터 이름
      - discovery.seed_hosts=es-master,es-data1,es-data2
      # 클러스터 디스커버리 시드 호스트 목록
      - cluster.initial_master_nodes=es-master
      # 초기 마스터 선출 대상 노드
      - node.master=true                       # 마스터 역할 활성화
      - node.data=false                        # 데이터 역할 비활성화 (마스터 전용)
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"       # JVM 힙 메모리 최소/최대 512MB로 설정
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      # 호스트의 custom 설정 파일을 컨테이너에 매핑
    ports:
      - 9200:9200  # 호스트 9200 포트를 컨테이너 9200 포트로 매핑 (REST API)
    networks:
      - elk-network  # ELK 스택 전용 네트워크에 연결

  es-data1:  # Elasticsearch 데이터 전용 노드1 서비스
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.2
    container_name: es-data1
    environment:
      - node.name=es-data1
      - cluster.name=es-cluster
      - discovery.seed_hosts=es-master,es-data1,es-data2
      - node.master=false   # 마스터 역할 비활성화
      - node.data=true      # 데이터 역할 활성화
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - data1:/usr/share/elasticsearch/data
      # Docker 볼륨을 데이터 디렉토리에 매핑해 영속성 확보
    networks:
      - elk-network

  es-data2:  # Elasticsearch 데이터 전용 노드2 서비스
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.2
    container_name: es-data2
    environment:
      - node.name=es-data2
      - cluster.name=es-cluster
      - discovery.seed_hosts=es-master,es-data1,es-data2
      - node.master=false
      - node.data=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - data2:/usr/share/elasticsearch/data
    networks:
      - elk-network

  logstash:  # Logstash 서비스 정의
    image: docker.elastic.co/logstash/logstash:8.10.2
    container_name: logstash
    depends_on:
      - es-master  # es-master 컨테이너가 먼저 시작된 후 의존성 만족
    volumes:
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      # 파이프라인 설정 파일 매핑
    ports:
      - 5044:5044  # Beats(예: Filebeat) 입력 포트 매핑
    networks:
      - elk-network

  kibana:  # Kibana 서비스 정의 (웹 UI)
    image: docker.elastic.co/kibana/kibana:8.10.2
    container_name: kibana
    depends_on:
      - es-master  # Elasticsearch 마스터 노드가 실행 중이어야 Kibana 시작 가능
    volumes:
      - ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml
      # Kibana 설정 파일 매핑
    ports:
      - 5601:5601  # 호스트 5601 포트를 컨테이너 5601 포트로 매핑 (웹 UI 접속)
    networks:
      - elk-network

volumes:
  data1:  # es-data1 컨테이너의 데이터 영속화용 볼륨
  data2:  # es-data2 컨테이너의 데이터 영속화용 볼륨

networks:
  elk-network:  # ELK 스택 전용 브리지 네트워크 정의
    driver: bridge

```

**설명**:
- `node.master`와 `node.data`: 마스터 노드는 클러스터 관리, 데이터 노드는 데이터 저장을 담당.
- `ES_JAVA_OPTS`: 메모리 설정으로 로컬 환경에서 리소스 사용량 최적화.
- `networks`: 컨테이너 간 통신을 위한 전용 네트워크 추가.

#### 1.3 설정 파일 작성
**1.3.1 `elasticsearch/config/elasticsearch.yml`**:
```yaml
cluster.name: es-cluster
network.host: 0.0.0.0
xpack.security.enabled: false
```

**1.3.2 `logstash/pipeline/logstash.conf`**:
```conf
input {
  tcp {
    port => 5044
    codec => json_lines
  }
}

filter {
  if [timestamp] {
    date { match => ["timestamp", "ISO8601"] }
  }
}

output {
  elasticsearch {
    hosts => ["http://es-master:9200"]
    index => "demo-logs-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

**1.3.3 `kibana/config/kibana.yml`**:
```yaml
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://es-master:9200"]
xpack.security.enabled: false
```

#### 1.4 클러스터 기동 및 검증
```bash
docker-compose up -d
docker-compose ps
docker-compose logs -f es-master
```

**헬스체크**:
```bash
curl -s http://localhost:9200/_cluster/health?pretty
```
- **정상**: `status: green` 또는 `yellow`
- **비정상**: `red`인 경우, 로그 확인(`docker-compose logs es-master`) 및 `discovery.seed_hosts` 설정 점검.

#### 1.5 샘플 로그 실습
**로그 전송**:
```bash
echo '{"message":"Hello ELK","timestamp":"'"$(date -Iseconds)"'"}' | nc localhost 5044
```

**인덱스 확인**:
```bash
curl -s 'http://localhost:9200/_cat/indices/demo-logs-*?v'
```

**Kibana에서 확인**:
1. `http://localhost:5601` 접속
2. **Discover** → **Create index pattern** → `demo-logs-*` 등록
3. 로그 조회 및 간단한 DSL 쿼리 예시:
   ```json
   {
     "query": {
       "match": {
         "message": "Hello ELK"
       }
     }
   }
   ```

#### 1.6 정리
```bash
docker-compose down -v
```
- `-v`: 볼륨 삭제로 데이터 완전 제거.

**오류 디버깅 팁**:
- **Logstash 연결 실패**: `es-master:9200`에 접근 가능한지 확인(`curl http://es-master:9200`).
- **Kibana 연결 오류**: `elasticsearch.hosts` 설정 확인.
- **클러스터 yellow 상태**: 데이터 노드 연결 확인(`docker-compose logs es-data1`).

**성능 최적화 팁**:
- 샤드 수: 소규모 환경에서는 인덱스당 샤드 1개, 레플리카 1개로 설정.
- JVM 힙: 로컬 환경에서는 512MB~1GB로 제한.

---

## Stage 2. 스테이징·QA 환경

### 무엇을?
Stage 1에 TLS, X-Pack 보안, Metricbeat 모니터링을 추가한 리허설 환경.

### 왜?
- 운영 전 보안 및 성능 검증.
- 실제 운영 환경과 유사한 설정 테스트.

### 어떻게?
Docker Compose에 X-Pack 보안 활성화, Metricbeat으로 모니터링 데이터 수집.

#### 2.1 TLS 인증서 생성
```bash
mkdir -p elk-docker/certs
cd elk-docker/certs
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -days 365 -subj "/CN=elk-ca" -out ca.crt
docker run -it --rm -v $(pwd):/certs docker.elastic.co/elasticsearch/elasticsearch:8.10.2 \
  bin/elasticsearch-certutil cert --ca-cert /certs/ca.crt --ca-key /certs/ca.key \
  --out /certs/elastic-certificates.p12 --pass ""
```

**설명**:
- `elasticsearch-certutil`: Elasticsearch 제공 도구로 PKCS#12 인증서 생성.
- 인증서 파일(`elastic-certificates.p12`)을 `elk-docker/certs/`에 저장.

#### 2.2 `docker-compose.yml` 수정
`es-master`, `es-data1`, `es-data2` 서비스에 다음 추가:
```yaml
volumes:
  - ./certs/elastic-certificates.p12:/usr/share/elasticsearch/config/certs/elastic-certificates.p12
environment:
  - xpack.security.enabled=true
  - xpack.security.transport.ssl.enabled=true
  - xpack.security.transport.ssl.verification_mode=certificate
  - xpack.security.transport.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
  - xpack.security.transport.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
  - xpack.security.http.ssl.enabled=true
  - xpack.security.http.ssl.keystore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
  - xpack.security.http.ssl.truststore.path=/usr/share/elasticsearch/config/certs/elastic-certificates.p12
```

**Kibana 수정**:
```yaml
environment:
  - ELASTICSEARCH_USERNAME=elastic
  - ELASTICSEARCH_PASSWORD=changeme
```

#### 2.3 사용자 인증 설정
```bash
docker exec -it es-master /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
```
- 출력된 `elastic` 사용자 비밀번호를 기록하고 `kibana.yml`에 반영.

#### 2.4 Metricbeat 배포
**`metricbeat.yml`**:
```yaml
metricbeat.modules:
- module: elasticsearch
  metricsets: ["node", "node_stats"]
  hosts: ["https://es-master:9200"]
  username: "elastic"
  password: "changeme"
  ssl.certificate_authorities: ["/usr/share/metricbeat/certs/ca.crt"]

output.elasticsearch:
  hosts: ["https://es-master:9200"]
  username: "elastic"
  password: "changeme"
  ssl.certificate_authorities: ["/usr/share/metricbeat/certs/ca.crt"]
```

**Metricbeat 실행**:
```bash
docker run -d --name metricbeat \
  -v $(pwd)/metricbeat.yml:/usr/share/metricbeat/metricbeat.yml \
  -v $(pwd)/certs/ca.crt:/usr/share/metricbeat/certs/ca.crt \
  docker.elastic.co/beats/metricbeat:8.10.2
```

#### 2.5 검증
- **헬스체크**:
  ```bash
  curl -k -u elastic:changeme https://localhost:9200/_cluster/health?pretty
  ```
- **Kibana**: `https://localhost:5601` 접속, `elastic` 계정으로 로그인.
- **Metricbeat 데이터 확인**: Kibana의 **Stack Monitoring**에서 노드 상태 확인.

**오류 디버깅 팁**:
- **TLS 오류**: 인증서 경로 또는 비밀번호 확인.
- **Metricbeat 연결 실패**: `metricbeat.yml`의 `hosts`와 인증 정보 점검.

**성능 최적화 팁**:
- TLS 오버헤드 최소화: `verification_mode: certificate` 사용.
- Metricbeat 수집 주기: `period: 10s`로 설정하여 부하 감소.

---

## Stage 3. 컨테이너 오케스트레이션 프로덕션 (Kubernetes + ECK)

### 무엇을?
ECK Operator로 Elasticsearch, Kibana, Beats를 선언적으로 배포.

### 왜?
- 자동 스케일링, 롤링 업데이트, 자가 치유.
- 대규모 프로덕션 환경 지원.

### 어떻게?
Kubernetes에서 ECK Operator와 CRD(Custom Resource Definition)를 활용.

#### 3.1 Kubernetes 환경 준비
```bash
# kubectl 설치 확인
kubectl version
# 네임스페이스 생성
kubectl create namespace elk
```

#### 3.2 ECK Operator 설치
```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.9.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.9.0/operator.yaml -n elk
```

#### 3.3 Elasticsearch CR 작성
**`elasticsearch-cluster.yaml`**:
```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: es-cluster
  namespace: elk
spec:
  version: 8.10.2
  auth:
    roles:
    - name: metricbeat
      cluster: ["monitor"]
  nodeSets:
  - name: master
    count: 3
    config:
      node.master: true
      node.data: false
  - name: data
    count: 2
    config:
      node.master: false
      node.data: true
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: standard
        resources:
          requests:
            storage: 50Gi
```

#### 3.4 Kibana CR 작성
**`kibana.yaml`**:
```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: elk
spec:
  version: 8.10.2
  count: 1
  elasticsearchRef:
    name: es-cluster
```

#### 3.5 Metricbeat CR 작성
**`metricbeat.yaml`**:
```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: metricbeat
  namespace: elk
spec:
  type: metricbeat
  version: 8.10.2
  elasticsearchRef:
    name: es-cluster
  config:
    metricbeat.modules:
    - module: elasticsearch
      metricsets: ["node", "node_stats"]
      period: 10s
  daemonSet:
    podTemplate:
      spec:
        containers:
        - name: metricbeat
          volumeMounts:
          - name: ca-crt
            mountPath: /usr/share/metricbeat/certs
        volumes:
        - name: ca-crt
          secret:
            secretName: es-cluster-es-http-certs-public
```

#### 3.6 배포 및 검증
```bash
kubectl apply -f elasticsearch-cluster.yaml -n elk
kubectl apply -f kibana.yaml -n elk
kubectl apply -f metricbeat.yaml -n elk
```

**검증**:
- **클러스터 상태**:
  ```bash
  kubectl get elasticsearch -n elk
  kubectl get pods -n elk
  ```
- **Kibana 접근**:
  ```bash
  kubectl port-forward service/kibana-kb-http -n elk 5601:5601
  ```
  `https://localhost:5601` 접속, `elastic` 계정으로 로그인.

**오류 디버깅 팁**:
- **Pod Pending**: 스토리지 클래스(`standard`) 또는 리소스 요청 확인.
- **ECK Operator 오류**: `kubectl logs -n elastic-system`으로 로그 확인.

**성능 최적화 팁**:
- 샤드/레플리카: 데이터 노드당 샤드 수 제한(예: 20개 이하).
- 리소스 할당: CPU/메모리 요청 및 제한 설정.

---

## Stage 4. 패키지 설치 기반 프로덕션 (Bare-Metal / VM)

### 무엇을?
OS 패키지 매니저로 Elasticsearch를 설치하고 서비스로 관리.

### 왜?
- 컨테이너 사용 제약 환경.
- OS 레벨 세밀한 튜닝 요구.

### 어떻게?
`dpkg`/`rpm` 설치, systemd 관리, 커널 파라미터 조정.

#### 4.1 설치
```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.10.2-amd64.deb
sudo dpkg -i elasticsearch-8.10.2-amd64.deb
```

#### 4.2 커널 파라미터 조정
```bash
sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=65536
sudo sysctl -p
```

#### 4.3 방화벽 및 SELinux 설정
```bash
sudo firewall-cmd --add-port=9200/tcp --permanent
sudo firewall-cmd --reload
sudo setenforce 0  # 또는 SELinux 정책 조정
```

#### 4.4 서비스 관리
```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

#### 4.5 검증
```bash
curl -s http://localhost:9200/_cluster/health?pretty
sudo journalctl -u elasticsearch -f
```

**오류 디버깅 팁**:
- **서비스 시작 실패**: `/var/log/elasticsearch/` 로그 확인.
- **포트 충돌**: `netstat -tuln | grep 9200`으로 점검.

**성능 최적화 팁**:
- JVM 힙: `/etc/elasticsearch/jvm.options`에서 `-Xms4g -Xmx4g`로 설정.
- 로그 로테이션: `/etc/logrotate.d/elasticsearch` 설정.

---

## Stage 5. 매니지드 서비스 (Elastic Cloud / AWS OpenSearch)

### 무엇을?
클라우드 콘솔에서 ELK 클러스터를 프로비저닝.

### 왜?
- 운영 부담 감소, 자동 백업, 버전 관리.
- 고가용성 클러스터 제공.

### 어떻게?
Elastic Cloud 또는 AWS OpenSearch Service 사용.

#### 5.1 Elastic Cloud
1. [Elastic Cloud 콘솔](https://cloud.elastic.co/)에서 클러스터 생성.
2. **Deployment** → **Create deployment** → Elasticsearch 및 Kibana 선택.
3. IP 화이트리스트 설정 및 사용자 인증(RBAC) 구성.

#### 5.2 AWS OpenSearch
```bash
aws opensearch create-domain \
  --domain-name elk-cluster \
  --engine-version OpenSearch_2.7 \
  --cluster-config InstanceType=m5.large.search,InstanceCount=3 \
  --ebs-options EBSEnabled=true,VolumeSize=100
```

**VPC 엔드포인트 설정**:
```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.opensearch
```

#### 5.3 검증
- Elastic Cloud: 콘솔에서 **Kibana** URL 접속.
- AWS OpenSearch: `aws opensearch describe-domain --domain-name elk-cluster`.

**오류 디버깅 팁**:
- **접근 오류**: IP 화이트리스트 또는 IAM 정책 확인.
- **성능 문제**: 인스턴스 크기 업그레이드 또는 샤드 수 조정.

**성능 최적화 팁**:
- 자동 스냅샷: 백업 주기 설정(예: 매일).
- 모니터링: CloudWatch 또는 Elastic Stack Monitoring 활용.

---

## 추가 팁
- **버전 호환성**: Elasticsearch 8.10.2 기준, 최신 버전으로 업그레이드 시 [공식 문서](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html) 확인.
- **시각화 자료**: 아키텍처 다이어그램 생성 요청 시, Mermaid 또는 PlantUML로 제공 가능.
- **Kibana 대시보드**: 샘플 데이터로 대시보드 생성 예시:
  ```bash
  curl -X POST "http://localhost:5601/api/saved_objects/_import" -H "kbn-xsrf: true" --form file=@dashboard.ndjson
  ```